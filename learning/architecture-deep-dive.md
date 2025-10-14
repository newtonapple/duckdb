# DuckDB Architecture Deep Dive

This document provides a comprehensive technical overview of DuckDB's internal architecture for developers who want to understand how the system works at a deep level.

## Table of Contents

1. [System Overview](#system-overview)
2. [Complete Query Execution Flow](#complete-query-execution-flow)
3. [Core Data Structures](#core-data-structures)
4. [Key Components and Their Relationships](#key-components-and-their-relationships)
5. [Memory Management and Buffer Management](#memory-management-and-buffer-management)
6. [Catalog System Design](#catalog-system-design)
7. [Type System Internals](#type-system-internals)
8. [Expression Evaluation](#expression-evaluation)
9. [Pipeline Architecture](#pipeline-architecture)
10. [Design Patterns](#design-patterns)
11. [Component Communication](#component-communication)

---

## System Overview

DuckDB follows a classic layered database architecture with a modern twist: it's designed as an embeddable analytical database with a columnar-vectorized execution engine. The system is composed of several major subsystems that work together to process SQL queries.

### High-Level Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│              Client API (C/C++/Python)              │
├─────────────────────────────────────────────────────┤
│                  ClientContext                      │
│         (Query Coordination & Session State)        │
├─────────────────────────────────────────────────────┤
│    Parser → Binder → Planner → Optimizer           │
│         (Query Processing Pipeline)                  │
├─────────────────────────────────────────────────────┤
│              Physical Plan Generator                 │
├─────────────────────────────────────────────────────┤
│        Executor (Pipeline-based Execution)          │
├─────────────────────────────────────────────────────┤
│     Catalog    │  Transaction  │  Storage           │
│     System     │   Manager     │  Manager           │
├─────────────────────────────────────────────────────┤
│            Buffer Manager & Block Manager           │
└─────────────────────────────────────────────────────┘
```

**Key Source Files:**
- Database Instance: `/src/include/duckdb/main/database.hpp:37-97`
- Client Context: `/src/include/duckdb/main/client_context.hpp:65-318`
- Core entry point: `/src/include/duckdb/main/connection.hpp`

---

## Complete Query Execution Flow

### Phase 1: Query Submission and Parsing

When a query is submitted through `ClientContext::Query()`, the following sequence occurs:

```
User Query (SQL String)
    ↓
ClientContext::Query() [client_context.hpp:109]
    ↓
ParseStatements() [client_context.hpp:181]
    ↓
PostgreSQL Parser (libpg_query)
    ↓
SQLStatement objects [sql_statement.hpp:20-66]
```

**Details:**
1. The query string is sent to `ClientContext::Query()` or `ClientContext::PendingQuery()`
2. The parser uses PostgreSQL's libpg_query to tokenize and parse the SQL
3. Parser produces a tree of `SQLStatement` objects with metadata:
   - `stmt_location`: Position in original query string
   - `stmt_length`: Length of statement
   - `named_param_map`: Named parameter mappings
   - `query`: Original query text

**Source Reference:** `/src/include/duckdb/parser/sql_statement.hpp:20-66`

### Phase 2: Binding (Semantic Analysis)

The Binder resolves symbols, validates types, and produces a Logical Query Plan.

```
SQLStatement
    ↓
Binder::Bind() [binder.hpp:218-219]
    ↓
┌─────────────────────────────────────┐
│    Symbol Resolution via Catalog    │
│  - Tables, Columns, Functions, etc. │
└─────────────────────────────────────┘
    ↓
Bind Context [bind_context.hpp]
    ↓
LogicalOperator tree [logical_operator.hpp:28-111]
```

**Key Responsibilities:**
- **Symbol Resolution**: Converts table/column names to actual catalog entries
- **Type Checking**: Ensures all expressions have valid types
- **Function Resolution**: Binds function calls to actual implementations
- **Subquery Handling**: Resolves correlated and uncorrelated subqueries
- **CTE Binding**: Handles Common Table Expressions

**Binder State:**
```cpp
class Binder {
    ClientContext &context;           // Database context
    BindContext bind_context;         // Current binding scope
    CorrelatedColumns correlated_columns;  // Correlated columns
    optional_ptr<DummyBinding> macro_binding;  // Macro parameters
    shared_ptr<GlobalBinderState> global_binder_state;
    shared_ptr<QueryBinderState> query_binder_state;
    // ... more fields
};
```

**Source Reference:** `/src/include/duckdb/planner/binder.hpp:193-543`

### Phase 3: Logical Planning

The Binder creates a tree of `LogicalOperator` nodes representing the query plan.

```
Common LogicalOperator Types:
┌──────────────────┬────────────────────────────────────┐
│ Operator Type    │ Purpose                            │
├──────────────────┼────────────────────────────────────┤
│ LogicalGet       │ Table scan operation               │
│ LogicalFilter    │ WHERE clause filtering             │
│ LogicalProjection│ SELECT list projection             │
│ LogicalJoin      │ Join operations (INNER, LEFT, etc.)│
│ LogicalAggregate │ GROUP BY aggregation               │
│ LogicalOrder     │ ORDER BY sorting                   │
│ LogicalLimit     │ LIMIT/OFFSET                       │
│ LogicalCTE       │ Common Table Expressions           │
│ LogicalWindow    │ Window functions                   │
└──────────────────┴────────────────────────────────────┘
```

**LogicalOperator Structure:**
```cpp
class LogicalOperator {
    LogicalOperatorType type;                      // Operator type enum
    vector<unique_ptr<LogicalOperator>> children;  // Child operators
    vector<unique_ptr<Expression>> expressions;    // Expressions (filters, projections)
    vector<LogicalType> types;                     // Output column types
    idx_t estimated_cardinality;                   // Estimated row count
    bool has_estimated_cardinality;
};
```

**Source Reference:** `/src/include/duckdb/planner/logical_operator.hpp:28-111`

### Phase 4: Optimization

The Optimizer transforms the logical plan into a more efficient equivalent plan.

```
LogicalOperator (Unoptimized)
    ↓
Optimizer::Optimize() [optimizer.hpp:26]
    ↓
┌─────────────────────────────────────┐
│    Rule-Based Optimizations         │
│  - Filter Pushdown                  │
│  - Projection Pushdown              │
│  - Common Subexpression Elimination │
│  - Expression Simplification        │
│  - Join Reordering (heuristic)      │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│    Cost-Based Optimizations         │
│  - Join Order Selection             │
│  - Index Selection                  │
└─────────────────────────────────────┘
    ↓
LogicalOperator (Optimized)
```

**Optimizer Architecture:**
```cpp
class Optimizer {
    ClientContext &context;
    Binder &binder;
    ExpressionRewriter rewriter;  // Rewrites expressions
    unique_ptr<LogicalOperator> plan;

    void RunBuiltInOptimizers();  // Runs all optimizer passes
};
```

**Key Optimization Passes:**
1. **Filter Pushdown**: Moves filters closer to data sources
2. **Projection Pushdown**: Eliminates unnecessary column reads
3. **Join Order Optimization**: Determines optimal join order using cardinality estimates
4. **Expression Simplification**: Constant folding, algebraic simplification
5. **Subquery Flattening**: Converts correlated subqueries to joins
6. **Common Subexpression Elimination**: Avoids redundant computation

**Source Reference:** `/src/include/duckdb/optimizer/optimizer.hpp:21-54`

### Phase 5: Physical Planning

Converts logical operators to physical operators that can be executed.

```
LogicalOperator
    ↓
PhysicalPlanGenerator
    ↓
PhysicalOperator tree [physical_operator.hpp:38-239]
```

**PhysicalOperator Structure:**
```cpp
class PhysicalOperator {
    PhysicalOperatorType type;                           // Operator type
    ArenaLinkedList<reference<PhysicalOperator>> children;  // Children
    vector<LogicalType> types;                           // Output types
    idx_t estimated_cardinality;

    unique_ptr<GlobalSinkState> sink_state;              // Global sink state
    unique_ptr<GlobalOperatorState> op_state;            // Global operator state

    // Operator interface
    virtual OperatorResultType Execute(...);

    // Source interface (for scan operators)
    virtual SourceResultType GetData(...);

    // Sink interface (for operators that consume data)
    virtual SinkResultType Sink(...);
};
```

**Key Physical Operators:**
- `PhysicalTableScan`: Reads data from storage
- `PhysicalFilter`: Applies filter predicates
- `PhysicalProjection`: Projects columns
- `PhysicalHashJoin`: Hash-based join
- `PhysicalHashAggregate`: Hash-based aggregation
- `PhysicalSort`: Sorting operation
- `PhysicalLimit`: Limit/offset

**Source Reference:** `/src/include/duckdb/execution/physical_operator.hpp:38-239`

### Phase 6: Execution

The Executor runs the physical plan using a pipeline-based execution model.

```
PhysicalOperator tree
    ↓
Executor::Initialize() [executor.hpp:54]
    ↓
Build Pipelines [pipeline.hpp:72-162]
    ↓
┌─────────────────────────────────────┐
│    Pipeline 1: Source → Operators   │
│    Pipeline 2: Operators → Sink     │
│    Pipeline 3: ...                  │
└─────────────────────────────────────┘
    ↓
Parallel Execution (Task Scheduler)
    ↓
PipelineExecutor::Execute()
    ↓
DataChunk-at-a-time processing
    ↓
Query Results
```

**Executor Architecture:**
```cpp
class Executor {
    ClientContext &context;
    optional_ptr<PhysicalOperator> physical_plan;
    vector<shared_ptr<Pipeline>> pipelines;
    vector<shared_ptr<Pipeline>> root_pipelines;
    unique_ptr<PipelineExecutor> root_executor;
    atomic<idx_t> completed_pipelines;
    idx_t total_pipelines;

    void Initialize(PhysicalOperator &physical_plan);
    PendingExecutionResult ExecuteTask(bool dry_run);
};
```

**Source Reference:** `/src/include/duckdb/execution/executor.hpp:37-194`

---

## Core Data Structures

### DataChunk: The Unit of Execution

A `DataChunk` represents a horizontal slice of data, typically 2048 rows (STANDARD_VECTOR_SIZE).

```cpp
class DataChunk {
    vector<Vector> data;      // Column vectors
    idx_t count;              // Number of tuples (rows)
    idx_t capacity;           // Maximum capacity
    idx_t initial_capacity;   // Capacity set at initialization
    vector<VectorCache> vector_caches;  // Memory caches
};
```

**Key Operations:**
- `Initialize()`: Allocates memory for vectors
- `Append()`: Appends another chunk
- `Copy()`: Copies data to another chunk
- `Slice()`: Creates a slice view
- `Flatten()`: Converts all vectors to flat format
- `Hash()`: Computes hash values for grouping/joining

**Memory Model:**
- DataChunk owns the memory for its vectors
- Vectors can be "referencing" (pointing to external data) or "owning"
- Selection vectors allow efficient filtering without data movement

**Source Reference:** `/src/include/duckdb/common/types/data_chunk.hpp:43-176`

### Vector: Columnar Data Container

A `Vector` represents a single column of data with various physical representations.

```cpp
class Vector {
    VectorType vector_type;      // FLAT, CONSTANT, DICTIONARY, SEQUENCE, FSST
    LogicalType type;            // Data type (INTEGER, VARCHAR, etc.)
    data_ptr_t data;             // Pointer to actual data
    ValidityMask validity;       // NULL bitmap
    buffer_ptr<VectorBuffer> buffer;      // Main data buffer
    buffer_ptr<VectorBuffer> auxiliary;   // Auxiliary data (strings, lists)
};
```

**Vector Types:**

1. **FLAT_VECTOR**: Contiguous array of values
   - Most common type during execution
   - Direct memory access: `data[i]`

2. **CONSTANT_VECTOR**: Single repeated value
   - Used for constant expressions
   - Only stores one value

3. **DICTIONARY_VECTOR**: Indirection through selection vector
   - Used for efficient filtering
   - Access: `data[sel_vector[i]]`

4. **SEQUENCE_VECTOR**: Arithmetic sequence
   - For sequences like `ROW_NUMBER()`
   - Stores only start and increment

5. **FSST_VECTOR**: Compressed string vector
   - Uses FSST compression for strings

**Key Operations:**
- `Flatten()`: Converts to FLAT_VECTOR
- `ToUnifiedFormat()`: Creates unified read interface
- `Slice()`: Creates dictionary view with selection vector
- `Reference()`: Makes this vector reference another

**Source Reference:** `/src/include/duckdb/common/types/vector.hpp:124-312`

### UnifiedVectorFormat: Uniform Access Pattern

To efficiently read from any vector type, DuckDB uses `UnifiedVectorFormat`:

```cpp
struct UnifiedVectorFormat {
    const SelectionVector *sel;  // Selection vector (may be identity)
    data_ptr_t data;             // Pointer to actual data
    ValidityMask validity;       // Validity mask
    PhysicalType physical_type;  // Physical storage type
};
```

**Usage Pattern:**
```cpp
// Convert vector to unified format
UnifiedVectorFormat vdata;
vector.ToUnifiedFormat(count, vdata);

// Access elements
auto data = UnifiedVectorFormat::GetData<int32_t>(vdata);
for (idx_t i = 0; i < count; i++) {
    auto idx = vdata.sel->get_index(i);
    if (vdata.validity.RowIsValid(idx)) {
        auto value = data[idx];
        // Process value
    }
}
```

**Source Reference:** `/src/include/duckdb/common/types/vector.hpp:29-74`

---

## Key Components and Their Relationships

### DatabaseInstance: The Root Object

```cpp
class DatabaseInstance {
    DBConfig config;                              // Configuration
    shared_ptr<BufferManager> buffer_manager;     // Memory management
    unique_ptr<DatabaseManager> db_manager;       // Attached databases
    unique_ptr<TaskScheduler> scheduler;          // Task scheduling
    unique_ptr<ObjectCache> object_cache;         // Cached objects
    unique_ptr<ConnectionManager> connection_manager;
    unique_ptr<ExtensionManager> extension_manager;
    shared_ptr<LogManager> log_manager;           // WAL logging
};
```

**Responsibilities:**
- Manages all database-wide resources
- Coordinates attached databases
- Manages buffer pool and memory
- Schedules parallel tasks
- Handles extensions

**Source Reference:** `/src/include/duckdb/main/database.hpp:37-97`

### ClientContext: Per-Connection State

```cpp
class ClientContext {
    shared_ptr<DatabaseInstance> db;          // Database instance
    atomic<bool> interrupted;                 // Interrupt flag
    unique_ptr<ClientData> client_data;       // Client-specific data
    TransactionContext transaction;           // Transaction state
    ClientConfig config;                      // Client configuration

    unique_ptr<ActiveQueryContext> active_query;  // Current query
    QueryProgress query_progress;             // Progress tracking
    mutex context_lock;                       // Synchronization
};
```

**Responsibilities:**
- Manages a single database connection/session
- Tracks active transactions
- Maintains prepared statements
- Handles query execution
- Manages connection-specific settings

**Source Reference:** `/src/include/duckdb/main/client_context.hpp:65-318`

### Relationship Diagram

```
DatabaseInstance (1)
    │
    ├─── BufferManager (1)
    │       └─── BufferPool
    │
    ├─── DatabaseManager (1)
    │       └─── AttachedDatabase (*)
    │               ├─── Catalog
    │               ├─── TransactionManager
    │               └─── StorageManager
    │
    ├─── TaskScheduler (1)
    │
    └─── ConnectionManager (1)
            └─── ClientContext (*)
                    ├─── TransactionContext
                    ├─── QueryProfiler
                    └─── PreparedStatements
```

---

## Memory Management and Buffer Management

### Buffer Manager Architecture

The Buffer Manager is responsible for managing memory for data storage and intermediate results.

```cpp
class BufferManager {
    virtual BufferHandle Allocate(MemoryTag tag, idx_t block_size,
                                  bool can_destroy = true) = 0;
    virtual BufferHandle Pin(shared_ptr<BlockHandle> &handle) = 0;
    virtual void Unpin(shared_ptr<BlockHandle> &handle) = 0;

    virtual idx_t GetUsedMemory() const = 0;
    virtual idx_t GetMaxMemory() const = 0;
    virtual idx_t GetUsedSwap() const = 0;
};
```

**Key Concepts:**

1. **Block Handle**: Reference to a block of memory
   - Can be pinned (loaded in memory) or unpinned (can be evicted)
   - Reference counted for safety

2. **Buffer Handle**: RAII wrapper for pinned memory
   - Automatically unpins on destruction
   - Provides pointer to data

3. **Memory Tags**: Track memory usage by category
   - `IN_MEMORY_TABLE`
   - `HASH_TABLE`
   - `PARQUET_READER`
   - etc.

**Buffer Replacement Strategy:**
- LRU (Least Recently Used) eviction
- Blocks can be written to temporary files if memory is exhausted
- Blocks marked as `can_destroy=false` cannot be evicted

**Source Reference:** `/src/include/duckdb/storage/buffer_manager.hpp:25-137`

### Memory Allocation Hierarchy

```
DatabaseInstance
    │
    └─── BufferPool (shared across all databases)
            │
            ├─── BufferManager (per database)
            │       │
            │       └─── BlockHandle (allocated blocks)
            │               ├─── FileBuffer (in-memory data)
            │               └─── TemporaryFileHandle (spilled to disk)
            │
            └─── Allocator (custom allocator)
```

**Key Operations:**

1. **Allocate**: Get new block of memory
   ```cpp
   auto handle = buffer_manager.Allocate(MemoryTag::HASH_TABLE,
                                         256 * 1024);  // 256KB
   ```

2. **Pin**: Load block into memory
   ```cpp
   auto buffer = buffer_manager.Pin(block_handle);
   // Use buffer.Ptr() to access data
   ```

3. **Unpin**: Allow block to be evicted
   ```cpp
   buffer_manager.Unpin(block_handle);
   ```

### Temporary Memory and Spilling

When memory is exhausted:
1. Unpinned blocks are written to temporary files
2. File paths: `<temp_directory>/<block_id>.tmp`
3. On access, blocks are read back from disk
4. Temporary files deleted on query completion or transaction commit

---

## Catalog System Design

The Catalog manages all database metadata: tables, views, functions, types, schemas, etc.

### Catalog Hierarchy

```
Catalog (per database)
    │
    └─── Schema (e.g., 'main', 'temp', 'information_schema')
            │
            ├─── Tables
            │       └─── TableCatalogEntry
            │               ├─── Column definitions
            │               ├─── Constraints
            │               ├─── Indexes
            │               └─── Statistics
            │
            ├─── Views
            │       └─── ViewCatalogEntry
            │               └─── Parsed query
            │
            ├─── Functions
            │       ├─── ScalarFunction
            │       ├─── AggregateFunction
            │       ├─── TableFunction
            │       └─── MacroFunction
            │
            ├─── Types (user-defined types)
            ├─── Sequences
            └─── Indexes
```

**Catalog Entry Types:**
```cpp
enum class CatalogType : uint8_t {
    TABLE_ENTRY,
    SCHEMA_ENTRY,
    VIEW_ENTRY,
    INDEX_ENTRY,
    PREPARED_STATEMENT,
    SEQUENCE_ENTRY,
    COLLATION_ENTRY,
    TYPE_ENTRY,
    DATABASE_ENTRY,
    // Function types
    SCALAR_FUNCTION_ENTRY,
    AGGREGATE_FUNCTION_ENTRY,
    TABLE_FUNCTION_ENTRY,
    MACRO_ENTRY,
    // ... more
};
```

### Catalog Implementation

```cpp
class Catalog {
    AttachedDatabase &db;     // Owning database
    string default_table;     // Default table for SELECT

    // Schema operations
    virtual optional_ptr<SchemaCatalogEntry> LookupSchema(
        CatalogTransaction transaction,
        const EntryLookupInfo &schema_lookup,
        OnEntryNotFound if_not_found) = 0;

    // Entry operations
    optional_ptr<CatalogEntry> GetEntry(
        ClientContext &context,
        const string &schema,
        const EntryLookupInfo &lookup_info,
        OnEntryNotFound if_not_found);
};
```

**DuckCatalog (default implementation):**
- Stores entries in `CatalogSet` (hash map)
- Thread-safe with MVCC for concurrent access
- Supports transactions
- Entries versioned by transaction

**Source Reference:** `/src/include/duckdb/catalog/catalog.hpp:83-455`

### Catalog Transactions

Catalog operations are transactional:
- CREATE/DROP/ALTER are logged to WAL
- Changes visible only within transaction until commit
- Concurrent transactions see consistent snapshots
- Dependencies tracked (e.g., views depend on tables)

### Dependency Management

The `DependencyManager` tracks dependencies between catalog entries:

```
CREATE TABLE t1 (x INT);
CREATE VIEW v1 AS SELECT * FROM t1;

Dependencies:
    v1 → t1 (view depends on table)

Dropping t1:
    - Check if v1 depends on t1
    - Either CASCADE (drop v1 too) or ERROR
```

---

## Type System Internals

### LogicalType: User-Facing Types

```cpp
class LogicalType {
    LogicalTypeId id;             // Type identifier
    shared_ptr<ExtraTypeInfo> type_info_;  // Extra info for complex types
    PhysicalType physical_type_;  // Storage type
};
```

**Type Categories:**

1. **Numeric Types**:
   - `TINYINT`, `SMALLINT`, `INTEGER`, `BIGINT`
   - `UTINYINT`, `USMALLINT`, `UINTEGER`, `UBIGINT`
   - `FLOAT`, `DOUBLE`
   - `DECIMAL(precision, scale)` - fixed-point decimal

2. **String Types**:
   - `VARCHAR` - variable-length string
   - `CHAR(n)` - fixed-length string
   - `BLOB` - binary data

3. **Temporal Types**:
   - `DATE` - calendar date
   - `TIME` - time of day
   - `TIMESTAMP` - date + time
   - `INTERVAL` - time duration

4. **Nested Types**:
   - `ARRAY[type, size]` - fixed-size array
   - `LIST(type)` - variable-length list
   - `STRUCT(name1 type1, ...)` - composite type
   - `MAP(key_type, value_type)` - key-value pairs
   - `UNION(tag type1, tag type2, ...)` - tagged union

5. **Special Types**:
   - `NULL` - null type
   - `BOOLEAN` - true/false
   - `ENUM` - enumerated values
   - `UUID` - universally unique identifier
   - `JSON` - JSON documents

### PhysicalType: Storage Representation

```cpp
enum class PhysicalType : uint8_t {
    BOOL,           // 1 byte
    INT8,           // 1 byte
    INT16,          // 2 bytes
    INT32,          // 4 bytes
    INT64,          // 8 bytes
    UINT8,
    UINT16,
    UINT32,
    UINT64,
    INT128,         // 16 bytes
    FLOAT,          // 4 bytes
    DOUBLE,         // 8 bytes
    VARCHAR,        // string_t (12 bytes: length + prefix + pointer)
    INTERVAL,       // interval_t
    STRUCT,         // nested structure
    LIST,           // list_entry_t (offset + length)
    // ... more
};
```

### Type Conversion and Casting

DuckDB supports implicit and explicit type conversions:

**Implicit Casts (automatic):**
- `INT8` → `INT16` → `INT32` → `INT64`
- `FLOAT` → `DOUBLE`
- Numeric to `VARCHAR`

**Explicit Casts (via CAST):**
- String parsing: `CAST('123' AS INTEGER)`
- Temporal conversions: `CAST(date AS VARCHAR)`
- Nested type conversions

**Cast Cost Model:**
Each conversion has an associated cost:
- Implicit casts: cost 0-100 (lower is preferred)
- No valid cast: cost -1

---

## Expression Evaluation

### Expression Types

```cpp
class Expression {
    ExpressionType type;           // AND, OR, COMPARE, FUNCTION, etc.
    ExpressionClass expression_class;  // BOUND_REF, BOUND_FUNCTION, etc.
    LogicalType return_type;       // Result type
};
```

**Key Expression Classes:**

1. **BoundColumnRefExpression**: Reference to a column
   ```cpp
   ColumnBinding binding;  // (table_index, column_index)
   LogicalType return_type;
   ```

2. **BoundConstantExpression**: Constant value
   ```cpp
   Value value;  // The constant
   ```

3. **BoundComparisonExpression**: Comparisons (=, <, >, etc.)
   ```cpp
   ExpressionType type;  // EQUAL, LESS_THAN, etc.
   unique_ptr<Expression> left;
   unique_ptr<Expression> right;
   ```

4. **BoundFunctionExpression**: Function calls
   ```cpp
   ScalarFunctionCatalogEntry *function;
   vector<unique_ptr<Expression>> children;  // Arguments
   FunctionData bind_info;  // Function-specific state
   ```

5. **BoundCaseExpression**: CASE WHEN expressions
   ```cpp
   unique_ptr<Expression> check;  // Optional CASE expression
   vector<CaseCheck> case_checks;  // WHEN/THEN pairs
   unique_ptr<Expression> else_expr;  // ELSE clause
   ```

6. **BoundSubqueryExpression**: Subqueries
   ```cpp
   unique_ptr<LogicalOperator> subquery;
   SubqueryType subquery_type;  // SCALAR, EXISTS, ANY, etc.
   ```

**Source Reference:** `/src/include/duckdb/planner/expression.hpp:19-71`

### Expression Execution

Expressions are evaluated in a vectorized manner on DataChunks:

```cpp
// Pseudo-code for expression execution
class ExpressionExecutor {
    void Execute(Expression &expr, DataChunk &input, Vector &result) {
        switch (expr.expression_class) {
        case ExpressionClass::BOUND_COLUMN_REF:
            // Copy column from input
            result.Reference(input.data[expr.binding.column_index]);
            break;

        case ExpressionClass::BOUND_FUNCTION:
            // Execute function on input vectors
            ExecuteFunction(expr, input, result);
            break;

        case ExpressionClass::BOUND_COMPARISON:
            // Execute comparison
            ExecuteComparison(expr, input, result);
            break;
        }
    }
};
```

### Function Execution

Functions in DuckDB are vectorized:

```cpp
typedef void (*scalar_function_t)(DataChunk &args, ExpressionState &state,
                                   Vector &result);

// Example: Addition function
void AddFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &left = args.data[0];
    auto &right = args.data[1];

    BinaryExecutor::Execute<int32_t, int32_t, int32_t>(
        left, right, result, args.size(),
        [&](int32_t a, int32_t b) { return a + b; }
    );
}
```

**Function Types:**
1. **Scalar Functions**: Transform individual values
2. **Aggregate Functions**: Combine multiple values (SUM, AVG, etc.)
3. **Table Functions**: Generate tables (read_csv, range, etc.)
4. **Window Functions**: Operate over window frames

---

## Pipeline Architecture

DuckDB uses a **push-based pipeline execution model** with operator fusion.

### Pipeline Concepts

A **Pipeline** is a sequence of operators that process data without materializing intermediate results:

```
Source → Filter → Projection → Sink
```

**Pipeline Breakers** (operators that materialize data):
- Hash Join (build side)
- Hash Aggregation (grouping)
- Sort
- Window functions (with ORDER BY)

### Pipeline Construction

```
Physical Plan:
    HashAggregate (Pipeline Breaker)
        ↓
    HashJoin (Pipeline Breaker)
        ↓
    Filter
        ↓
    TableScan

Pipelines:
    Pipeline 1: TableScan → Filter → HashJoin (build)
    Pipeline 2: TableScan → HashJoin (probe) → HashAggregate (build)
    Pipeline 3: HashAggregate (scan) → Output
```

**Pipeline Execution Order:**
1. Pipeline 1 builds hash table (child pipelines execute first)
2. Pipeline 2 probes hash table and builds aggregate hash table
3. Pipeline 3 scans aggregate hash table and outputs results

**Source Reference:** `/src/include/duckdb/parallel/pipeline.hpp:72-162`

### Operator Interfaces

Physical operators implement three interfaces:

#### 1. Operator Interface (intermediate operators)
```cpp
virtual OperatorResultType Execute(ExecutionContext &context,
                                   DataChunk &input,
                                   DataChunk &output,
                                   GlobalOperatorState &gstate,
                                   OperatorState &lstate) const;
```

#### 2. Source Interface (scan operators)
```cpp
virtual SourceResultType GetData(ExecutionContext &context,
                                 DataChunk &chunk,
                                 OperatorSourceInput &input) const;
```

#### 3. Sink Interface (materializing operators)
```cpp
virtual SinkResultType Sink(ExecutionContext &context,
                            DataChunk &chunk,
                            OperatorSinkInput &input) const;

virtual SinkFinalizeType Finalize(Pipeline &pipeline,
                                  Event &event,
                                  ClientContext &context,
                                  OperatorSinkFinalizeInput &input) const;
```

### Parallel Execution

Pipelines execute in parallel using multiple threads:

1. **Task Scheduler**: Manages thread pool
2. **PipelineTask**: Represents work for one thread
3. **PipelineExecutor**: Executes pipeline on one thread

```cpp
class PipelineTask : public ExecutorTask {
    Pipeline &pipeline;
    unique_ptr<PipelineExecutor> pipeline_executor;

    TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override {
        // Execute pipeline on current thread
        return pipeline_executor->Execute();
    }
};
```

**Parallelization Strategies:**

1. **Partition-based**: Divide source data into partitions
2. **Pipeline parallelism**: Multiple operators run concurrently on different data
3. **Intra-operator parallelism**: Single operator uses multiple threads

**Synchronization:**
- Sources have `GlobalSourceState` (shared) and `LocalSourceState` (per-thread)
- Sinks have `GlobalSinkState` (shared) and `LocalSinkState` (per-thread)
- Mutexes protect shared state

---

## Design Patterns

### 1. Visitor Pattern

Used extensively for traversing operator trees:

```cpp
class LogicalOperatorVisitor {
    virtual void VisitOperator(LogicalOperator &op);
    virtual void VisitExpression(unique_ptr<Expression> &expr);
};

// Usage:
class MyVisitor : public LogicalOperatorVisitor {
    void VisitOperator(LogicalOperator &op) override {
        // Custom logic for each operator type
        if (op.type == LogicalOperatorType::LOGICAL_FILTER) {
            // Handle filter
        }
        // Recurse to children
        LogicalOperatorVisitor::VisitOperator(op);
    }
};
```

**Source Reference:** `/src/include/duckdb/planner/logical_operator_visitor.hpp`

### 2. RAII (Resource Acquisition Is Initialization)

Used for automatic resource management:

```cpp
// BufferHandle automatically unpins on destruction
{
    BufferHandle handle = buffer_manager.Pin(block);
    // Use handle.Ptr() to access data
    // ...
} // handle automatically unpinned here

// ClientContextLock automatically releases mutex
{
    auto lock = context.LockContext();
    // Critical section
} // mutex released here
```

### 3. Type Erasure

Used for generic operator and expression handling:

```cpp
class LogicalOperator {
    template <class TARGET>
    TARGET &Cast() {
        if (type != TARGET::TYPE) {
            throw InternalException("Type mismatch");
        }
        return reinterpret_cast<TARGET &>(*this);
    }
};

// Usage:
auto &filter = op.Cast<LogicalFilter>();
```

### 4. Factory Pattern

Used for creating operators, expressions, and functions:

```cpp
// Expression factory
unique_ptr<Expression> Expression::Deserialize(Deserializer &source);

// Operator factory
unique_ptr<LogicalOperator> LogicalOperator::Deserialize(Deserializer &source);
```

### 5. State Pattern

Used for operator execution state:

```cpp
// Each operator has local and global state
class OperatorState {
    virtual void Finalize(PhysicalOperator &op, ExecutionContext &context);
};

class GlobalOperatorState {
    // Shared state across all threads
};

class LocalOperatorState : public OperatorState {
    // Per-thread state
};
```

### 6. Strategy Pattern

Used for different algorithm implementations:

```cpp
// Different join algorithms
class PhysicalHashJoin : public PhysicalJoin { ... };
class PhysicalNestedLoopJoin : public PhysicalJoin { ... };
class PhysicalMergeJoin : public PhysicalJoin { ... };

// Planner chooses strategy based on statistics
```

### 7. Template Method Pattern

Used in operator base classes:

```cpp
class PhysicalOperator {
    // Template method
    OperatorResultType ExecuteInternal(...) {
        // Common setup
        auto result = Execute(...);  // Subclass implements
        // Common cleanup
        return result;
    }

    virtual OperatorResultType Execute(...) = 0;  // Pure virtual
};
```

---

## Component Communication

### Data Flow

```
┌──────────────┐
│ ClientContext│
└──────┬───────┘
       │ Query String
       ↓
┌──────────────┐
│    Parser    │
└──────┬───────┘
       │ SQLStatement
       ↓
┌──────────────┐
│    Binder    │←─────────┐
└──────┬───────┘          │
       │ LogicalOperator  │
       │                  │
       ↓                  │
┌──────────────┐          │
│  Optimizer   │          │ Catalog Lookups
└──────┬───────┘          │
       │ LogicalOperator  │
       │                  │
       ↓                  │
┌─────────────────┐       │
│ Physical Planner│       │
└──────┬──────────┘       │
       │ PhysicalOperator │
       │                  │
       ↓                  │
┌──────────────┐          │
│   Executor   │          │
└──────┬───────┘          │
       │ Pipelines        │
       │                  │
       ↓                  │
┌──────────────┐          │
│TaskScheduler │          │
└──────┬───────┘          │
       │ PipelineTasks    │
       │                  │
       ↓                  │
┌──────────────┐   ┌──────┴──────┐
│ DataChunk    │──→│   Catalog   │
│ Processing   │   └─────────────┘
└──────┬───────┘          ↑
       │ Results          │
       │                  │ Metadata
       ↓                  │
┌──────────────┐   ┌──────┴──────┐
│ QueryResult  │   │   Storage   │
└──────────────┘   └─────────────┘
```

### Communication Mechanisms

1. **Function Calls**: Synchronous, direct communication
   - Most common within a single component
   - Example: Binder calling Catalog

2. **Shared State**: Via global and local operator states
   - Used in parallel execution
   - Protected by mutexes

3. **Events**: For asynchronous pipeline coordination
   - Pipelines finish → notify dependent pipelines
   - Build hash table → notify probe pipeline

4. **Callbacks**: For extensibility
   - Table functions provide callbacks for scanning
   - User-defined functions

5. **Message Passing**: Through DataChunks
   - Operators pass DataChunks to each other
   - Source → Operator → Operator → Sink

### Transaction Coordination

```
┌──────────────┐
│ClientContext │
└──────┬───────┘
       │ Begin Transaction
       ↓
┌─────────────────────┐
│ TransactionManager  │
└──────┬──────────────┘
       │ Transaction object
       ↓
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Catalog    │────→│   Storage    │────→│BufferManager │
│  (metadata)  │     │   (data)     │     │  (memory)    │
└──────────────┘     └──────────────┘     └──────────────┘
       ↓                     ↓                     ↓
┌────────────────────────────────────────────────────────┐
│              Write-Ahead Log (WAL)                     │
│  (ensures durability of committed transactions)        │
└────────────────────────────────────────────────────────┘
       ↓
  Commit/Rollback
```

**Transaction Protocol:**
1. **Begin**: Create transaction object, assign transaction ID
2. **Execute**: Perform operations (catalog changes, data modifications)
3. **Commit**: Write to WAL, make changes visible, release locks
4. **Rollback**: Discard changes, release locks

### Query Progress Tracking

```
Executor
    │
    ├─ Pipeline 1 ─→ ProgressData
    ├─ Pipeline 2 ─→ ProgressData
    └─ Pipeline 3 ─→ ProgressData
              ↓
    ClientContext::GetQueryProgress()
              ↓
    Aggregate progress from all pipelines
              ↓
    Return percentage complete
```

---

## Advanced Topics

### Vectorized Execution

DuckDB processes data in batches (vectors) rather than tuple-at-a-time:

**Benefits:**
1. **Cache Efficiency**: Data fits in CPU cache
2. **SIMD**: Enables vectorized CPU instructions
3. **Reduced Overhead**: Fewer function calls per tuple
4. **Better Pipelining**: CPUs can better predict branches

**Example - Vectorized Filter:**
```cpp
// Tuple-at-a-time (slow)
for (int i = 0; i < count; i++) {
    if (predicate(input[i])) {
        output[output_count++] = input[i];
    }
}

// Vectorized (fast)
SelectionVector sel;
idx_t result_count = 0;
for (idx_t i = 0; i < count; i++) {
    sel.set_index(result_count, i);
    result_count += predicate(input[i]);
}
output.Slice(input, sel, result_count);
```

### Morsel-Driven Parallelism

DuckDB divides work into "morsels" (chunks of ~2048 rows):
- Each thread processes one morsel at a time
- Dynamic load balancing
- Minimal synchronization overhead

### Operator Fusion

Adjacent operators are fused into a single pipeline:
- Avoids materializing intermediate results
- Better cache locality
- Reduced memory bandwidth

**Example:**
```sql
SELECT a + 1 FROM t WHERE b > 10
```

Operators fused into one pipeline:
```
TableScan → Filter(b > 10) → Projection(a + 1)
```
All executed together without intermediate materialization.

---

## Summary

DuckDB's architecture is carefully designed for high-performance analytical query processing:

1. **Columnar Vectorized Execution**: Processes data in column-oriented batches
2. **Pipeline Parallelism**: Efficient parallel execution with minimal synchronization
3. **Advanced Query Optimization**: Both rule-based and cost-based optimization
4. **Flexible Type System**: Rich support for complex nested types
5. **Efficient Memory Management**: Smart buffer management with spilling to disk
6. **Extensibility**: Well-defined interfaces for functions, types, and storage

The modular design allows components to be developed, tested, and optimized independently while maintaining clean interfaces between layers.

---

## Key Takeaways for Developers

1. **Follow the Data**: Understanding DataChunk and Vector is essential
2. **Vectorize Everything**: Write vectorized code, not tuple-at-a-time
3. **Understand Pipelines**: Know when operators break pipelines
4. **Memory Awareness**: Be mindful of memory allocations and buffer management
5. **Type Safety**: Leverage the strong type system
6. **Test Thoroughly**: Use both unit tests (C++) and sqllogictest
7. **Read the Source**: This document is a guide, not a replacement for reading code

---

## Further Reading

- **Parser**: `/src/parser/` - SQL parsing using libpg_query
- **Planner**: `/src/planner/` - Binding and logical planning
- **Optimizer**: `/src/optimizer/` - Query optimization passes
- **Execution**: `/src/execution/` - Physical operators and executor
- **Storage**: `/src/storage/` - Data storage and buffer management
- **Common**: `/src/common/` - Shared data structures and utilities
- **Function**: `/src/function/` - Built-in functions implementation

For specific implementation details, always refer to the source code at the file:line references provided throughout this document.
