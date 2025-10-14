# DuckDB Debugging Tips for Developers

This guide provides comprehensive debugging techniques and workflows for DuckDB developers. It covers everything from basic debug builds to advanced debugging scenarios.

## Table of Contents

1. [Debug Builds](#debug-builds)
2. [Debugger Basics](#debugger-basics)
3. [Sanitizers](#sanitizers)
4. [Debugging Crashes and Segfaults](#debugging-crashes-and-segfaults)
5. [Performance Debugging](#performance-debugging)
6. [Using D_ASSERT Effectively](#using-d_assert-effectively)
7. [Logging and Debug Output](#logging-and-debug-output)
8. [Common Error Patterns](#common-error-patterns)
9. [Debugging the Parser](#debugging-the-parser)
10. [Debugging the Optimizer](#debugging-the-optimizer)
11. [Debugging Execution Engine](#debugging-execution-engine)
12. [Memory Leak Detection](#memory-leak-detection)
13. [Debugging Extensions](#debugging-extensions)
14. [Advanced Debug Flags](#advanced-debug-flags)

---

## Debug Builds

### Basic Debug Build

The debug build enables assertions, disables optimizations, and enables sanitizers by default:

```bash
make debug
```

This creates a debug build in `build/debug/` with:
- Assertions enabled (`D_ASSERT` checks)
- AddressSanitizer (ASan) enabled
- UndefinedBehaviorSanitizer (UBSan) enabled
- Debug symbols for debugger use
- No compiler optimizations (-O0)

### Running Tests with Debug Build

```bash
# Run all fast unit tests
make unit

# Run specific test
build/debug/test/unittest "TestName"

# Run tests with a specific tag
build/debug/test/unittest "[arrow]"
```

### Alternative Debug Builds

```bash
# Debug build without sanitizers (faster compilation/execution)
DISABLE_SANITIZER=1 make debug

# Release build with debug info (RelWithDebInfo)
make reldebug

# Release build with assertions enabled
make relassert

# Clean debug build (no unity builds, useful for better error messages)
DISABLE_UNITY=1 make debug
```

### Build with Parallel Jobs

```bash
# Use Ninja for faster parallel builds
GEN=ninja make debug

# Limit parallel jobs (useful on memory-constrained systems)
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make debug
```

---

## Debugger Basics

### GDB (Linux and Some macOS)

#### Starting GDB

```bash
# Debug the unittest binary
gdb build/debug/test/unittest

# Debug with arguments
gdb --args build/debug/test/unittest "TestName"

# Attach to running process
gdb -p <pid>
```

#### Essential GDB Commands

```gdb
# Run the program
run

# Run with arguments
run "TestName"

# Set breakpoint
break filename.cpp:123
break FunctionName
break duckdb::Parser::ParseQuery

# Conditional breakpoint
break filename.cpp:123 if variable == value

# Watch variable changes
watch variable_name

# Print variable
print variable_name
print *pointer

# Print with pretty printing
print /x variable  # hexadecimal
print /d variable  # decimal
print /t variable  # binary

# Examine memory
x/10x address  # 10 words in hex
x/s string_ptr # string

# Stack trace
backtrace
backtrace full  # with local variables
frame N         # switch to frame N
up/down         # move up/down stack

# Step through code
next    # step over (n)
step    # step into (s)
finish  # step out
continue # continue execution (c)

# Info commands
info breakpoints
info threads
info locals
info args

# Delete breakpoints
delete <number>
clear function_name
```

#### GDB with DuckDB-Specific Tips

```gdb
# Break on D_ASSERT failures
break duckdb::DuckDBAssertInternal

# Break on exceptions
catch throw

# Break on specific exception type
catch throw duckdb::InternalException

# Print DuckDB strings
print string_variable.GetString()

# Print Vector contents
print vector.GetData()
print vector.GetValidity()
```

#### GDB Configuration (.gdbinit)

Create `~/.gdbinit` for persistent settings:

```gdb
# Enable pretty printing
set print pretty on
set print object on
set print static-members on
set print vtbl on
set print demangle on

# History
set history save on
set history size 10000

# Break on assert
catch throw
```

### LLDB (macOS)

LLDB is the default debugger on macOS and has similar but slightly different syntax than GDB.

#### Starting LLDB

```bash
# Debug the unittest binary
lldb build/debug/test/unittest

# Debug with arguments
lldb -- build/debug/test/unittest "TestName"

# Attach to running process
lldb -p <pid>
```

#### Essential LLDB Commands

```lldb
# Run the program
run
r "TestName"

# Set breakpoint
breakpoint set --file filename.cpp --line 123
breakpoint set --name FunctionName
b filename.cpp:123
b FunctionName

# Conditional breakpoint
breakpoint set --file filename.cpp --line 123 --condition 'variable == value'

# Watch variable
watchpoint set variable variable_name
watchpoint set expression -- &variable

# Print variable
print variable_name
p *pointer
frame variable  # print all local variables
v variable_name # shortcut for frame variable

# Examine memory
memory read address
x address

# Stack trace
thread backtrace
bt
frame select N
up/down

# Step through code
next     # step over (n)
step     # step into (s)
finish   # step out
continue # continue execution (c)

# Info commands
breakpoint list
thread list
frame variable

# Delete breakpoints
breakpoint delete <number>
breakpoint delete
```

#### LLDB with DuckDB-Specific Tips

```lldb
# Break on D_ASSERT failures
breakpoint set --name duckdb::DuckDBAssertInternal

# Break on exceptions
breakpoint set -E C++

# Break on specific exception
breakpoint set -F 'duckdb::InternalException::InternalException'

# Print DuckDB strings
print string_variable.GetString()
```

#### LLDB Configuration (.lldbinit)

Create `~/.lldbinit` for persistent settings:

```lldb
# Better formatting
settings set target.x86-disassembly-flavor intel
settings set target.process.stop-on-exec false

# Break on exceptions
breakpoint set -E C++
```

---

## Sanitizers

DuckDB uses sanitizers to catch bugs at runtime. They are enabled by default in debug builds.

### AddressSanitizer (ASan)

Detects memory errors: use-after-free, buffer overflows, memory leaks, etc.

**Enabled by default in debug builds.**

```bash
# Build with ASan (default)
make debug

# Disable ASan
DISABLE_SANITIZER=1 make debug

# Force ASan in release build
FORCE_SANITIZER=1 make release
```

#### ASan Output

When ASan detects an error:

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x...
READ of size 8 at 0x... thread T0
    #0 0x... in Function /path/to/file.cpp:123
    #1 0x... in Caller /path/to/file.cpp:456
    ...
```

#### Debugging ASan Errors

1. Look at the stack trace to find where the invalid access occurred
2. Look for "freed by thread" to find where memory was deallocated
3. Look for "allocated by thread" to find where memory was allocated
4. Set breakpoint before the error and inspect memory state

#### ASan Environment Variables

```bash
# More verbose output
ASAN_OPTIONS=verbosity=1 build/debug/test/unittest

# Disable leak checking (faster, useful for quick tests)
ASAN_OPTIONS=detect_leaks=0 build/debug/test/unittest

# Check for stack-use-after-return
ASAN_OPTIONS=detect_stack_use_after_return=1 build/debug/test/unittest
```

### UndefinedBehaviorSanitizer (UBSan)

Detects undefined behavior: integer overflow, null pointer dereference, misaligned pointers, etc.

**Enabled by default in debug builds.**

```bash
# Disable UBSan only
DISABLE_UBSAN=1 make debug

# Disable vptr checks (useful for M1 Mac false positives)
DISABLE_VPTR_SANITIZER=1 make debug
```

#### UBSan Output

```
/path/to/file.cpp:123:45: runtime error: signed integer overflow:
2147483647 + 1 cannot be represented in type 'int'
```

### ThreadSanitizer (TSan)

Detects data races and threading issues.

**NOT enabled by default** (conflicts with ASan).

```bash
# Build with ThreadSanitizer
THREADSAN=1 make debug

# Run tests with TSan
THREADSAN=1 make debug && build/debug/test/unittest
```

#### TSan Output

```
==================
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x... by thread T1:
    #0 Function /path/to/file.cpp:123
  Previous read of size 4 at 0x... by main thread:
    #0 OtherFunction /path/to/file.cpp:456
```

#### TSan Tips

- TSan cannot run with ASan simultaneously
- TSan has higher memory overhead
- Use TSan when debugging parallel execution issues
- False positives can occur; analyze carefully

---

## Debugging Crashes and Segfaults

### Getting a Stack Trace

#### Option 1: Run Under Debugger

```bash
# GDB
gdb --args build/debug/test/unittest "TestName"
(gdb) run
# When it crashes:
(gdb) backtrace

# LLDB
lldb -- build/debug/test/unittest "TestName"
(lldb) run
# When it crashes:
(lldb) bt
```

#### Option 2: Core Dumps (Linux)

```bash
# Enable core dumps
ulimit -c unlimited

# Run the program
build/debug/test/unittest

# After crash, analyze core dump
gdb build/debug/test/unittest core
(gdb) backtrace
```

#### Option 3: DEBUG_STACKTRACE Flag

```bash
# Build with automatic stack trace on crashes
DEBUG_STACKTRACE=1 make debug

# Now crashes will automatically print stack traces
build/debug/test/unittest
```

### Debugging Segfaults

#### Common Causes

1. **Null pointer dereference**: Most common
2. **Use-after-free**: Accessing freed memory
3. **Buffer overflow**: Writing past array bounds
4. **Stack overflow**: Deep recursion or large stack allocations
5. **Invalid cast**: Casting to wrong type

#### Debugging Workflow

```bash
# Step 1: Build with sanitizers (catches most issues)
make debug
build/debug/test/unittest

# Step 2: If sanitizer doesn't catch it, run under debugger
gdb --args build/debug/test/unittest "TestName"
(gdb) run
# When it crashes
(gdb) backtrace
(gdb) frame 0
(gdb) print variable_name
(gdb) info locals

# Step 3: Set breakpoint before crash and inspect state
(gdb) break suspicious_function
(gdb) run
(gdb) print pointer_variable
(gdb) print *pointer_variable
```

### Crash on Assert

By default, DuckDB throws an exception on assertion failure. To get a core dump instead:

```bash
# Build with CRASH_ON_ASSERT
CRASH_ON_ASSERT=1 make debug

# Now D_ASSERT failures will trigger SIGABRT
build/debug/test/unittest
```

---

## Performance Debugging

### Profiling with EXPLAIN ANALYZE

```sql
-- Basic query profiling
PRAGMA enable_profiling;
SELECT * FROM my_table WHERE condition;

-- More detailed profiling output
PRAGMA enable_profiling='query_tree';
SELECT * FROM my_table WHERE condition;

-- JSON output for programmatic analysis
PRAGMA enable_profiling='json';
SELECT * FROM my_table WHERE condition;
```

### Profiling Output Modes

```sql
-- Query tree (default)
PRAGMA enable_profiling='query_tree';

-- Optimizer profiling
PRAGMA enable_profiling='query_tree_optimizer';

-- JSON format
PRAGMA enable_profiling='json';

-- Disable profiling
PRAGMA disable_profiling;
```

### Using EXPLAIN

```sql
-- See query plan without execution
EXPLAIN SELECT * FROM my_table WHERE condition;

-- See query plan with execution and timing
EXPLAIN ANALYZE SELECT * FROM my_table WHERE condition;
```

### Benchmark Framework

```bash
# Build with benchmarks
BUILD_BENCHMARK=1 make

# Run specific benchmark
build/release/benchmark/benchmark_runner "benchmark_name"

# Run all benchmarks
build/release/benchmark/benchmark_runner
```

### CPU Profiling with Perf (Linux)

```bash
# Install perf
sudo apt-get install linux-tools-common linux-tools-generic

# Profile the test
perf record -g build/release/test/unittest "TestName"

# Analyze results
perf report

# Generate flamegraph
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### Profiling on macOS (Instruments)

```bash
# Build release with debug info
make reldebug

# Profile with Instruments
instruments -t "Time Profiler" build/reldebug/test/unittest "TestName"

# Or use the Instruments GUI
open -a Instruments
```

---

## Using D_ASSERT Effectively

### What is D_ASSERT?

`D_ASSERT` is DuckDB's assertion macro that checks conditions during development:
- Enabled in debug builds
- Disabled in release builds (unless `FORCE_ASSERT=1`)
- Should check for programmer errors, not user input errors

### When to Use D_ASSERT

```cpp
// ✓ GOOD: Check invariants
D_ASSERT(count >= 0);
D_ASSERT(index < array_size);
D_ASSERT(pointer != nullptr);

// ✓ GOOD: Check postconditions
D_ASSERT(result.IsValid());
D_ASSERT(output.size() == input.size());

// ✗ BAD: Check user input (use exceptions instead)
// D_ASSERT(sql_is_valid); // NO! Throw ParserException instead

// ✗ BAD: Check external state (use exceptions instead)
// D_ASSERT(file_exists); // NO! Throw IOException instead
```

### D_ASSERT Best Practices

```cpp
// Add descriptive comments explaining what went wrong
// "Element index out of range - indicates a bug in the caller"
D_ASSERT(idx < elements.size());

// Check complex invariants
D_ASSERT(left_child->parent == this && right_child->parent == this);

// Verify algorithm preconditions
D_ASSERT(IsSorted(input)); // comment: "Input must be sorted by caller"
```

### FORCE_ASSERT for Release Builds

```bash
# Enable assertions in release build (for debugging performance issues)
FORCE_ASSERT=1 make release

# Or relassert build (release + assertions)
make relassert
```

### ALWAYS_ASSERT

For checks that should always run, even in release builds:

```cpp
// Always check, even in release builds
ALWAYS_ASSERT(critical_invariant_holds);
```

---

## Logging and Debug Output

### Printer Class

```cpp
#include "duckdb/common/printer.hpp"

// Print to stderr
Printer::Print("Debug message: " + value);

// Print to stdout or stderr
Printer::Print(OutputStream::STREAM_STDERR, message);
Printer::Print(OutputStream::STREAM_STDOUT, message);

// Flush output
Printer::Flush(OutputStream::STREAM_STDERR);
```

### Debug Print Helper

```cpp
// Quick debug printing in development
printf("DEBUG: value=%d\n", value);
fprintf(stderr, "DEBUG: %s\n", str.c_str());

// Remember to remove before committing!
```

### Query Logging

```bash
# Force query logging (useful for debugging)
FORCE_QUERY_LOG=1 make debug

# Now all queries will be logged
build/debug/test/unittest
```

### Tree Rendering

```cpp
#include "duckdb/common/tree_renderer.hpp"

// Render logical plan
auto renderer = TreeRenderer::CreateRenderer(ExplainFormat::DEFAULT);
logical_plan.ToStream(renderer, ss);

// Render physical plan
physical_plan.ToStream(renderer, ss);
```

### Vector Type Debugging

```sql
-- Built-in debug function to inspect vector types
SELECT vector_type(column_name) FROM table_name;
```

---

## Common Error Patterns

### Null Pointer Dereference

**Symptoms**: Segfault when accessing object members

**Detection**:
```bash
# ASan will catch this
make debug
build/debug/test/unittest
```

**Prevention**:
```cpp
// Check before dereferencing
D_ASSERT(ptr != nullptr);
if (!ptr) {
    throw InternalException("Unexpected null pointer");
}

// Use references instead of pointers when possible
void Function(Object &obj);  // Can't be null
```

### Use-After-Free

**Symptoms**: Random crashes, corrupted data

**Detection**:
```bash
# ASan is excellent at catching these
make debug
build/debug/test/unittest
```

**Common Causes**:
```cpp
// Storing reference to temporary
const auto &ref = GetTemporary(); // Temporary destroyed!
UseValue(ref); // Use-after-free

// Storing pointer into vector that gets reallocated
auto *ptr = &vec[0];
vec.push_back(item); // Vector may reallocate!
UsePointer(ptr); // May be invalid

// Manual memory management errors (avoid!)
```

### Buffer Overflow

**Symptoms**: Crashes, corrupted data, weird behavior

**Detection**:
```bash
# ASan catches these
make debug
build/debug/test/unittest
```

**Prevention**:
```cpp
// Use bounds-checked access
D_ASSERT(idx < vector.size());
auto value = vector[idx];

// Use range-based for loops
for (const auto &item : vector) {
    Process(item);
}

// Use standard containers with automatic sizing
```

### Integer Overflow

**Symptoms**: Wrong results, unexpected negative numbers

**Detection**:
```bash
# UBSan catches these
make debug
build/debug/test/unittest
```

**Prevention**:
```cpp
// Use appropriate types
idx_t count; // Not int or size_t
int64_t value; // Not int

// Check before arithmetic
if (a > NumericLimits<int64_t>::Maximum() - b) {
    throw OutOfRangeException("Integer overflow");
}
```

### Memory Leaks

**Symptoms**: Growing memory usage over time

**Detection**: See [Memory Leak Detection](#memory-leak-detection)

---

## Debugging the Parser

### Parser Overview

The parser converts SQL text into a parse tree (`SQLStatement` objects).

### Common Parser Issues

1. **Syntax errors**: Wrong SQL syntax
2. **Ambiguous grammar**: Multiple parse interpretations
3. **Keyword conflicts**: Reserved words used as identifiers

### Debugging Parser Issues

```cpp
// Enable parser debugging (if needed, modify parser code)
#include "duckdb/parser/parser.hpp"

Parser parser;
parser.ParseQuery(sql);

// Inspect parsed statements
for (auto &statement : parser.statements) {
    // Print statement type
    Printer::Print(StatementTypeToString(statement->type));
}
```

### Parser Tests

```bash
# Run parser-specific tests
build/debug/test/unittest "[parser]"

# Test files in test/sql/parser/
ls test/sql/parser/
```

### Parser Error Messages

Parser errors include:
- Line and column numbers
- Syntax error details
- Context around the error

```cpp
// Parser throws ParserException on syntax errors
try {
    parser.ParseQuery(sql);
} catch (ParserException &e) {
    Printer::Print(e.what());
}
```

---

## Debugging the Optimizer

### Optimizer Overview

The optimizer transforms logical plans into more efficient equivalent plans.

### Optimizer Rules

Located in `src/optimizer/`:
- `filter_pushdown.cpp`: Push filters down
- `join_order_optimizer.cpp`: Optimize join order
- `expression_rewriter.cpp`: Rewrite expressions
- `statistics_propagation.cpp`: Propagate statistics
- And many more...

### Debugging Optimizer Issues

#### Disable Specific Optimizers

```sql
-- Disable join order optimizer
PRAGMA disable_optimizer;
PRAGMA enable_optimizer='filter_pushdown';
PRAGMA enable_optimizer='expression_rewriter';
-- (enable each optimizer you want)

-- Or disable specific optimizer
PRAGMA disabled_optimizers='join_order';
```

#### View Optimizer Steps

```sql
-- Profile optimizer steps
PRAGMA enable_profiling='query_tree_optimizer';
SELECT ...;
```

#### Compare Plans

```sql
-- Get unoptimized plan
PRAGMA disable_optimizer;
EXPLAIN SELECT ...;

-- Get optimized plan
PRAGMA enable_optimizer;
EXPLAIN SELECT ...;
```

### Optimizer Tests

```bash
# Run optimizer tests
build/debug/test/unittest "[optimizer]"

# Test files in test/sql/optimizer/
ls test/sql/optimizer/
```

### Debugging Optimizer Rule

```cpp
// Add debug output in optimizer rule
void MyOptimizer::Optimize(unique_ptr<LogicalOperator> &op) {
    Printer::Print("Before optimization:");
    op->Print();

    // Optimization logic

    Printer::Print("After optimization:");
    op->Print();
}
```

---

## Debugging Execution Engine

### Execution Overview

The execution engine runs physical operators to execute queries.

### Common Execution Issues

1. **Incorrect results**: Wrong output
2. **Performance problems**: Slow execution
3. **Crashes during execution**: Segfaults, assertions
4. **Resource issues**: Out of memory

### Debugging Execution Issues

#### View Physical Plan

```sql
EXPLAIN SELECT ...;
```

#### Profile Execution

```sql
PRAGMA enable_profiling;
SELECT ...;
```

#### Debug Specific Operator

```cpp
// Add debug output in operator
void PhysicalMyOperator::Execute(ExecutionContext &context) {
    Printer::Print("Executing MyOperator");
    Printer::Print("Input rows: " + to_string(input.size()));

    // Execution logic

    Printer::Print("Output rows: " + to_string(output.size()));
}
```

#### Verify Vectors

```bash
# Enable vector verification
VERIFY_VECTOR=dictionary_expression make debug
build/debug/test/unittest

# Options:
# - dictionary_expression
# - dictionary_operator
# - constant_operator
# - sequence_operator
# - nested_shuffle
# - variant_vector
```

### Execution Tests

```bash
# Run execution tests
build/debug/test/unittest "[execution]"

# Test files in test/sql/
ls test/sql/
```

---

## Memory Leak Detection

### Built-in Memory Leak Tests

```bash
# Run memory leak tests
python3 test/memoryleak/test_memory_leaks.py

# Run specific test
python3 test/memoryleak/test_memory_leaks.py --test="Test name"

# Run with debugger
build/debug/test/unittest "Test name" --memory-leak-tests
```

### DEBUG_ALLOCATION Flag

Track all allocations to find leaks:

```bash
# Build with allocation tracking
DEBUG_ALLOCATION=1 make debug

# Run test - will report outstanding allocations on exit
build/debug/test/unittest
```

**Output on leak**:
```
Outstanding allocations found for Allocator
Allocated at /path/to/file.cpp:123
Size: 1024 bytes
Stack trace:
  #0 AllocateData
  #1 MyFunction
  ...
```

### ASan Leak Detection

```bash
# ASan detects leaks by default
make debug
build/debug/test/unittest

# Increase leak detection verbosity
ASAN_OPTIONS=verbosity=1:detect_leaks=1 build/debug/test/unittest

# Just check for leaks at exit
ASAN_OPTIONS=detect_leaks=1 build/debug/test/unittest
```

### Valgrind (Linux)

```bash
# Install valgrind
sudo apt-get install valgrind

# Build without sanitizers (conflicts with valgrind)
DISABLE_SANITIZER=1 make debug

# Run with valgrind
valgrind --leak-check=full --show-leak-kinds=all \
    build/debug/test/unittest "TestName"

# Valgrind with suppression file
valgrind --leak-check=full --suppressions=duckdb.supp \
    build/debug/test/unittest
```

### Common Leak Patterns

```cpp
// ✗ BAD: Missing unique_ptr
auto *ptr = new Object();
// ... forgot to delete

// ✓ GOOD: Use smart pointers
auto ptr = make_uniq<Object>();

// ✗ BAD: Circular references with shared_ptr
class Node {
    shared_ptr<Node> parent; // Creates cycle!
    shared_ptr<Node> child;
};

// ✓ GOOD: Break cycles with weak_ptr or raw pointers
class Node {
    Node *parent; // Raw pointer, no ownership
    unique_ptr<Node> child; // Unique ownership
};
```

---

## Debugging Extensions

### Building Extensions

```bash
# Build with specific extensions
DUCKDB_EXTENSIONS="parquet;json" make debug

# Build all in-tree extensions
BUILD_ALL_IT_EXT=1 make debug
```

### Testing Extensions

```bash
# Run extension tests
build/debug/test/unittest "[parquet]"
build/debug/test/unittest "[json]"

# Test extension loading
build/debug/duckdb
> LOAD 'parquet';
> SELECT * FROM parquet_scan('file.parquet');
```

### Debugging Extension Loading

```cpp
// Extension code is in extension/ directory
// Example: extension/parquet/

// Enable debug output in extension
#include "duckdb/common/printer.hpp"

void ParquetExtension::Load(DuckDB &db) {
    Printer::Print("Loading Parquet extension");
    // Loading logic
}
```

### Common Extension Issues

1. **Loading failures**: Missing dependencies, ABI mismatch
2. **Incorrect results**: Bug in extension code
3. **Crashes**: Memory issues in extension

### Extension Tests

```bash
# Extension tests are in test/sql/<extension>/
ls test/sql/parquet/
ls test/sql/json/

# Run extension-specific tests
build/debug/test/unittest test/sql/parquet/
```

---

## Advanced Debug Flags

### Complete List of Debug Flags

```bash
# Core debugging
DEBUG_STACKTRACE=1      # Print stack traces on crashes/asserts
DEBUG_ALLOCATION=1      # Track all allocations
DEBUG_MOVE=1            # Verify std::move usage
CRASH_ON_ASSERT=1       # Trigger SIGABRT on assert failure
FORCE_ASSERT=1          # Enable asserts in release builds
FORCE_DEBUG=1           # Add debug define in release builds

# Build options
DISABLE_SANITIZER=1     # Disable all sanitizers
DISABLE_UBSAN=1         # Disable UBSan only
DISABLE_VPTR_SANITIZER=1 # Disable vptr sanitizer (M1 Mac fix)
THREADSAN=1             # Enable ThreadSanitizer
DISABLE_UNITY=1         # Disable unity builds

# Testing and verification
VERIFY_VECTOR=option    # Verify vector operations
BLOCK_VERIFICATION=1    # Verify block operations
RUN_SLOW_VERIFIERS=1    # Run slow verification checks
ALTERNATIVE_VERIFY=1    # Use alternative verification

# Memory and performance
DESTROY_UNPINNED_BLOCKS=1 # Destroy unpinned blocks (catch use-after-free)
DISABLE_MEMORY_SAFETY=1   # Disable memory safety checks
FORCE_ASYNC_SINK_SOURCE=1 # Test async sink/source

# Other
FORCE_QUERY_LOG=1       # Log all queries
STATIC_LIBCPP=1         # Statically link C++ stdlib
NATIVE_ARCH=1           # Optimize for native architecture
```

### Using Multiple Flags

```bash
# Combine flags
DEBUG_STACKTRACE=1 DEBUG_ALLOCATION=1 DISABLE_UNITY=1 make debug

# Use environment variables
export DEBUG_STACKTRACE=1
export DEBUG_ALLOCATION=1
make debug
```

### Recommended Debug Configurations

```bash
# Maximum debugging (slowest, catches most bugs)
DEBUG_STACKTRACE=1 DEBUG_ALLOCATION=1 DISABLE_UNITY=1 make debug

# Fast debugging (quick compile, still safe)
DISABLE_SANITIZER=1 make debug

# Memory leak hunting
DEBUG_ALLOCATION=1 make debug

# Crash debugging
DEBUG_STACKTRACE=1 CRASH_ON_ASSERT=1 make debug

# Threading issues
THREADSAN=1 make debug

# Performance debugging with assertions
FORCE_ASSERT=1 make reldebug
```

---

## Common Debugging Workflows

### Workflow 1: Test Failure

```bash
# 1. Reproduce the failure
build/debug/test/unittest "FailingTest"

# 2. Run under debugger
gdb --args build/debug/test/unittest "FailingTest"
(gdb) run
# When it fails:
(gdb) backtrace
(gdb) print variables

# 3. Add more assertions and rebuild
# Edit code, add D_ASSERT checks
make debug
gdb --args build/debug/test/unittest "FailingTest"
```

### Workflow 2: Segfault

```bash
# 1. Let sanitizer catch it
make debug
build/debug/test/unittest "CrashingTest"
# ASan will show stack trace

# 2. If sanitizer doesn't catch it, use debugger
gdb --args build/debug/test/unittest "CrashingTest"
(gdb) run
(gdb) backtrace

# 3. Enable stack traces
DEBUG_STACKTRACE=1 make debug
build/debug/test/unittest "CrashingTest"
```

### Workflow 3: Memory Leak

```bash
# 1. Use ASan leak detection
make debug
ASAN_OPTIONS=detect_leaks=1 build/debug/test/unittest

# 2. Use DEBUG_ALLOCATION for detailed tracking
DEBUG_ALLOCATION=1 make debug
build/debug/test/unittest

# 3. Run dedicated leak test
python3 test/memoryleak/test_memory_leaks.py
```

### Workflow 4: Wrong Query Result

```bash
# 1. Simplify the query
# Create minimal reproducing case

# 2. Check the plan
echo "EXPLAIN SELECT ..." | build/debug/duckdb

# 3. Profile the query
echo "PRAGMA enable_profiling; SELECT ..." | build/debug/duckdb

# 4. Disable optimizers to isolate issue
echo "PRAGMA disable_optimizer; SELECT ..." | build/debug/duckdb

# 5. Run under debugger, break in suspected operator
gdb build/debug/duckdb
(gdb) break PhysicalHashJoin::Execute
(gdb) run
# Execute query at prompt
```

### Workflow 5: Performance Issue

```bash
# 1. Profile the query
echo "PRAGMA enable_profiling='query_tree'; SELECT ..." | build/release/duckdb

# 2. Check if optimizer is helping
echo "PRAGMA disable_optimizer; EXPLAIN SELECT ..." | build/release/duckdb
echo "PRAGMA enable_optimizer; EXPLAIN SELECT ..." | build/release/duckdb

# 3. Use release build with assertions
FORCE_ASSERT=1 make reldebug

# 4. Profile with system tools
perf record -g build/release/duckdb < query.sql
perf report
```

---

## Tips and Pitfalls

### DO

- ✓ Always use debug builds during development
- ✓ Run tests frequently (`make unit`)
- ✓ Let sanitizers catch bugs early
- ✓ Use D_ASSERT liberally for invariants
- ✓ Add comments explaining what assertions check
- ✓ Use smart pointers (`unique_ptr`, not raw `new`/`delete`)
- ✓ Use range-based for loops
- ✓ Format code before committing (`make format-fix`)

### DON'T

- ✗ Don't disable sanitizers unless necessary
- ✗ Don't use raw pointers for ownership
- ✗ Don't use malloc/free/new/delete (use smart pointers)
- ✗ Don't commit debug print statements
- ✗ Don't commit commented-out code
- ✗ Don't use D_ASSERT to check user input (throw exceptions instead)
- ✗ Don't use single-letter variables in nested loops
- ✗ Don't import namespaces (`using namespace std`)

### Common Pitfalls

1. **Forgetting to rebuild**: After changing code, always rebuild
2. **Testing release build**: Use debug builds for development
3. **Ignoring sanitizer warnings**: Fix them immediately
4. **Using wrong types**: Use `idx_t`, `int64_t`, not `int`/`size_t`
5. **Not checking pointers**: Always check before dereferencing
6. **Circular dependencies**: Careful with shared_ptr
7. **Iterator invalidation**: Modifying container while iterating

---

## Additional Resources

### Documentation

- DuckDB Architecture: `CLAUDE.md`
- DuckDB Overview: `learning/overview.md`
- Test README: `test/memoryleak/README.md`

### Code Locations

- Assertions: `src/include/duckdb/common/assert.hpp`
- Exception types: `src/include/duckdb/common/exception.hpp`
- Printer: `src/common/printer.cpp`
- Parser: `src/parser/`
- Optimizer: `src/optimizer/`
- Execution: `src/execution/`

### Build System

- Main CMakeLists: `CMakeLists.txt`
- Makefile wrapper: `Makefile`
- Format script: `scripts/format.py`
- Test runner: `scripts/run_tests_one_by_one.py`

### Getting Help

- Check existing tests for examples
- Look at similar code in the codebase
- Read the component's README if available
- Use EXPLAIN and profiling to understand query execution

---

**Remember**: The best debugging strategy is prevention. Write tests, use assertions, let sanitizers run, and code review carefully. Debug builds are slow but catch bugs early. When in doubt, start with the simplest debug build and add flags as needed.
