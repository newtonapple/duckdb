# Performance Optimization Guide for DuckDB Developers

This guide covers performance optimization techniques, profiling tools, and benchmarking methodologies for DuckDB developers. It focuses on practical workflows for writing fast, efficient code and identifying performance bottlenecks.

## Table of Contents

1. [Performance Philosophy](#performance-philosophy)
2. [Profiling Tools and Techniques](#profiling-tools-and-techniques)
3. [Using the Benchmark Runner](#using-the-benchmark-runner)
4. [Reading Query Plans](#reading-query-plans)
5. [Identifying Bottlenecks](#identifying-bottlenecks)
6. [Common Performance Anti-Patterns](#common-performance-anti-patterns)
7. [Optimization Techniques](#optimization-techniques)
8. [Memory Optimization](#memory-optimization)
9. [Parallelization for Performance](#parallelization-for-performance)
10. [Cache-Friendly Code Patterns](#cache-friendly-code-patterns)
11. [Benchmarking Methodology](#benchmarking-methodology)
12. [Performance Testing and Regression Detection](#performance-testing-and-regression-detection)

---

## Performance Philosophy

DuckDB's performance philosophy centers on several key principles:

### Vectorized Execution

DuckDB uses a **vectorized push-based execution model**:
- Operations process data in batches (vectors) of **2048 rows** by default (`STANDARD_VECTOR_SIZE`)
- Vectors enable SIMD optimizations and better CPU cache utilization
- Push-based model allows for pipelining and reduces materialization overhead

### Zero-Copy and Minimal Materialization

- Avoid copying data whenever possible
- Use references and pointers to existing data
- Materialize results only when absolutely necessary
- Leverage column-based storage for projection pushdown

### Optimization-First Mindset

- The optimizer should produce fast plans automatically
- Users shouldn't need to manually tune queries
- Statistics and cardinality estimation drive query planning
- Cost-based optimization for join ordering and operator selection

### Memory Efficiency

- Spill to disk when memory is constrained
- Use compression to reduce memory footprint
- Efficient buffer management with LRU eviction
- Minimize allocations in hot paths

### Performance is a Feature

- Performance regressions are bugs
- Benchmark important workloads regularly
- Profile before optimizing
- Measure, don't guess

---

## Profiling Tools and Techniques

### Built-in Query Profiler

DuckDB includes a comprehensive query profiler that tracks execution metrics per operator.

#### Enable Profiling

```sql
-- Enable profiling with default output (query tree)
PRAGMA enable_profiling;
SELECT * FROM large_table WHERE condition;

-- Enable profiling with query tree output
PRAGMA enable_profiling='query_tree';

-- Enable profiling with optimizer information
PRAGMA enable_profiling='query_tree_optimizer';

-- Enable profiling with JSON output
PRAGMA enable_profiling='json';

-- Disable profiling
PRAGMA disable_profiling;
```

#### Configure Profiling Output Location

```sql
-- Write profiling output to file
PRAGMA profiling_output='profile_output.json';

-- Use default output (stderr)
PRAGMA profiling_output='';
```

#### Profiling Coverage

```sql
-- Profile only SELECT statements (default)
SET profiling_coverage='SELECT';

-- Profile all statements (ATTACH, CREATE, INSERT, etc.)
SET profiling_coverage='ALL';
```

#### Profiling Metrics

DuckDB tracks numerous metrics (see `src/include/duckdb/common/enums/metric_type.hpp`):

**Query-level metrics:**
- `LATENCY`: Total query execution time
- `CPU_TIME`: CPU time (excludes I/O wait)
- `BLOCKED_THREAD_TIME`: Time threads were blocked
- `TOTAL_BYTES_READ`: Bytes read from storage
- `TOTAL_BYTES_WRITTEN`: Bytes written to storage

**Operator-level metrics:**
- `OPERATOR_TIMING`: Time spent in each operator
- `OPERATOR_CARDINALITY`: Rows returned by operator
- `OPERATOR_ROWS_SCANNED`: Rows scanned by operator
- `RESULT_SET_SIZE`: Size of result set
- `SYSTEM_PEAK_BUFFER_MEMORY`: Peak buffer manager memory
- `SYSTEM_PEAK_TEMP_DIR_SIZE`: Peak temporary directory size

**Phase timing metrics:**
- `PLANNER`: Planning phase timing
- `PLANNER_BINDING`: Binding phase timing
- `PHYSICAL_PLANNER`: Physical planning timing
- `OPTIMIZER_*`: Individual optimizer timings (join order, filter pushdown, etc.)

#### Custom Profiling Settings

```sql
-- Enable specific metrics
PRAGMA profiling_settings = 'OPERATOR_TIMING,OPERATOR_CARDINALITY';

-- View current settings
PRAGMA profiling_settings;
```

### EXPLAIN ANALYZE

`EXPLAIN ANALYZE` runs a query and shows the execution plan with timing information:

```sql
-- Basic explain analyze
EXPLAIN ANALYZE SELECT * FROM table WHERE condition;

-- JSON format
EXPLAIN ANALYZE (FORMAT JSON) SELECT * FROM table;

-- HTML format (visual tree)
EXPLAIN ANALYZE (FORMAT HTML) SELECT * FROM table;

-- Graphviz format
EXPLAIN ANALYZE (FORMAT GRAPHVIZ) SELECT * FROM table;
```

Example output:
```
┌─────────────────────────────────┐
│┌───────────────────────────────┐│
││   Query Profiling Information ││
│└───────────────────────────────┘│
└─────────────────────────────────┘
EXPLAIN_ANALYZE
├── Physical Plan
│   ├── PROJECTION (1.2ms, 10000 rows)
│   │   └── SEQ_SCAN[integers] (0.8ms, 10000 rows)
└── Total Time: 2.0ms
```

### Reading Profiling Output

#### Query Tree Format

```
Query Profiling Information
============================
Total Time: 0.123s
├─ HASH_GROUP_BY (0.089s, 1000 rows)
│  ├─ HASH_JOIN (0.025s, 10000 rows)
│  │  ├─ SEQ_SCAN[orders] (0.005s, 5000 rows)
│  │  └─ SEQ_SCAN[lineitem] (0.018s, 15000 rows)
```

Key information:
- **Operator name**: Type of physical operator
- **Time**: Wall-clock time spent in operator (includes children)
- **Rows**: Cardinality (number of rows returned)
- **Extra info**: Additional operator-specific details

#### JSON Format

```json
{
  "query": "SELECT ...",
  "latency": 0.123,
  "operator_timing": 0.089,
  "operators": [
    {
      "operator_name": "HASH_GROUP_BY",
      "operator_timing": 0.089,
      "operator_cardinality": 1000,
      "extra_info": {...}
    }
  ]
}
```

JSON output is ideal for programmatic analysis and automated performance regression testing.

### System Profiling Tools

#### Linux: perf

```bash
# Build release with debug info
make reldebug

# Profile CPU usage
perf record -g build/reldebug/duckdb < query.sql

# View report
perf report

# Generate flamegraph
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

Key `perf` commands:
- `perf stat`: Get high-level performance counters
- `perf record -g`: Record with call graphs
- `perf top`: Real-time profiling
- `perf annotate`: Annotate assembly with performance data

#### macOS: Instruments

```bash
# Build release with debug info
make reldebug

# Profile with Time Profiler
instruments -t "Time Profiler" build/reldebug/duckdb

# Or use Instruments GUI
open -a Instruments
```

Instruments templates:
- **Time Profiler**: CPU profiling
- **Allocations**: Memory allocations
- **Leaks**: Memory leak detection
- **System Trace**: System-wide performance

#### Valgrind Callgrind (Linux)

```bash
# Build without sanitizers (conflict with valgrind)
DISABLE_SANITIZER=1 make reldebug

# Profile with callgrind
valgrind --tool=callgrind build/reldebug/duckdb < query.sql

# Visualize with kcachegrind
kcachegrind callgrind.out.*
```

### Micro-Benchmarking with Profiler Class

For micro-benchmarks in C++ code:

```cpp
#include "duckdb/common/profiler.hpp"

Profiler profiler;
profiler.Start();

// Code to profile
PerformOperation();

profiler.End();
double elapsed_seconds = profiler.Elapsed();
```

---

## Using the Benchmark Runner

DuckDB includes a dedicated benchmark framework for performance testing.

### Building with Benchmarks

```bash
# Build release with benchmarks
BUILD_BENCHMARK=1 make

# Build with specific extensions and benchmarks
BUILD_BENCHMARK=1 BUILD_TPCH=1 make

# Build with Ninja (faster)
BUILD_BENCHMARK=1 GEN=ninja make
```

The benchmark runner binary is at: `build/release/benchmark/benchmark_runner`

### Running Benchmarks

```bash
# List all benchmarks
build/release/benchmark/benchmark_runner --list

# Run all benchmarks
build/release/benchmark/benchmark_runner

# Run specific benchmark by name
build/release/benchmark/benchmark_runner "Append100KIntegersAPPENDER"

# Run benchmarks matching pattern
build/release/benchmark/benchmark_runner ".*TPCH.*"

# Run benchmarks in a group
build/release/benchmark/benchmark_runner "Q01"
```

### Benchmark Runner Options

```bash
# Set thread count
build/release/benchmark/benchmark_runner --threads=4

# Set memory limit
build/release/benchmark/benchmark_runner --memory_limit=8GB

# Write results to file
build/release/benchmark/benchmark_runner --out=results.csv

# Write detailed logs
build/release/benchmark/benchmark_runner --log=benchmark.log

# Enable profiling
build/release/benchmark/benchmark_runner --profile

# Detailed profiling
build/release/benchmark/benchmark_runner --detailed-profile

# Show benchmark info
build/release/benchmark/benchmark_runner --info "BenchmarkName"

# Show query for benchmark
build/release/benchmark/benchmark_runner --query "BenchmarkName"

# Set root directory
build/release/benchmark/benchmark_runner --root-dir /path/to/dir

# Disable timeout (for long benchmarks)
build/release/benchmark/benchmark_runner --disable-timeout
```

### Benchmark Output Format

Standard output format:
```
name                              run    timing
Append100KIntegersAPPENDER        1      0.123
Append100KIntegersAPPENDER        2      0.119
Append100KIntegersAPPENDER        3      0.121
Append100KIntegersAPPENDER        4      0.118
Append100KIntegersAPPENDER        5      0.120
```

- First run (0) is warmup and not reported
- Next 5 runs (1-5) are measured
- Timing is in seconds

### Creating Custom Benchmarks

#### C++ Micro-Benchmark

```cpp
#include "benchmark_runner.hpp"
#include "duckdb_benchmark_macro.hpp"

DUCKDB_BENCHMARK(MyBenchmark, "[mygroup]")

void Load(DuckDBBenchmarkState *state) override {
    // Setup phase: create tables, load data
    state->conn.Query("CREATE TABLE test(i INTEGER)");
}

void RunBenchmark(DuckDBBenchmarkState *state) override {
    // Benchmark phase: execute operation being measured
    state->conn.Query("INSERT INTO test VALUES (42)");
}

void Cleanup(DuckDBBenchmarkState *state) override {
    // Cleanup phase: reset state between runs
    state->conn.Query("DELETE FROM test");
}

string VerifyResult(QueryResult *result) override {
    // Verification: check correctness
    if (result->HasError()) {
        return result->GetError();
    }
    return string();
}

string BenchmarkInfo() override {
    return "Description of what this benchmark tests";
}

FINISH_BENCHMARK(MyBenchmark)
```

#### Interpreted Benchmark (.benchmark files)

Create a `.benchmark` file in `benchmark/` directory:

```
# name: Q01
# group: [tpch]
# description: TPC-H Query 1

name Q01
group tpch

load
CREATE TABLE lineitem AS SELECT * FROM tpch_sf1_lineitem();

run
SELECT
    l_returnflag,
    l_linestatus,
    sum(l_quantity) as sum_qty
FROM lineitem
WHERE l_shipdate <= DATE '1998-12-01'
GROUP BY l_returnflag, l_linestatus
ORDER BY l_returnflag, l_linestatus;
```

### TPC-H Benchmarks

DuckDB includes TPC-H benchmark suite:

```bash
# Build with TPC-H extension
BUILD_BENCHMARK=1 BUILD_TPCH=1 make

# Run TPC-H Q1
build/release/benchmark/benchmark_runner "Q01"

# Run all TPC-H queries
build/release/benchmark/benchmark_runner "Q.*"

# Run SF1 (Scale Factor 1) benchmarks
build/release/benchmark/benchmark_runner --root-dir . ".*sf1.*"
```

---

## Reading Query Plans

Understanding query plans is essential for performance optimization.

### EXPLAIN vs EXPLAIN ANALYZE

```sql
-- EXPLAIN: Shows plan without executing
EXPLAIN SELECT * FROM table WHERE condition;

-- EXPLAIN ANALYZE: Executes and shows actual timings
EXPLAIN ANALYZE SELECT * FROM table WHERE condition;
```

Use `EXPLAIN` for quick plan inspection, `EXPLAIN ANALYZE` for performance analysis.

### Physical Operator Types

Common operators and their performance characteristics:

#### Scan Operators

- **SEQ_SCAN**: Sequential table scan
  - Fast for small tables or when reading most rows
  - Slow for large tables with selective filters
  - Check: Is filter being applied during scan?

- **INDEX_SCAN**: Index-based lookup
  - Fast for selective queries
  - Requires index on filter columns
  - Check: Is index being used correctly?

- **PARQUET_SCAN**: Reading from Parquet files
  - Efficient column-based reading
  - Benefits from predicate pushdown
  - Check: Are filters pushed down?

#### Join Operators

- **HASH_JOIN**: Hash-based join
  - Build hash table on smaller side (build), probe with larger side (probe)
  - Fast for equi-joins
  - Check: Is smaller table on build side?

- **PIECEWISE_MERGE_JOIN**: Sort-merge join
  - Used for sorted inputs
  - Good for large tables with existing sort order
  - Check: Are inputs pre-sorted?

- **NESTED_LOOP_JOIN**: Nested loop join
  - Used when no better alternative exists
  - Slow for large tables (O(n*m) complexity)
  - Check: Why isn't hash join being used?

- **CROSS_PRODUCT**: Cartesian product
  - Extremely expensive (O(n*m))
  - Usually indicates missing join condition
  - Check: Is this intentional?

#### Aggregation Operators

- **HASH_GROUP_BY**: Hash-based aggregation
  - Fast for moderate cardinality
  - May spill to disk if many groups
  - Check: Cardinality of groups

- **PERFECT_HASH_GROUP_BY**: Perfect hash aggregation
  - Optimized for known small cardinality (e.g., enums)
  - Very fast, no collisions
  - Check: Is this being used when possible?

- **SORT_GROUP_BY**: Sort-based aggregation
  - Used when input is sorted
  - Streaming, low memory
  - Check: Is input sorted?

#### Sort Operators

- **ORDER_BY**: External sort
  - May spill to disk for large datasets
  - Check: Is sort necessary? Can it be eliminated?

- **TOP_N**: Optimized for LIMIT + ORDER BY
  - Maintains only top N elements
  - Much faster than full sort
  - Check: Is this being used for LIMIT queries?

#### Other Operators

- **PROJECTION**: Column projection
  - Usually very fast
  - Check: Are unnecessary columns being projected?

- **FILTER**: Row filtering
  - Fast if filter is selective
  - Check: Can filter be pushed down?

- **MATERIALIZED_CTE**: Materialized CTE
  - Caches CTE result for reuse
  - Check: Is materialization beneficial?

### Reading Extra Info

Each operator includes `extra_info` with operator-specific details:

```
SEQ_SCAN [table_name]
    Projections: col1, col2
    Filters: col3 > 100 AND col3 IS NOT NULL

HASH_JOIN [INNER]
    Build: probe_side = build_side
    Build Side: right (10000 rows estimated)
    Probe Side: left (50000 rows estimated)
```

Key things to check:
- **Filters**: Are filters being applied early?
- **Projections**: Are we reading only needed columns?
- **Cardinality estimates**: Are estimates accurate?
- **Join order**: Is smaller table on build side?

### Identifying Plan Issues

**Bad pattern: CROSS_PRODUCT**
```
CROSS_PRODUCT
├─ SEQ_SCAN[table1] (1M rows)
└─ SEQ_SCAN[table2] (1M rows)
```
Problem: Missing join condition, 1M * 1M = 1 trillion rows!

**Good pattern: HASH_JOIN**
```
HASH_JOIN[INNER] (50K rows)
├─ SEQ_SCAN[table1] (1M rows, filtered to 50K)
└─ SEQ_SCAN[table2] (10K rows, build side)
```
Solution: Proper join with filters pushed down.

### Using Optimizer Profiling

```sql
PRAGMA enable_profiling='query_tree_optimizer';
SELECT ...;
```

Shows time spent in each optimization phase:
- Binding
- Filter pushdown
- Join order optimization
- Expression rewriting
- Statistics propagation
- etc.

Useful for identifying expensive optimization phases.

---

## Identifying Bottlenecks

### Step-by-Step Bottleneck Analysis

#### 1. Get Baseline Performance

```bash
# Measure query time
echo "SELECT ... FROM ..." | time build/release/duckdb
```

#### 2. Profile the Query

```sql
PRAGMA enable_profiling='query_tree';
SELECT ... FROM ...;
```

Look for:
- Operators consuming most time
- Operators returning many rows
- Unexpected operators (e.g., CROSS_PRODUCT)

#### 3. Check Cardinality Estimates

Compare estimated vs actual cardinality in `EXPLAIN ANALYZE`:

```sql
EXPLAIN ANALYZE SELECT * FROM table WHERE rare_condition;
```

If estimates are way off:
- Statistics may be stale: `ANALYZE table;`
- Statistics may be missing: Check if table has stats
- Query may be complex: Simplify to isolate issue

#### 4. Isolate the Problem

Break complex query into parts:

```sql
-- Test subquery performance
EXPLAIN ANALYZE SELECT * FROM (
    SELECT ... FROM table1 WHERE condition
) subquery;

-- Test join performance separately
EXPLAIN ANALYZE SELECT * FROM table1 JOIN table2 USING (key);
```

#### 5. Check System Resources

```sql
-- Check memory usage
SELECT * FROM duckdb_memory();

-- Check temporary directory usage
SELECT * FROM duckdb_temporary_files();
```

If spilling to disk:
- Increase memory limit: `SET memory_limit='16GB';`
- Optimize query to use less memory
- Check for memory leaks

### Common Bottleneck Patterns

#### Bottleneck: Full Table Scan

**Symptom**: SEQ_SCAN taking most time, reading many rows

**Diagnosis**:
```sql
EXPLAIN ANALYZE SELECT * FROM large_table WHERE rare_condition;
-- Shows: SEQ_SCAN reading 1B rows, returning 100 rows
```

**Solutions**:
- Add index: `CREATE INDEX idx ON large_table(column);`
- Ensure filter is selective enough
- Check if statistics are up to date
- Consider partitioning large tables

#### Bottleneck: Expensive Join

**Symptom**: HASH_JOIN taking most time

**Diagnosis**:
```sql
EXPLAIN ANALYZE SELECT * FROM t1 JOIN t2 ON t1.key = t2.key;
-- Shows: HASH_JOIN building large hash table
```

**Solutions**:
- Check join order (smaller table should be build side)
- Add filters before join: `WHERE t1.filter_col = value`
- Check cardinality estimates
- Consider adding indexes
- Try disabling join order optimizer to test: `PRAGMA disable_optimizer='join_order';`

#### Bottleneck: Large Group By

**Symptom**: HASH_GROUP_BY consuming memory, possibly spilling

**Diagnosis**:
```sql
EXPLAIN ANALYZE SELECT key, SUM(value) FROM table GROUP BY key;
-- Shows: HASH_GROUP_BY with high cardinality
```

**Solutions**:
- Check if grouping columns have high cardinality
- Consider pre-aggregating data
- Increase memory limit if spilling
- Check if indexes can help

#### Bottleneck: Sort Operation

**Symptom**: ORDER_BY taking significant time

**Diagnosis**:
```sql
EXPLAIN ANALYZE SELECT * FROM table ORDER BY column;
-- Shows: ORDER_BY sorting large dataset
```

**Solutions**:
- Add LIMIT if you don't need all rows (enables TOP_N optimization)
- Check if sort can be eliminated
- Consider keeping data pre-sorted
- Add index on sort column

#### Bottleneck: Repeated Computation

**Symptom**: Same subquery or CTE executed multiple times

**Diagnosis**:
```sql
EXPLAIN WITH cte AS (...) SELECT * FROM cte JOIN cte ON ...;
-- Shows: CTE inlined and computed twice
```

**Solutions**:
- Materialize CTE: `PRAGMA enable_optimizer='materialized_cte';`
- Or: Create temporary table
- Check if multiple references to same subquery exist

### System-Level Bottlenecks

#### CPU-Bound

**Indicators**:
- CPU usage near 100% on all cores
- Low I/O wait time
- Query scales with thread count

**Investigation**:
```bash
# Profile with perf
perf record -g build/release/duckdb < query.sql
perf report
```

**Solutions**:
- Look for hot functions in profile
- Optimize inner loops
- Consider SIMD optimizations
- Check for expensive expressions

#### I/O-Bound

**Indicators**:
- High I/O wait time
- CPU usage low
- Slow storage access times

**Investigation**:
```sql
-- Check bytes read
PRAGMA enable_profiling='json';
SELECT ...;
-- Look at "total_bytes_read" metric
```

**Solutions**:
- Use faster storage (SSD vs HDD)
- Reduce data read (better filters, projections)
- Use compression (Parquet, etc.)
- Increase buffer pool size

#### Memory-Bound

**Indicators**:
- High memory usage
- Spilling to disk
- Slow when memory limit reached

**Investigation**:
```sql
-- Check if spilling
SELECT * FROM duckdb_temporary_files();
```

**Solutions**:
- Increase memory limit: `SET memory_limit='32GB';`
- Reduce memory usage (better filters, avoid large intermediate results)
- Process data in chunks
- Use streaming operators where possible

---

## Common Performance Anti-Patterns

### Anti-Pattern 1: SELECT * When You Don't Need All Columns

**Bad**:
```sql
SELECT * FROM large_table WHERE condition;
```

**Good**:
```sql
SELECT col1, col2, col3 FROM large_table WHERE condition;
```

**Why**: Reading only needed columns reduces I/O, memory usage, and network transfer.

### Anti-Pattern 2: Missing WHERE Clauses

**Bad**:
```sql
SELECT SUM(amount) FROM transactions;  -- Scanning billions of rows
```

**Good**:
```sql
SELECT SUM(amount) FROM transactions WHERE date >= '2024-01-01';
```

**Why**: Filters reduce data scanned. Use time-based filters, partition filters, etc.

### Anti-Pattern 3: Non-Selective Filters After Joins

**Bad**:
```sql
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.date >= '2024-01-01';  -- Filter after join
```

**Good**:
```sql
SELECT * FROM (
    SELECT * FROM orders WHERE date >= '2024-01-01'
) o
JOIN customers c ON o.customer_id = c.id;
```

**Why**: Filtering before join reduces join input size. (Note: optimizer should do this automatically via filter pushdown, but explicit can help.)

### Anti-Pattern 4: Implicit Cross Products

**Bad**:
```sql
SELECT * FROM table1, table2, table3;  -- Cartesian product
```

**Good**:
```sql
SELECT * FROM table1
JOIN table2 ON table1.id = table2.table1_id
JOIN table3 ON table2.id = table3.table2_id;
```

**Why**: Cross products are extremely expensive (O(n*m*k)).

### Anti-Pattern 5: Correlated Subqueries

**Bad**:
```sql
SELECT *,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) as order_count
FROM customers c;  -- Subquery executes for each row
```

**Good**:
```sql
SELECT c.*, COALESCE(o.order_count, 0) as order_count
FROM customers c
LEFT JOIN (
    SELECT customer_id, COUNT(*) as order_count
    FROM orders GROUP BY customer_id
) o ON c.id = o.customer_id;
```

**Why**: Correlated subqueries can execute N times. Joins are optimized.

### Anti-Pattern 6: Unnecessary DISTINCT

**Bad**:
```sql
SELECT DISTINCT * FROM table WHERE unique_key = value;
```

**Good**:
```sql
SELECT * FROM table WHERE unique_key = value;
```

**Why**: DISTINCT adds overhead. Use only when necessary.

### Anti-Pattern 7: Functions on Indexed Columns

**Bad**:
```sql
SELECT * FROM table WHERE YEAR(date_column) = 2024;  -- Can't use index
```

**Good**:
```sql
SELECT * FROM table WHERE date_column >= '2024-01-01' AND date_column < '2025-01-01';
```

**Why**: Functions prevent index usage. Use sargable predicates.

### Anti-Pattern 8: Large IN Clauses

**Bad**:
```sql
SELECT * FROM table WHERE id IN (1, 2, 3, ..., 10000);  -- 10K values
```

**Good**:
```sql
CREATE TEMP TABLE filter_ids(id INTEGER);
INSERT INTO filter_ids VALUES (1), (2), ...;
SELECT * FROM table WHERE id IN (SELECT id FROM filter_ids);
```

**Why**: Large IN clauses are slow to parse and execute. Use temp table + join.

### Anti-Pattern 9: Reading Unnecessary Data Formats

**Bad**:
```sql
SELECT COUNT(*) FROM read_json_auto('large_file.json');  -- JSON is slow
```

**Good**:
```sql
-- Convert to Parquet once
COPY (SELECT * FROM read_json_auto('file.json')) TO 'file.parquet';
-- Then use Parquet
SELECT COUNT(*) FROM 'file.parquet';
```

**Why**: Parquet is much faster than JSON/CSV for analytical queries.

### Anti-Pattern 10: Not Using Prepared Statements

**Bad**:
```python
for i in range(1000000):
    conn.execute(f"INSERT INTO table VALUES ({i})")
```

**Good**:
```python
stmt = conn.prepare("INSERT INTO table VALUES (?)")
for i in range(1000000):
    stmt.execute([i])
```

**Better**:
```python
# Use Appender for bulk inserts
appender = conn.appender("table")
for i in range(1000000):
    appender.append([i])
appender.close()
```

**Why**: Parsing overhead adds up. Prepared statements and Appender are much faster.

---

## Optimization Techniques

### Query-Level Optimizations

#### 1. Filter Pushdown

Manually push filters as close to source as possible:

```sql
-- Instead of:
SELECT * FROM (SELECT * FROM large_table) WHERE condition;

-- Write:
SELECT * FROM (SELECT * FROM large_table WHERE condition);
```

Check that optimizer is doing this automatically:
```sql
EXPLAIN SELECT * FROM (SELECT * FROM large_table) WHERE condition;
-- Should show filter in SEQ_SCAN, not separate FILTER operator
```

#### 2. Projection Pushdown

Only select needed columns at source:

```sql
-- Instead of:
WITH data AS (SELECT * FROM table)
SELECT col1, col2 FROM data;

-- Write:
WITH data AS (SELECT col1, col2 FROM table)
SELECT col1, col2 FROM data;
```

#### 3. Join Order Optimization

While optimizer handles this, you can help:

```sql
-- Ensure smaller table is on right (build side) in hash join
SELECT * FROM large_table l
JOIN small_table s ON l.key = s.key;  -- Good: small on right
```

Disable optimizer to test manually:
```sql
PRAGMA disable_optimizer='join_order';
```

#### 4. Limit Pushdown

Use LIMIT early to reduce data processed:

```sql
-- Instead of:
SELECT * FROM (
    SELECT * FROM table ORDER BY date DESC
) LIMIT 100;

-- Optimizer does this, but verify:
EXPLAIN SELECT * FROM table ORDER BY date DESC LIMIT 100;
-- Should show TOP_N operator, not ORDER_BY + LIMIT
```

#### 5. Index Usage

Create indexes on frequently filtered/joined columns:

```sql
CREATE INDEX idx_customer_id ON orders(customer_id);
CREATE INDEX idx_date ON orders(date);

-- Verify index is used
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;
-- Should show INDEX_SCAN, not SEQ_SCAN
```

#### 6. Statistics Management

Keep statistics up to date:

```sql
-- Analyze table
ANALYZE table_name;

-- Check statistics
SELECT * FROM duckdb_tables() WHERE table_name = 'your_table';
```

Statistics improve cardinality estimates and query plans.

### Code-Level Optimizations

#### 1. Minimize Virtual Function Calls

Virtual functions have overhead in hot paths. Use templates or inline where possible:

```cpp
// Instead of virtual function in hot loop:
for (idx_t i = 0; i < count; i++) {
    obj->VirtualFunction(i);  // Indirect call
}

// Use template:
template<class T>
void ProcessData(T& obj, idx_t count) {
    for (idx_t i = 0; i < count; i++) {
        obj.Function(i);  // Direct call, can inline
    }
}
```

#### 2. Branch Prediction

Help the CPU predict branches correctly:

```cpp
// Mark likely/unlikely branches
if (DUCKDB_LIKELY(common_case)) {
    // Fast path
} else {
    // Rare path
}

// Or use early returns for error cases
if (DUCKDB_UNLIKELY(error_condition)) {
    return ErrorResult();
}
// Normal path continues
```

#### 3. Loop Optimizations

Write loops that compiler can vectorize:

```cpp
// Good: Compiler can vectorize
for (idx_t i = 0; i < count; i++) {
    output[i] = input[i] * 2;
}

// Bad: Hard to vectorize
for (idx_t i = 0; i < count; i++) {
    if (input[i] > 0) {
        output[i] = input[i] * 2;
    } else {
        output[i] = 0;
    }
}

// Better: Remove branch
for (idx_t i = 0; i < count; i++) {
    output[i] = input[i] > 0 ? input[i] * 2 : 0;
}
```

#### 4. Use SIMD Intrinsics

For performance-critical operations, use SIMD:

```cpp
#include "duckdb/common/types/simd.hpp"

// Process 8 values at once
for (idx_t i = 0; i < count; i += 8) {
    __m256d vec = _mm256_load_pd(&input[i]);
    vec = _mm256_mul_pd(vec, factor);
    _mm256_store_pd(&output[i], vec);
}
```

DuckDB has SIMD utilities in `src/include/duckdb/common/types/simd.hpp`.

#### 5. Reduce Memory Allocations

Avoid allocations in hot paths:

```cpp
// Bad: Allocates every iteration
for (idx_t i = 0; i < count; i++) {
    vector<int> temp;  // Allocation!
    ProcessData(temp);
}

// Good: Reuse allocation
vector<int> temp;
for (idx_t i = 0; i < count; i++) {
    temp.clear();
    ProcessData(temp);
}
```

#### 6. Use Reference Types

Pass by const reference for non-trivial types:

```cpp
// Bad: Copies string
void ProcessString(string str);

// Good: No copy
void ProcessString(const string &str);

// For return values, use RVO
string CreateString() {
    string result = ...;
    return result;  // RVO: no copy
}
```

#### 7. Data-Oriented Design

Organize data for cache efficiency (see Cache-Friendly Code Patterns).

---

## Memory Optimization

### Understanding Memory Usage

#### Check Memory Statistics

```sql
-- Current memory usage
SELECT * FROM duckdb_memory();

-- Memory limit
PRAGMA memory_limit;

-- Set memory limit
SET memory_limit='16GB';
```

#### Profile Memory Usage

```sql
PRAGMA profiling_settings='SYSTEM_PEAK_BUFFER_MEMORY';
PRAGMA enable_profiling='json';
SELECT ...;
```

Metrics:
- `SYSTEM_PEAK_BUFFER_MEMORY`: Peak buffer manager memory
- `SYSTEM_PEAK_TEMP_DIR_SIZE`: Peak temporary directory size

### Memory Reduction Techniques

#### 1. Limit Intermediate Results

```sql
-- Bad: Large intermediate result
SELECT * FROM (
    SELECT * FROM orders  -- 1B rows
) JOIN customers USING (customer_id);

-- Good: Filter first
SELECT * FROM (
    SELECT * FROM orders WHERE date >= '2024-01-01'  -- 10M rows
) JOIN customers USING (customer_id);
```

#### 2. Use Streaming Operations

DuckDB automatically streams when possible, but you can help:

```sql
-- Streaming aggregation (sorted input)
SELECT date, SUM(amount) FROM orders
WHERE date >= '2024-01-01'
GROUP BY date
ORDER BY date;  -- Already sorted by GROUP BY
```

#### 3. Spill to Disk

Allow DuckDB to spill to disk when memory is full:

```sql
-- Set temp directory
SET temp_directory='/path/to/fast/storage';

-- Check if spilling
SELECT * FROM duckdb_temporary_files();
```

DuckDB automatically spills hash tables, sorts, etc. to disk when memory is constrained.

#### 4. Process in Batches

For ETL workloads, process data in chunks:

```python
# Instead of loading all at once:
df = duckdb.sql("SELECT * FROM huge_table").df()  # OOM!

# Process in batches:
offset = 0
batch_size = 1000000
while True:
    batch = duckdb.sql(f"""
        SELECT * FROM huge_table
        LIMIT {batch_size} OFFSET {offset}
    """).df()
    if len(batch) == 0:
        break
    process_batch(batch)
    offset += batch_size
```

#### 5. Use External Tables

Keep large tables in external formats:

```sql
-- Instead of loading into DuckDB:
CREATE TABLE data AS SELECT * FROM 'huge_file.parquet';  -- Loads into DuckDB

-- Query directly:
SELECT * FROM 'huge_file.parquet' WHERE condition;  -- Stays external
```

### Memory Debugging

#### DEBUG_ALLOCATION Flag

Track memory allocations:

```bash
DEBUG_ALLOCATION=1 make debug
build/debug/test/unittest

# Will report:
# - Outstanding allocations on exit
# - Stack traces for leaks
# - Total allocation counts
```

#### Memory Leak Detection

```bash
# Use ASan leak detection
make debug
ASAN_OPTIONS=detect_leaks=1 build/debug/duckdb
```

---

## Parallelization for Performance

DuckDB automatically parallelizes query execution using a pipeline-based model.

### Understanding Parallelization

#### Pipelines

DuckDB breaks queries into **pipelines**:
- Each pipeline is a sequence of operators
- Pipelines are executed in parallel across threads
- Data flows through pipeline in vectorized chunks (2048 rows)

#### Thread Model

```sql
-- Set thread count
SET threads=8;

-- Check current threads
PRAGMA threads;
```

Default: Uses hardware concurrency (number of logical cores).

### Morsel-Driven Parallelism

DuckDB uses **morsel-driven parallelism**:
- Data is split into "morsels" (chunks)
- Each thread processes morsels independently
- Dynamic load balancing

### Parallelization Patterns

#### 1. Parallel Scan

Table scans are automatically parallelized:

```sql
-- Automatically uses all threads
SELECT * FROM large_table WHERE condition;
```

Each thread scans a portion of the table.

#### 2. Parallel Aggregation

Aggregations are parallelized with partitioning:

```sql
-- Parallel hash aggregation
SELECT key, SUM(value) FROM large_table GROUP BY key;
```

Each thread builds a local hash table, then results are merged.

#### 3. Parallel Join

Hash joins are parallelized:

```sql
-- Parallel hash join
SELECT * FROM table1 JOIN table2 ON table1.key = table2.key;
```

Build side is partitioned across threads, then probed in parallel.

#### 4. Parallel Sort

External sort uses parallel merge sort:

```sql
-- Parallel sort
SELECT * FROM large_table ORDER BY column;
```

Data is partitioned, sorted locally, then merged.

### Optimizing for Parallelism

#### 1. Ensure Sufficient Data

Parallelism has overhead. Only beneficial for sufficient data:

- Small queries (< 1000 rows): May be faster single-threaded
- Medium queries (1K-1M rows): Benefit from 2-4 threads
- Large queries (> 1M rows): Benefit from many threads

#### 2. Avoid Serial Bottlenecks

Some operations don't parallelize well:
- Sequential I/O from single file
- Operations requiring global state
- Small hash tables

Split work to enable parallelism:

```sql
-- Instead of single large file:
SELECT * FROM 'huge_file.csv';

-- Use multiple files:
SELECT * FROM 'data/*.csv';  -- Reads files in parallel
```

#### 3. Balance Work

Ensure even work distribution:

```sql
-- Bad: Skewed distribution
SELECT key, COUNT(*) FROM table GROUP BY key;
-- If one key has 99% of rows, most threads are idle

-- Solution: Pre-partition or handle skew
```

#### 4. Pipeline Breakers

Some operators break pipelines (require materialization):
- Sorting
- Hash aggregation with many groups
- Hash join build side

Minimize pipeline breakers when possible.

### Thread Synchronization

In C++ code, minimize synchronization:

```cpp
// Bad: Global mutex in hot path
std::mutex global_lock;
for (idx_t i = 0; i < count; i++) {
    std::lock_guard<std::mutex> lock(global_lock);
    UpdateGlobalState(i);
}

// Good: Thread-local accumulation
thread_local State local_state;
for (idx_t i = 0; i < count; i++) {
    UpdateLocalState(local_state, i);
}
// Merge at end
MergeStates(local_state);
```

Use atomic operations for counters:

```cpp
#include "duckdb/common/atomic.hpp"

atomic<idx_t> counter(0);
counter.fetch_add(1, std::memory_order_relaxed);
```

---

## Cache-Friendly Code Patterns

CPU cache performance is critical for database systems.

### Cache Hierarchy

Modern CPUs have multiple cache levels:
- **L1**: 32-64 KB per core, ~4 cycles
- **L2**: 256-512 KB per core, ~10 cycles
- **L3**: 8-32 MB shared, ~40 cycles
- **RAM**: GBs, ~200 cycles

Accessing RAM is **50x slower** than L1 cache!

### Cache Line Basics

- Cache line size: **64 bytes** on most architectures
- Data is loaded in cache lines
- Accessing one byte loads entire 64-byte line

### Pattern 1: Structure of Arrays (SoA) vs Array of Structures (AoS)

```cpp
// Bad: Array of Structures (AoS)
struct Point {
    double x;
    double y;
    double z;
};
vector<Point> points;

for (auto &p : points) {
    p.x *= 2;  // Loads all of x, y, z even though we only use x
}

// Good: Structure of Arrays (SoA)
struct Points {
    vector<double> x;
    vector<double> y;
    vector<double> z;
};
Points points;

for (auto &x_val : points.x) {
    x_val *= 2;  // Only loads x values, better cache usage
}
```

**DuckDB uses SoA extensively (columnar storage)**

### Pattern 2: Sequential Access

```cpp
// Good: Sequential access (cache-friendly)
for (idx_t i = 0; i < count; i++) {
    Process(data[i]);
}

// Bad: Random access (cache-unfriendly)
for (idx_t i = 0; i < count; i++) {
    idx_t idx = random_indices[i];
    Process(data[idx]);  // Random jumps, poor cache locality
}
```

### Pattern 3: Data Alignment

Align data to cache line boundaries:

```cpp
// Align to 64 bytes (cache line size)
struct alignas(64) CacheAlignedData {
    int64_t values[8];  // 8 * 8 = 64 bytes
};

// Prevent false sharing in multithreaded code
struct ThreadState {
    alignas(64) atomic<idx_t> counter;  // Own cache line
    alignas(64) atomic<idx_t> other;    // Own cache line
};
```

### Pattern 4: Prefetching

Hint CPU to prefetch data:

```cpp
#include <xmmintrin.h>  // SSE intrinsics

for (idx_t i = 0; i < count; i++) {
    // Prefetch next iteration's data
    if (i + 8 < count) {
        _mm_prefetch(&data[i + 8], _MM_HINT_T0);
    }
    Process(data[i]);
}
```

Use sparingly; CPU prefetcher is usually good enough.

### Pattern 5: Minimize Indirection

```cpp
// Bad: Pointer indirection
vector<Data*> data_ptrs;
for (auto *ptr : data_ptrs) {
    Process(*ptr);  // Pointer dereference, poor locality
}

// Good: Direct data
vector<Data> data;
for (auto &item : data) {
    Process(item);  // Sequential access, good locality
}
```

### Pattern 6: Hot/Cold Data Separation

```cpp
// Bad: Mix hot and cold data
struct DataWithMetadata {
    int64_t hot_value;        // Used in hot path
    string cold_metadata;     // Rarely used
    double another_hot;       // Used in hot path
};

// Good: Separate hot and cold
struct HotData {
    int64_t hot_value;
    double another_hot;
};

struct ColdData {
    string cold_metadata;
};
```

Keep hot data together in cache lines.

### Pattern 7: Batch Processing

```cpp
// Bad: Process one at a time
for (idx_t i = 0; i < count; i++) {
    ProcessSingle(data[i]);
}

// Good: Process in batches (vectorize)
for (idx_t i = 0; i < count; i += BATCH_SIZE) {
    idx_t batch_end = std::min(i + BATCH_SIZE, count);
    ProcessBatch(&data[i], batch_end - i);
}
```

Batching enables better cache usage and SIMD.

### Pattern 8: Branch Prediction

```cpp
// Bad: Unpredictable branches in loop
for (idx_t i = 0; i < count; i++) {
    if (random_condition(i)) {  // Unpredictable!
        DoA(i);
    } else {
        DoB(i);
    }
}

// Better: Eliminate branch
for (idx_t i = 0; i < count; i++) {
    // Branchless: result = condition ? a : b
    result[i] = condition[i] * a[i] + (!condition[i]) * b[i];
}

// Or: Partition data
vector<idx_t> a_indices, b_indices;
for (idx_t i = 0; i < count; i++) {
    if (condition(i)) {
        a_indices.push_back(i);
    } else {
        b_indices.push_back(i);
    }
}
for (auto idx : a_indices) DoA(idx);
for (auto idx : b_indices) DoB(idx);
```

### DuckDB-Specific Patterns

#### Use Vector Operations

```cpp
// DuckDB processes data in vectors of STANDARD_VECTOR_SIZE (2048)
Vector input(LogicalType::INTEGER, STANDARD_VECTOR_SIZE);
Vector output(LogicalType::INTEGER, STANDARD_VECTOR_SIZE);

// Process entire vector at once
VectorOperations::Add(input, 10, output, STANDARD_VECTOR_SIZE);
```

#### Use Unified Vector Format

```cpp
UnifiedVectorFormat input_data;
input.ToUnifiedFormat(count, input_data);

auto input_values = UnifiedVectorFormat::GetData<int64_t>(input_data);
auto &input_validity = input_data.validity;

// Sequential access to data
for (idx_t i = 0; i < count; i++) {
    auto idx = input_data.sel->get_index(i);
    if (input_validity.RowIsValid(idx)) {
        Process(input_values[idx]);
    }
}
```

---

## Benchmarking Methodology

### Principles of Good Benchmarking

1. **Warmup**: Run query once to warm caches before measuring
2. **Multiple Runs**: Run at least 3-5 times, report median or mean
3. **Stable Environment**: Minimize background processes
4. **Consistent Hardware**: Use same machine for comparisons
5. **Version Control**: Track code version for each benchmark
6. **Automated**: Automate benchmarking to run regularly

### Setting Up Benchmarks

#### 1. Choose Representative Workload

```sql
-- Micro-benchmark: Specific operation
SELECT SUM(value) FROM table;

-- Macro-benchmark: Complex query
SELECT customer_id, COUNT(*), SUM(total)
FROM orders
WHERE date >= '2024-01-01'
GROUP BY customer_id
HAVING COUNT(*) > 5
ORDER BY SUM(total) DESC
LIMIT 100;

-- End-to-end: Full TPC-H suite
-- Tests complete system
```

#### 2. Control Data Size

```sql
-- Generate consistent test data
CREATE TABLE test_data AS
SELECT range AS id,
       (range * 12345) % 100 AS category,
       range * 1.5 AS value
FROM range(1000000);
```

Use TPC-H for standard benchmark data:

```sql
-- Load TPC-H SF1 (1GB)
CALL dbgen(sf=1);

-- Load TPC-H SF10 (10GB)
CALL dbgen(sf=10);
```

#### 3. Warm Up Caches

```python
# Python example
import duckdb

conn = duckdb.connect()

# Warmup run (not measured)
conn.execute("SELECT * FROM large_table").fetchall()

# Measured runs
import time
times = []
for i in range(5):
    start = time.time()
    conn.execute("SELECT * FROM large_table").fetchall()
    end = time.time()
    times.append(end - start)

print(f"Mean: {sum(times) / len(times):.3f}s")
print(f"Median: {sorted(times)[len(times)//2]:.3f}s")
```

### Benchmark Comparison

#### Before-After Testing

```bash
# Baseline (before changes)
git checkout main
make clean && make release
build/release/benchmark/benchmark_runner "Q01" --out=baseline.csv

# After changes
git checkout my-optimization-branch
make clean && make release
build/release/benchmark/benchmark_runner "Q01" --out=optimized.csv

# Compare results
python compare_benchmarks.py baseline.csv optimized.csv
```

#### Statistical Significance

Run enough iterations to detect meaningful differences:

```python
import numpy as np
from scipy import stats

baseline = [1.23, 1.25, 1.24, 1.26, 1.23]
optimized = [1.15, 1.14, 1.16, 1.15, 1.14]

# T-test
t_stat, p_value = stats.ttest_ind(baseline, optimized)
print(f"P-value: {p_value}")  # < 0.05 means significant

# Effect size
mean_improvement = (np.mean(baseline) - np.mean(optimized)) / np.mean(baseline)
print(f"Improvement: {mean_improvement * 100:.1f}%")
```

### Benchmarking Best Practices

#### DO:
- Run benchmarks on isolated, consistent hardware
- Measure wall-clock time, not CPU time (for I/O-bound workloads)
- Report variance/stddev along with mean
- Use release builds for performance benchmarks
- Keep benchmark data in version control
- Automate benchmark runs in CI

#### DON'T:
- Run benchmarks on laptop with background processes
- Compare benchmarks across different machines
- Cherry-pick best run (report all runs or use statistics)
- Use debug builds for performance measurement
- Change data between benchmark runs
- Ignore variance (high variance = unreliable benchmark)

### Creating a Benchmark Suite

```bash
# benchmark/my_benchmarks/scan.benchmark
name ScanWithFilter
group scan

load
CREATE TABLE test AS SELECT * FROM range(10000000);

run
SELECT SUM(i) FROM test WHERE i % 100 = 0;

# Run suite
build/release/benchmark/benchmark_runner "scan"
```

---

## Performance Testing and Regression Detection

### Continuous Performance Testing

#### 1. Automated Benchmark Runs

```bash
# Run benchmarks on every commit
#!/bin/bash
set -e

BUILD_BENCHMARK=1 make clean && make release
mkdir -p benchmark_results

DATE=$(date +%Y%m%d_%H%M%S)
COMMIT=$(git rev-parse --short HEAD)

build/release/benchmark/benchmark_runner \
    --out=benchmark_results/results_${COMMIT}_${DATE}.csv

# Compare with baseline
python scripts/compare_performance.py \
    benchmark_results/baseline.csv \
    benchmark_results/results_${COMMIT}_${DATE}.csv
```

#### 2. Regression Detection

```python
# scripts/compare_performance.py
import pandas as pd
import sys

THRESHOLD = 0.05  # 5% slowdown is regression

baseline = pd.read_csv(sys.argv[1])
current = pd.read_csv(sys.argv[2])

merged = baseline.merge(current, on='name', suffixes=('_baseline', '_current'))

regressions = []
for _, row in merged.iterrows():
    baseline_time = row['timing_baseline']
    current_time = row['timing_current']

    slowdown = (current_time - baseline_time) / baseline_time

    if slowdown > THRESHOLD:
        regressions.append({
            'name': row['name'],
            'baseline': baseline_time,
            'current': current_time,
            'slowdown': f"{slowdown * 100:.1f}%"
        })

if regressions:
    print("Performance regressions detected:")
    for reg in regressions:
        print(f"  {reg['name']}: {reg['baseline']:.3f}s -> {reg['current']:.3f}s ({reg['slowdown']})")
    sys.exit(1)
else:
    print("No regressions detected")
```

### Performance Testing Workflow

#### 1. Establish Baseline

```bash
# On main branch
git checkout main
BUILD_BENCHMARK=1 make release
build/release/benchmark/benchmark_runner --out=baseline.csv
```

#### 2. Make Changes

```bash
# Create feature branch
git checkout -b optimization-feature

# Make code changes
# ...

# Test performance
BUILD_BENCHMARK=1 make release
build/release/benchmark/benchmark_runner --out=optimized.csv

# Compare
python scripts/compare_performance.py baseline.csv optimized.csv
```

#### 3. Iterate

If performance regressed:
- Profile to understand why
- Adjust implementation
- Re-benchmark
- Repeat until improvement achieved

#### 4. Document Results

```markdown
## Performance Results

Benchmark: TPC-H Q01
- Baseline: 1.234s
- Optimized: 0.987s
- Improvement: 20.0%

Changes:
- Added index on l_shipdate
- Enabled perfect hash aggregation for l_returnflag

Profiling showed:
- 40% time in sequential scan (reduced by index)
- 30% time in hash aggregation (reduced by perfect hash)
```

### Identifying Performance Regressions

#### Bisecting Regressions

```bash
# Find commit that introduced regression
git bisect start
git bisect bad HEAD  # Current version is slow
git bisect good v1.0.0  # v1.0.0 was fast

# Git will checkout middle commit
BUILD_BENCHMARK=1 make release
build/release/benchmark/benchmark_runner "SlowQuery" --out=test.csv

# If slow:
git bisect bad

# If fast:
git bisect good

# Repeat until found
```

#### Automated Bisection

```bash
#!/bin/bash
# bisect_helper.sh

BUILD_BENCHMARK=1 make clean && make release
RESULT=$(build/release/benchmark/benchmark_runner "Q01" | grep "timing" | awk '{print $3}')

# Compare with threshold (e.g., 1.5s)
if (( $(echo "$RESULT > 1.5" | bc -l) )); then
    exit 1  # Bad
else
    exit 0  # Good
fi

# Run bisect
git bisect run ./bisect_helper.sh
```

### Performance Monitoring Dashboard

Track performance over time:

```python
# Store results in database
import sqlite3

conn = sqlite3.connect('performance_history.db')
conn.execute('''
    CREATE TABLE IF NOT EXISTS benchmarks (
        id INTEGER PRIMARY KEY,
        timestamp TEXT,
        commit TEXT,
        benchmark_name TEXT,
        timing REAL
    )
''')

# Insert results
import datetime
timestamp = datetime.datetime.now().isoformat()
commit = subprocess.check_output(['git', 'rev-parse', 'HEAD']).decode().strip()

for benchmark, timing in results.items():
    conn.execute('''
        INSERT INTO benchmarks (timestamp, commit, benchmark_name, timing)
        VALUES (?, ?, ?, ?)
    ''', (timestamp, commit, benchmark, timing))

conn.commit()
```

Plot over time:

```python
import matplotlib.pyplot as plt

df = pd.read_sql('SELECT * FROM benchmarks WHERE benchmark_name = "Q01"', conn)
plt.plot(df['timestamp'], df['timing'])
plt.xlabel('Time')
plt.ylabel('Query Time (s)')
plt.title('TPC-H Q01 Performance Over Time')
plt.show()
```

---

## Summary and Quick Reference

### Quick Performance Checklist

- [ ] Profile before optimizing (EXPLAIN ANALYZE)
- [ ] Check cardinality estimates (update statistics if needed)
- [ ] Verify indexes are being used
- [ ] Look for expensive operators (CROSS_PRODUCT, full scans)
- [ ] Check for filter pushdown opportunities
- [ ] Minimize data read (projection pushdown)
- [ ] Optimize join order (smaller table on build side)
- [ ] Use appropriate data formats (Parquet > CSV > JSON)
- [ ] Monitor memory usage (avoid spilling if possible)
- [ ] Benchmark before and after changes

### Common Commands

```bash
# Build and run benchmarks
BUILD_BENCHMARK=1 make release
build/release/benchmark/benchmark_runner "BenchmarkName"

# Profile query
echo "PRAGMA enable_profiling='json'; SELECT ..." | build/release/duckdb

# System profiling (Linux)
perf record -g build/release/duckdb < query.sql
perf report

# Check for regressions
python scripts/compare_performance.py baseline.csv current.csv
```

### SQL Performance Tips

```sql
-- Enable profiling
PRAGMA enable_profiling='query_tree';

-- Set memory limit
SET memory_limit='16GB';

-- Set thread count
SET threads=8;

-- Update statistics
ANALYZE table_name;

-- Create index
CREATE INDEX idx_name ON table(column);

-- Check memory usage
SELECT * FROM duckdb_memory();

-- Check temporary files (spilling)
SELECT * FROM duckdb_temporary_files();
```

### Key Performance Metrics

- **Latency**: Total query time
- **Throughput**: Queries per second
- **CPU utilization**: Are cores fully utilized?
- **Memory usage**: Peak memory, spilling?
- **I/O**: Bytes read/written
- **Cardinality**: Rows processed at each stage

### When to Optimize

**Optimize when**:
- Query is too slow for use case
- Performance regressed from previous version
- Memory usage is too high
- System resources are bottleneck

**Don't optimize when**:
- Query is already fast enough
- Optimization adds complexity
- No clear bottleneck identified
- Haven't profiled yet

### Remember

> "Premature optimization is the root of all evil" - Donald Knuth

> "Measure, don't guess" - Performance Engineering Wisdom

**Always profile before optimizing. Make one change at a time. Measure the impact.**

---

## Additional Resources

- **Architecture Overview**: `learning/architecture-deep-dive.md`
- **Vectorized Execution**: `learning/vectorized-execution.md`
- **Debugging Tips**: `learning/debugging-tips.md`
- **Testing Guide**: `learning/testing-guide.md`
- **CLAUDE.md**: `CLAUDE.md`

### Code References

- **Query Profiler**: `src/main/query_profiler.cpp`, `src/include/duckdb/main/query_profiler.hpp`
- **Profiling Info**: `src/include/duckdb/main/profiling_info.hpp`
- **Metrics Types**: `src/include/duckdb/common/enums/metric_type.hpp`
- **Benchmark Framework**: `benchmark/include/`, `benchmark/benchmark_runner.cpp`
- **Vector Operations**: `src/common/vector_operations/`
- **Physical Operators**: `src/execution/operator/`
- **Optimizer**: `src/optimizer/`
- **Parallel Execution**: `src/parallel/`, `src/execution/executor.cpp`

---

**This guide is a living document. Performance optimization is an ongoing process. Profile, optimize, and measure continuously.**
