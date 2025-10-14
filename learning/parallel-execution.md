# Parallel Query Execution in DuckDB

This guide provides an in-depth look at DuckDB's parallel execution system for developers working on the codebase. It covers the pipeline model, task scheduling, thread management, and parallelization patterns used throughout the execution engine.

## Table of Contents

1. [Parallel Execution Architecture](#parallel-execution-architecture)
2. [Pipeline Model](#pipeline-model)
3. [Task Scheduling](#task-scheduling)
4. [Thread Management](#thread-management)
5. [Pipeline Breaking](#pipeline-breaking)
6. [Parallel Operators](#parallel-operators)
7. [Data Partitioning Strategies](#data-partitioning-strategies)
8. [Inter-Operator Parallelism](#inter-operator-parallelism)
9. [Intra-Operator Parallelism](#intra-operator-parallelism)
10. [Performance Considerations](#performance-considerations)
11. [Debugging Parallel Execution](#debugging-parallel-execution)
12. [Common Parallelization Patterns](#common-parallelization-patterns)

---

## Parallel Execution Architecture

DuckDB uses a **push-based, vectorized, pipeline-parallel execution model**. The parallel execution system is primarily implemented in:

- `src/parallel/` - Core parallel execution infrastructure
- `src/execution/` - Physical operator execution

### Key Components

1. **Executor** (`duckdb/execution/executor.hpp`) - Manages the overall query execution
2. **Pipeline** (`duckdb/parallel/pipeline.hpp`) - Represents a chain of operators that can be executed together
3. **MetaPipeline** (`duckdb/parallel/meta_pipeline.hpp`) - Groups pipelines sharing the same sink
4. **PipelineExecutor** (`duckdb/parallel/pipeline_executor.hpp`) - Executes a pipeline on a single thread
5. **TaskScheduler** (`duckdb/parallel/task_scheduler.hpp`) - Manages worker threads and task distribution
6. **Event** (`duckdb/parallel/event.hpp`) - Coordinates pipeline dependencies and execution

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                         Executor                              │
│  - Manages all pipelines                                      │
│  - Schedules events                                           │
│  - Handles errors and cancellation                            │
└──────────────┬───────────────────────────────────────────────┘
               │
               │ creates
               ↓
┌──────────────────────────────────────────────────────────────┐
│                      MetaPipeline(s)                          │
│  - Groups pipelines with same sink                            │
│  - Manages dependencies between pipelines                     │
└──────────────┬───────────────────────────────────────────────┘
               │
               │ contains
               ↓
┌──────────────────────────────────────────────────────────────┐
│                        Pipeline(s)                            │
│  - Source → Operators → Sink                                  │
│  - Can execute in parallel if operators support it            │
└──────────────┬───────────────────────────────────────────────┘
               │
               │ scheduled as
               ↓
┌──────────────────────────────────────────────────────────────┐
│                      Pipeline Event(s)                        │
│  - PipelineInitializeEvent                                    │
│  - PipelineEvent (main execution)                             │
│  - PipelinePrepareFinishEvent                                 │
│  - PipelineFinishEvent                                        │
│  - PipelineCompleteEvent                                      │
└──────────────┬───────────────────────────────────────────────┘
               │
               │ spawns
               ↓
┌──────────────────────────────────────────────────────────────┐
│                      Pipeline Task(s)                         │
│  - Executed by worker threads                                 │
│  - Each task has a PipelineExecutor                           │
└──────────────┬───────────────────────────────────────────────┘
               │
               │ executed by
               ↓
┌──────────────────────────────────────────────────────────────┐
│                      TaskScheduler                            │
│  - Thread pool management                                     │
│  - Task queue (ConcurrentQueue)                               │
│  - Worker threads                                             │
└──────────────────────────────────────────────────────────────┘
```

---

## Pipeline Model

### What is a Pipeline?

A **Pipeline** is a linear chain of operators that can execute data in a push-based manner without materializing intermediate results. It consists of:

- **Source**: Produces data (e.g., `PhysicalTableScan`)
- **Operators**: Transform data (e.g., filters, projections)
- **Sink**: Consumes data (e.g., `PhysicalHashAggregate`, `PhysicalHashJoin`)

**Key source file**: `src/include/duckdb/parallel/pipeline.hpp`

### Pipeline Structure

```cpp
class Pipeline : public enable_shared_from_this<Pipeline> {
public:
    Executor &executor;

    // Pipeline components
    optional_ptr<PhysicalOperator> source;              // Data producer
    vector<reference<PhysicalOperator>> operators;      // Intermediate operators
    optional_ptr<PhysicalOperator> sink;                // Data consumer

    // Global source state (shared across all threads)
    unique_ptr<GlobalSourceState> source_state;

    // Dependencies
    vector<weak_ptr<Pipeline>> parents;                 // Dependent pipelines
    vector<weak_ptr<Pipeline>> dependencies;            // This pipeline depends on these

    // Batch tracking for order-preserving operations
    idx_t base_batch_index = 0;
    multiset<idx_t> batch_indexes;
};
```

### Pipeline Execution Flow

```
Source.GetData()
    ↓
[Operator[0].Execute()]
    ↓
[Operator[1].Execute()]
    ↓
    ...
    ↓
[Operator[N].Execute()]
    ↓
Sink.Sink()
```

Each operator processes data in **chunks** (typically 2048 tuples) using a vectorized execution model.

### MetaPipeline

A **MetaPipeline** groups multiple pipelines that share the same sink operator. This is important for operators like hash joins where:
- The build side creates one pipeline (to build the hash table)
- The probe side creates another pipeline (to probe the hash table)
- Both share the hash join operator as their sink

**Key source file**: `src/include/duckdb/parallel/meta_pipeline.hpp`

```cpp
class MetaPipeline : public enable_shared_from_this<MetaPipeline> {
public:
    Executor &executor;
    PipelineBuildState &state;

    optional_ptr<PhysicalOperator> sink;              // Shared sink
    MetaPipelineType type;                            // REGULAR or JOIN_BUILD

    vector<shared_ptr<Pipeline>> pipelines;           // All pipelines with same sink
    vector<shared_ptr<MetaPipeline>> children;        // Child MetaPipelines

    // Dependencies between pipelines within this MetaPipeline
    reference_map_t<Pipeline, vector<reference<Pipeline>>> pipeline_dependencies;
};
```

**MetaPipeline Types**:
- `REGULAR`: Standard pipeline
- `JOIN_BUILD`: Build side of a hash join (must complete before probe can start)

---

## Task Scheduling

### Task Hierarchy

DuckDB uses a hierarchy of task types:

1. **Task** - Base class for all tasks (`duckdb/parallel/task.hpp`)
2. **ExecutorTask** - Tasks that belong to a query executor (`duckdb/parallel/executor_task.hpp`)
3. **PipelineTask** - Tasks that execute a pipeline (`duckdb/parallel/pipeline.hpp`)

```cpp
class Task : public enable_shared_from_this<Task> {
public:
    virtual TaskExecutionResult Execute(TaskExecutionMode mode) = 0;
    virtual void Deschedule();  // Remove from execution
    virtual void Reschedule();  // Add back to queue

    optional_ptr<ProducerToken> token;  // Queue identifier
};

enum class TaskExecutionResult : uint8_t {
    TASK_FINISHED,      // Task completed
    TASK_NOT_FINISHED,  // More work to do
    TASK_ERROR,         // Error occurred
    TASK_BLOCKED        // Waiting for I/O or another task
};

enum class TaskExecutionMode : uint8_t {
    PROCESS_ALL,     // Execute until complete
    PROCESS_PARTIAL  // Execute limited chunks then return
};
```

### Task Scheduler

The **TaskScheduler** manages a pool of worker threads and a concurrent task queue.

**Key source file**: `src/include/duckdb/parallel/task_scheduler.hpp`

```cpp
class TaskScheduler {
public:
    // Schedule a single task
    void ScheduleTask(ProducerToken &producer, shared_ptr<Task> task);

    // Schedule multiple tasks
    void ScheduleTasks(ProducerToken &producer, vector<shared_ptr<Task>> &tasks);

    // Execute tasks (worker thread loop)
    void ExecuteForever(atomic<bool> *marker);

    // Set thread count
    void SetThreads(idx_t total_threads, idx_t external_threads);

    // Get thread count
    int32_t NumberOfThreads();

private:
    DatabaseInstance &db;
    unique_ptr<ConcurrentQueue> queue;              // Lock-free task queue
    vector<unique_ptr<SchedulerThread>> threads;    // Worker threads
    vector<unique_ptr<atomic<bool>>> markers;       // Thread stop signals
};
```

### Concurrent Queue

DuckDB uses **moodycamel::ConcurrentQueue**, a high-performance lock-free queue that supports:
- Multiple producers (each executor gets a `ProducerToken`)
- Multiple consumers (worker threads)
- Bulk enqueue/dequeue operations

### PipelineTask Execution

```cpp
class PipelineTask : public ExecutorTask {
    static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;

    Pipeline &pipeline;
    unique_ptr<PipelineExecutor> pipeline_executor;

public:
    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override {
        if (!pipeline_executor) {
            pipeline_executor = make_uniq<PipelineExecutor>(
                pipeline.GetClientContext(), pipeline);
        }

        if (mode == TaskExecutionMode::PROCESS_PARTIAL) {
            // Process up to 50 chunks
            auto res = pipeline_executor->Execute(PARTIAL_CHUNK_COUNT);
            // ... handle result
        } else {
            // Process until complete
            auto res = pipeline_executor->Execute();
            // ... handle result
        }
    }
};
```

**PROCESS_PARTIAL mode** allows:
- Better thread utilization (threads don't get stuck on long-running tasks)
- Fairer scheduling (small tasks don't wait behind large ones)
- Earlier error detection

---

## Thread Management

### Thread Count Configuration

Thread count is configurable via the `threads` setting:

```sql
SET threads TO 4;  -- Use 4 threads
SET threads TO 0;  -- Use all available cores (default)
```

The actual thread count is determined by:

```cpp
void TaskScheduler::SetThreads(idx_t total_threads, idx_t external_threads) {
    // total_threads = requested threads (from setting)
    // external_threads = threads that will call ExecuteTasks (e.g., main thread)
    // background_threads = total_threads - external_threads

    idx_t background_threads = total_threads - external_threads;
    // Launch 'background_threads' worker threads
    RelaunchThreadsInternal(background_threads);
}
```

### Worker Thread Lifecycle

Each worker thread runs:

```cpp
void TaskScheduler::ExecuteForever(atomic<bool> *marker) {
    while (*marker) {  // Run until marker is set to false
        shared_ptr<Task> task;
        if (queue->Dequeue(task)) {
            // Execute task
            auto result = task->Execute(TaskExecutionMode::PROCESS_PARTIAL);

            if (result == TaskExecutionResult::TASK_NOT_FINISHED) {
                // Re-enqueue for more work
                ScheduleTask(*task->token, task);
            }
            // else: task finished or blocked
        } else {
            // No tasks available, wait on semaphore
            queue->semaphore.wait(TASK_TIMEOUT_USECS);
        }
    }
}
```

### Thread Context

Each thread executing a pipeline gets a **ThreadContext** that maintains:

```cpp
class ThreadContext {
public:
    // Per-thread allocator
    Allocator &GetAllocator();

    // Per-thread profiler
    optional_ptr<QueryProfiler> profiler;
};
```

The **PipelineExecutor** creates its own `ThreadContext`:

```cpp
class PipelineExecutor {
    Pipeline &pipeline;
    ThreadContext thread;                    // Per-thread context
    ExecutionContext context;                // Execution state

    vector<unique_ptr<DataChunk>> intermediate_chunks;     // One per operator
    vector<unique_ptr<OperatorState>> intermediate_states; // One per operator

    unique_ptr<LocalSourceState> local_source_state;
    unique_ptr<LocalSinkState> local_sink_state;
};
```

---

## Pipeline Breaking

Pipeline breakers are operators that **materialize** data, forcing a pipeline boundary. These operators are both **sinks** (consuming data) and **sources** (producing data).

### Common Pipeline Breakers

1. **Hash Aggregate** (`PhysicalHashAggregate`)
   - Builds hash table (sink)
   - Scans hash table for results (source)

2. **Hash Join** (`PhysicalHashJoin`)
   - Build side: Builds hash table (sink)
   - Probe side: Probes hash table (operator)
   - Can become source for full/right outer joins

3. **Order By** (`PhysicalOrder`)
   - Collects all data and sorts (sink)
   - Outputs sorted data (source)

4. **Window Functions** (`PhysicalWindow`)
   - Buffers data for window computation (sink)
   - Outputs computed results (source)

5. **Materialized CTE** (`PhysicalMaterializedCTE`)
   - Executes CTE once and stores results (sink)
   - Multiple scans read from materialized data (source)

### Pipeline Breaking Example: Hash Join

```
Pipeline 1: Scan(R) → Filter → [HashJoin.BuildSink]
                                       ↓
                              [Hash Table Built]
                                       ↓
Pipeline 2: Scan(S) → Filter → [HashJoin.Probe] → Project → Result
```

- **Pipeline 1**: Builds the hash table (sink)
- **Pipeline 2**: Probes the hash table (depends on Pipeline 1 completing)

The `PhysicalHashJoin::BuildPipelines()` method creates both pipelines and sets up the dependency.

### MetaPipeline Build Rules

From `src/include/duckdb/parallel/meta_pipeline.hpp`:

```cpp
// MetaPipeline build rules:
// 1. For joins, build blocking side before probe side
//    - Current pipeline depends on child pipeline (dependency across MetaPipelines)
//
// 2. Build child pipelines last (e.g., Hash Join becomes source after probe)
//    - Child pipeline depends on:
//      * Current streaming pipeline
//      * All pipelines added to MetaPipeline after current
```

---

## Parallel Operators

For an operator to support parallelism, it must implement:

1. **ParallelSource()** / **ParallelSink()** / **ParallelOperator()** - Return `true`
2. **Global state** - Shared across all threads
3. **Local state** - Per-thread private state
4. **Thread-safe operations** - Proper locking for shared state

### Parallel Source Example: PhysicalTableScan

**Key source file**: `src/include/duckdb/execution/operator/scan/physical_table_scan.hpp`

```cpp
class PhysicalTableScan : public PhysicalOperator {
public:
    bool IsSource() const override { return true; }
    bool ParallelSource() const override;  // Returns true if function supports it

    // Global state: Tracks which parts of table have been scanned
    unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) const override;

    // Local state: Per-thread scan state
    unique_ptr<LocalSourceState> GetLocalSourceState(
        ExecutionContext &context, GlobalSourceState &gstate) const override;

    // Get data: Thread-safe
    SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                            OperatorSourceInput &input) const override;
};
```

### Parallel Sink Example: PhysicalHashAggregate

**Key source file**: `src/include/duckdb/execution/operator/aggregate/physical_hash_aggregate.hpp`

```cpp
class PhysicalHashAggregate : public PhysicalOperator {
public:
    bool IsSink() const override { return true; }
    bool ParallelSink() const override { return true; }

    // Global state: Shared hash table (thread-safe via partitioning)
    unique_ptr<GlobalSinkState> GetGlobalSinkState(ClientContext &context) const override;

    // Local state: Per-thread hash table partition
    unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context) const override;

    // Sink: Thread-safe (writes to local partition)
    SinkResultType Sink(ExecutionContext &context, DataChunk &chunk,
                       OperatorSinkInput &input) const override;

    // Combine: Merges local state into global state (called per thread)
    SinkCombineResultType Combine(ExecutionContext &context,
                                 OperatorSinkCombineInput &input) const override;

    // Finalize: Called once after all threads finish (single-threaded)
    SinkFinalizeType Finalize(Pipeline &pipeline, Event &event,
                             ClientContext &context,
                             OperatorSinkFinalizeInput &input) const override;
};
```

### Parallel Operator Example: PhysicalHashJoin (Probe)

**Key source file**: `src/include/duckdb/execution/operator/join/physical_hash_join.hpp`

```cpp
class PhysicalHashJoin : public PhysicalComparisonJoin {
public:
    // Probe side is a parallel operator
    bool ParallelOperator() const override { return true; }

    // Also a parallel sink (build side) and source (for outer joins)
    bool ParallelSink() const override { return true; }
    bool ParallelSource() const override { return true; }

    // Operator state for probe
    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) const override;

    // Execute probe (thread-safe - reads from shared hash table)
    OperatorResultType ExecuteInternal(ExecutionContext &context,
                                      DataChunk &input, DataChunk &chunk,
                                      GlobalOperatorState &gstate,
                                      OperatorState &state) const override;
};
```

### Checking Parallel Support

```cpp
bool Pipeline::ScheduleParallel(shared_ptr<Event> &event) {
    // Check if sink supports parallelism
    if (!sink->ParallelSink()) {
        return false;
    }

    // Check if source supports parallelism
    if (!source->ParallelSource()) {
        return false;
    }

    // Check if all operators support parallelism
    for (auto &op_ref : operators) {
        auto &op = op_ref.get();
        if (!op.ParallelOperator()) {
            return false;
        }
    }

    // All operators support parallelism - schedule parallel tasks
    return LaunchScanTasks(event, max_threads);
}
```

---

## Data Partitioning Strategies

### Partition Types

**Key source file**: `src/include/duckdb/execution/partition_info.hpp`

```cpp
struct OperatorPartitionInfo {
    bool batch_index = false;              // Requires batch index for ordering
    vector<column_t> partition_columns;    // Columns to partition on

    bool RequiresBatchIndex() const { return batch_index; }
    bool RequiresPartitionColumns() const { return !partition_columns.empty(); }
    bool AnyRequired() const { return RequiresBatchIndex() || RequiresPartitionColumns(); }
};
```

### 1. No Partitioning (Free Parallelism)

Used when order doesn't matter and no coordination needed:

```cpp
class PhysicalFilter : public PhysicalOperator {
    bool ParallelOperator() const override { return true; }

    OperatorPartitionInfo RequiredPartitionInfo() const override {
        return OperatorPartitionInfo::NoPartitionInfo();
    }
};
```

Example: Filters, projections, simple scans

### 2. Batch Index Partitioning (Order Preservation)

Used when insertion order must be preserved:

```cpp
class PhysicalInsert : public PhysicalOperator {
    bool ParallelSink() const override { return true; }

    OperatorPartitionInfo RequiredPartitionInfo() const override {
        // Requires batch_index to preserve order
        return OperatorPartitionInfo::BatchIndex();
    }
};
```

**How it works**:
- Each source chunk gets a globally unique, increasing `batch_index`
- Sink processes chunks in batch index order
- Multiple threads can work in parallel, but output preserves order

**Implementation** (from `src/parallel/pipeline_executor.cpp`):

```cpp
// Initialize batch index for this thread
auto &partition_info = local_sink_state->partition_info;
partition_info.batch_index = pipeline.RegisterNewBatchIndex();

// When batch changes
SinkNextBatchType PipelineExecutor::NextBatch(DataChunk &source_chunk) {
    auto partition_data = pipeline.source->GetPartitionData(
        context, source_chunk, *pipeline.source_state,
        *local_source_state, required_partition_info);

    auto next_batch_index = pipeline.base_batch_index + partition_data.batch_index + 1;
    partition_info.batch_index = next_batch_index;

    // Update minimum batch index across all threads
    partition_info.min_batch_index = pipeline.UpdateBatchIndex(
        current_batch, next_batch_index);

    // Notify sink of new batch
    return pipeline.sink->NextBatch(context, next_batch_input);
}
```

### 3. Hash Partitioning (Radix Partitioning)

Used for hash-based operators (aggregates, joins):

```cpp
// PhysicalHashAggregate uses radix partitioning
class RadixPartitionedHashTable {
    // Each thread gets a partition
    vector<unique_ptr<LocalPartitionState>> local_partitions;

    // Global partitioned hash table
    vector<unique_ptr<GlobalPartition>> global_partitions;

    // Partition count is typically a power of 2 (e.g., 256)
    static constexpr idx_t PARTITION_COUNT = 256;
};
```

**Benefits**:
- Minimizes lock contention (each thread writes to different partition)
- Enables efficient combining (merge partitions independently)
- Cache-friendly (data locality within partition)

### 4. Range Partitioning

Used for sorted data:

```cpp
struct ColumnPartitionData {
    Value min_val;  // Minimum value in partition
    Value max_val;  // Maximum value in partition
};

struct SourcePartitionInfo {
    vector<ColumnPartitionData> partition_data;
};
```

---

## Inter-Operator Parallelism

**Inter-operator parallelism** means multiple pipelines execute simultaneously, working on different parts of the query plan.

### Example: Multi-Way Join

```sql
SELECT *
FROM A
JOIN B ON A.id = B.id
JOIN C ON B.id = C.id;
```

**Pipeline structure**:
```
Pipeline 1: Scan(B) → HashJoin_AB.Build  ┐
                                         ├─ Execute in parallel
Pipeline 2: Scan(C) → HashJoin_BC.Build  ┘

Pipeline 3: Scan(A) → HashJoin_AB.Probe → HashJoin_BC.Probe → Result
            (waits for Pipeline 1 and 2 to complete)
```

### Example: UNION ALL

```sql
SELECT * FROM A
UNION ALL
SELECT * FROM B
UNION ALL
SELECT * FROM C;
```

**Pipeline structure**:
```
Pipeline 1: Scan(A) → Union.Sink  ┐
Pipeline 2: Scan(B) → Union.Sink  ├─ Execute in parallel
Pipeline 3: Scan(C) → Union.Sink  ┘

Pipeline 4: Union.Source → Result
```

All three scan pipelines can execute simultaneously, feeding the shared union sink.

---

## Intra-Operator Parallelism

**Intra-operator parallelism** means a single pipeline executes with multiple threads working on different data chunks.

### Example: Parallel Table Scan

```cpp
// Global state tracks progress
struct TableScanGlobalState : public GlobalSourceState {
    atomic<idx_t> current_row;  // Next row to scan
    idx_t max_row;              // Total rows

    idx_t MaxThreads() override {
        // Allow as many threads as configured
        return max_threads;
    }
};

// Local state is per-thread
struct TableScanLocalState : public LocalSourceState {
    idx_t start_row;  // This thread's current row
    idx_t end_row;    // This thread's end row
};

// GetData is thread-safe
SourceResultType PhysicalTableScan::GetData(
    ExecutionContext &context, DataChunk &chunk,
    OperatorSourceInput &input) const {

    auto &gstate = input.global_state.Cast<TableScanGlobalState>();
    auto &lstate = input.local_state.Cast<TableScanLocalState>();

    // Atomically get next chunk to scan
    lstate.start_row = gstate.current_row.fetch_add(STANDARD_VECTOR_SIZE);
    if (lstate.start_row >= gstate.max_row) {
        return SourceResultType::FINISHED;
    }

    // Scan this chunk (no locks needed - different rows)
    ScanChunk(lstate.start_row, chunk);
    return SourceResultType::HAVE_MORE_OUTPUT;
}
```

### Example: Parallel Hash Aggregate

```cpp
// Global state holds partitioned hash table
struct HashAggregateGlobalState : public GlobalSinkState {
    RadixPartitionedHashTable ht;  // Thread-safe via partitioning
};

// Local state is per-thread
struct HashAggregateLocalState : public LocalSinkState {
    unique_ptr<LocalPartitionState> local_partition;  // Private partition
};

// Sink is thread-safe
SinkResultType PhysicalHashAggregate::Sink(
    ExecutionContext &context, DataChunk &chunk,
    OperatorSinkInput &input) const {

    auto &lstate = input.local_state.Cast<HashAggregateLocalState>();

    // Hash and insert into local partition (no locks)
    lstate.local_partition->Sink(chunk);

    return SinkResultType::NEED_MORE_INPUT;
}

// Combine merges local into global (called per thread)
SinkCombineResultType PhysicalHashAggregate::Combine(
    ExecutionContext &context, OperatorSinkCombineInput &input) const {

    auto &gstate = input.global_state.Cast<HashAggregateGlobalState>();
    auto &lstate = input.local_state.Cast<HashAggregateLocalState>();

    // Merge local partition into global (locks only affected partitions)
    gstate.ht.Combine(*lstate.local_partition);

    return SinkCombineResultType::FINISHED;
}
```

---

## Performance Considerations

### 1. Parallelism Overhead

Parallelism has costs:
- Thread creation/destruction
- Task scheduling overhead
- Lock contention
- Cache coherency traffic
- Memory allocation per thread

**Rule of thumb**: Only parallelize if work per thread > ~10ms

### 2. Thread Saturation

```cpp
bool PhysicalOperator::CanSaturateThreads(ClientContext &context) const {
    auto estimated_threads = EstimatedThreadCount();
    auto &scheduler = TaskScheduler::GetScheduler(context);
    return estimated_threads >= idx_t(scheduler.NumberOfThreads());
}
```

If an operator can't saturate threads, consider:
- Increasing parallelism at higher level (inter-operator)
- Reducing thread count for this pipeline
- Adding more work per task

### 3. Order Preservation

Order-preserving operations (batch index) add overhead:
- Batch index tracking
- Potential blocking if batches processed out of order

Only use when necessary (e.g., INSERT to preserve row order).

### 4. Partition Count

For hash-based operators:
- **Too few partitions**: Lock contention, poor cache locality
- **Too many partitions**: Overhead per partition, poor cache utilization

Typical partition count: 256 (2^8)

### 5. Chunk Size

Default chunk size: 2048 tuples (STANDARD_VECTOR_SIZE)

- **Smaller chunks**: More frequent synchronization, better interleaving
- **Larger chunks**: Less overhead, but potential imbalance

### 6. Task Granularity

```cpp
static constexpr const idx_t PARTIAL_CHUNK_COUNT = 50;  // Process 50 chunks per task
```

- **Fine-grained** (few chunks): Better load balancing, more scheduling overhead
- **Coarse-grained** (many chunks): Less overhead, potential imbalance

---

## Debugging Parallel Execution

### 1. Disable Parallelism

```sql
SET threads TO 1;  -- Single-threaded execution
```

This helps identify if issues are parallelism-related.

### 2. Debug Flags

Several debug flags are available:

```cpp
#ifdef DUCKDB_DEBUG_ASYNC_SINK_SOURCE
// Force blocking behavior to test interrupt handling
int debug_blocked_sink_count = 0;
int debug_blocked_source_count = 0;
int debug_blocked_target_count = 1;  // Number of times to block
#endif
```

Compile with `-DDUCKDB_DEBUG_ASYNC_SINK_SOURCE` to enable.

### 3. Pipeline Visualization

```cpp
// Print pipeline structure
void Pipeline::Print() const {
    // Prints: Source → Operators → Sink
}

// Print all pipelines
void Executor::VerifyPipelines() {
    for (auto &pipeline : pipelines) {
        pipeline->Print();
    }
}
```

### 4. Event Dependencies

```cpp
// Verify event dependencies (detect cycles)
void Executor::VerifyScheduledEvents(const ScheduleEventData &event_data) {
    // Uses DFS to detect dependency cycles
    vector<bool> visited;
    vector<bool> recursion_stack;
    VerifyScheduledEventsInternal(0, vertices, visited, recursion_stack);
}
```

### 5. Progress Tracking

```cpp
bool Pipeline::GetProgress(ProgressData &progress) {
    progress = source->GetProgress(client, *source_state);
    progress = sink->GetSinkProgress(client, *sink->sink_state, progress);
    return progress.IsValid();
}
```

Use to monitor pipeline execution progress.

### 6. Interrupt State

```cpp
class InterruptState {
public:
    InterruptMode mode;  // NO_INTERRUPTS, TASK, BLOCKING
    weak_ptr<Task> current_task;

    void Callback() const;  // Signal task to resume
};

enum class InterruptMode : uint8_t {
    NO_INTERRUPTS,  // Will throw if operator blocks
    TASK,          // Deschedule task, reschedule on callback
    BLOCKING       // Block thread, wake on callback
};
```

Operators can return `BLOCKED` to indicate they're waiting for I/O or another operation.

### 7. Assertions

Enable debug assertions:

```cpp
#ifdef DEBUG
D_ASSERT(pipeline.source);
D_ASSERT(!in_process_operators.empty());
#endif
```

Build with `make debug` to enable assertions.

### 8. Profiling

Use the query profiler:

```sql
PRAGMA enable_profiling;
PRAGMA profiling_mode = 'detailed';

-- Your query here

PRAGMA disable_profiling;
```

Shows per-operator timing, parallelism, and memory usage.

---

## Common Parallelization Patterns

### Pattern 1: Parallel Scan + Parallel Sink

**Example**: Scan table, aggregate in parallel

```cpp
// Scan supports parallel source
bool PhysicalTableScan::ParallelSource() const { return true; }

// Aggregate supports parallel sink
bool PhysicalHashAggregate::ParallelSink() const { return true; }

// Pipeline: Scan → Aggregate
// Multiple threads execute this pipeline simultaneously
// Each thread scans different rows, aggregates into local partition
```

### Pattern 2: Sequential Build, Parallel Probe

**Example**: Hash join

```cpp
// Build pipeline (can be parallel or sequential)
Pipeline 1: Scan(R) → HashJoin.BuildSink

// Probe pipeline (parallel)
Pipeline 2: Scan(S) → HashJoin.Probe → ...
```

The probe side reads from the shared hash table (read-only, no locks needed).

### Pattern 3: Parallel Build, Single Combine

**Example**: Hash aggregate

```cpp
// Parallel sink phase
for each thread:
    local_state = GetLocalSinkState()
    while (more data):
        Sink(chunk, local_state)

// Single-threaded combine phase
for each thread:
    Combine(global_state, local_state)  // Can still be parallel

// Single-threaded finalize phase
Finalize(global_state)
```

### Pattern 4: Parallel Union

**Example**: UNION ALL

```cpp
// Multiple pipelines feed same sink in parallel
Pipeline 1: Scan(A) → Union.Sink  ┐
Pipeline 2: Scan(B) → Union.Sink  ├─ Parallel
Pipeline 3: Scan(C) → Union.Sink  ┘

// Union sink is thread-safe (uses locks or partitioning)
```

### Pattern 5: Pipeline Breaker with Fan-Out

**Example**: CTE with multiple consumers

```cpp
// Build CTE once
Pipeline 1: Scan → Filter → CTE.Sink

// Multiple consumers read in parallel
Pipeline 2: CTE.Scan → ... → Result1  ┐
Pipeline 3: CTE.Scan → ... → Result2  ├─ Parallel
Pipeline 4: CTE.Scan → ... → Result3  ┘
```

### Pattern 6: Batch Index for Order Preservation

**Example**: Parallel INSERT

```cpp
// Each thread gets unique, increasing batch_index
Thread 1: batch_index = 1, 3, 5, 7, ...
Thread 2: batch_index = 2, 4, 6, 8, ...

// Sink processes in batch order
// Can process out-of-order internally, but output preserves order
```

### Pattern 7: Blocking Operator (Interrupt)

**Example**: Async I/O

```cpp
SourceResultType AsyncScan::GetData(...) {
    if (!io_ready) {
        // Block this task until I/O completes
        auto &interrupt = input.interrupt_state;

        // Register callback
        RegisterIOCallback([interrupt]() {
            interrupt.Callback();  // Reschedule task
        });

        return SourceResultType::BLOCKED;
    }

    // I/O ready, return data
    return SourceResultType::HAVE_MORE_OUTPUT;
}
```

The task is descheduled and rescheduled when I/O completes.

---

## Summary

DuckDB's parallel execution system provides:

1. **Pipeline-based execution** - Minimize materialization, maximize throughput
2. **Flexible parallelism** - Inter-operator and intra-operator parallelism
3. **Event-driven scheduling** - Efficient dependency management
4. **Lock-free queues** - High-performance task distribution
5. **Partitioning strategies** - Minimize contention, maximize locality
6. **Interrupt support** - Handle async operations gracefully

### Key Takeaways for Developers

- **Implement parallel operators** by:
  - Returning `true` from `ParallelSource()` / `ParallelSink()` / `ParallelOperator()`
  - Providing thread-safe global and local state
  - Using appropriate synchronization (locks, atomics, partitioning)

- **Pipeline breaking** occurs at:
  - Operators that materialize data (aggregates, sorts, joins)
  - Operators become both sink (consume) and source (produce)

- **Choose partitioning strategy** based on:
  - No partitioning: Order-independent, no coordination
  - Batch index: Order must be preserved
  - Hash partitioning: Hash-based operators
  - Range partitioning: Sorted data

- **Test with**:
  - Single-threaded mode (`SET threads TO 1`)
  - Debug assertions (`make debug`)
  - Query profiler (identify bottlenecks)

- **Profile and optimize**:
  - Minimize synchronization overhead
  - Balance partition counts
  - Tune task granularity
  - Consider cache effects

For more information, see:
- `learning/architecture-deep-dive.md` - Overall architecture
- `learning/vectorized-execution.md` - Vectorized execution model
- `learning/debugging-tips.md` - Debugging techniques
