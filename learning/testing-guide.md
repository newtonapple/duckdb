# DuckDB Testing Guide for Developers

This comprehensive guide covers everything you need to know about testing in DuckDB, from writing your first test to debugging complex failures.

## Table of Contents

1. [Testing Philosophy](#testing-philosophy)
2. [Test Framework Overview](#test-framework-overview)
3. [SQL Logic Tests (.test files)](#sql-logic-tests-test-files)
4. [C++ Unit Tests (Catch2)](#c-unit-tests-catch2)
5. [Running Tests](#running-tests)
6. [Test Organization](#test-organization)
7. [Slow Tests vs Fast Tests](#slow-tests-vs-fast-tests)
8. [Writing Effective Tests](#writing-effective-tests)
9. [Testing Different Data Types](#testing-different-data-types)
10. [Testing Error Cases](#testing-error-cases)
11. [Debugging Failing Tests](#debugging-failing-tests)
12. [CI/CD and Automated Testing](#cicd-and-automated-testing)
13. [Test-Driven Development Workflow](#test-driven-development-workflow)

---

## Testing Philosophy

DuckDB's testing philosophy emphasizes:

- **Comprehensive Coverage**: Tests should cover normal cases, edge cases, and error conditions
- **Prefer SQL Logic Tests**: Use `.test` files for most testing (simpler, more maintainable)
- **Fast by Default**: Keep most tests fast; mark slow tests explicitly
- **Regression Prevention**: Every bug fix should include a test
- **Type Coverage**: Test across different data types (integers, strings, nested types, etc.)

---

## Test Framework Overview

DuckDB uses **two complementary testing frameworks**:

### 1. SQL Logic Tests (`.test` files)
- Simple text-based format
- Located in `test/` directory
- **Strongly preferred** for new tests
- Easy to read and write
- No compilation required

### 2. C++ Unit Tests (Catch2)
- Located in `test/` directory with `.cpp` extension
- Uses the Catch2 testing framework
- Best for testing internal C++ APIs
- Requires compilation

**Rule of Thumb**: Use SQL logic tests unless you specifically need to test C++ APIs or internal behavior.

---

## SQL Logic Tests (.test files)

### Basic Structure

Every `.test` file has a header followed by test statements:

```sql
# name: test/sql/aggregate/aggregates/test_sum.test
# description: Test sum aggregate
# group: [aggregates]

statement ok
CREATE TABLE integers(i INTEGER);

statement ok
INSERT INTO integers SELECT * FROM range(0, 1000, 1);

query I
SELECT SUM(i) FROM integers;
----
499500
```

### Header Format

All `.test` files must start with:
```
# name: <path-to-file>
# description: <brief description>
# group: [<group-name>]
```

### Statement Types

#### 1. `statement ok` - Statement that should succeed
```sql
statement ok
CREATE TABLE test(i INTEGER);
```

#### 2. `statement error` - Statement that should fail
```sql
statement error
SELECT * FROM nonexistent_table;
----
<REGEX>:.*Table.*not found.*
```

The `----` line separates the statement from the expected error message. Use `<REGEX>:` prefix for regex matching.

#### 3. `query` - Query with expected results

Format: `query <type_string>`

Type string characters:
- `I` - INTEGER
- `T` - TEXT/VARCHAR
- `R` - REAL/DOUBLE
- `B` - BOOLEAN

```sql
query I
SELECT 42;
----
42

query IT
SELECT 1, 'hello';
----
1	hello

query III
SELECT * FROM range(3);
----
0
1
2
```

Results are tab-separated for multiple columns.

### Special Features

#### Loop Constructs

Test the same logic across multiple types:

```sql
foreach type <integral>

statement ok
CREATE TABLE test (i ${type})

statement ok
INSERT INTO test VALUES (100), (25), (75), (50);

query T
SELECT * FROM test ORDER BY i
----
25
50
75
100

statement ok
DROP TABLE test

endloop
```

Built-in type groups:
- `<integral>` - All integer types (tinyint, smallint, integer, bigint, hugeint, etc.)
- `<numeric>` - All numeric types
- `<signed_integral>` - Signed integer types

Custom lists:
```sql
foreach type smallint usmallint integer uinteger bigint ubigint

# test code here

endloop
```

#### Require Directives

Skip tests based on conditions:

```sql
# Require a specific extension
require parquet

# Require environment variable
require-env LOCAL_EXTENSION_REPO

# Require specific platform
require notwindows
require notmingw

# Require specific feature
require skip_reload
require allow_unsigned_extensions
```

#### Pragmas

Control test behavior:

```sql
# Enable query verification (checks results match optimized vs non-optimized)
statement ok
PRAGMA enable_verification

# Disable verification (e.g., for non-deterministic queries)
statement ok
PRAGMA disable_verification

# Disable specific verifications
statement ok
PRAGMA disable_verify_fetch_row

# Control thread count
statement ok
PRAGMA threads=4
```

### Testing Error Messages

Test expected errors with regex patterns:

```sql
statement error
SELECT 1/0;
----
<REGEX>:.*Division by zero.*

statement error
SELECT * FROM nonexistent;
----
<REGEX>:.*Table.*nonexistent.*not found.*

# Test error without checking message
statement error
SELECT CAST('abc' AS INTEGER);
```

### Special Result Values

```sql
# NULL values
query I
SELECT NULL;
----
NULL

# Empty result
query I
SELECT * FROM test WHERE i > 100;
----

# Empty string
query T
SELECT '';
----
(empty)
```

### Real-World Example

```sql
# name: test/sql/types/string/test_unicode.test
# description: Test unicode strings
# group: [string]

statement ok
CREATE TABLE emojis(id INTEGER, s VARCHAR);

statement ok
INSERT INTO emojis VALUES (1, '🦆'), (2, '🦆🍞🦆')

query IT
SELECT * FROM emojis ORDER BY id
----
1	🦆
2	🦆🍞🦆

# substring on unicode
query TT
SELECT substring(s, 1, 1), substring(s, 2, 1) FROM emojis ORDER BY id
----
🦆	(empty)
🦆	🍞

# length on emojis
query I
SELECT length(s) FROM emojis ORDER BY id
----
1
3
```

---

## C++ Unit Tests (Catch2)

### Basic Structure

```cpp
#include "catch.hpp"
#include "test_helpers.hpp"

using namespace duckdb;
using namespace std;

TEST_CASE("Test description", "[tag]") {
    DuckDB db(nullptr);
    Connection con(db);

    // Test code here
}
```

### Common Macros

#### Assertions
```cpp
// Require condition
REQUIRE(condition);
REQUIRE(value == 42);

// Check without failing
CHECK(condition);

// Require no exception
REQUIRE_NOTHROW(con.Query("SELECT 42"));

// Require exception
REQUIRE_THROWS(con.Query("SELECT * FROM nonexistent"));

// Require query succeeds
REQUIRE_NO_FAIL(con.Query("CREATE TABLE test(i INTEGER)"));

// Require query fails
REQUIRE_FAIL(con.Query("SELECT * FROM nonexistent"));
```

#### Result Checking
```cpp
auto result = con.Query("SELECT * FROM test");

// Check column values
REQUIRE(CHECK_COLUMN(result, 0, {1, 2, 3}));
REQUIRE(CHECK_COLUMN(result, 0, {Value()}));  // NULL value

// Check for error
REQUIRE(result->HasError());
```

### Test Organization

Use `TEST_CASE` with tags:

```cpp
TEST_CASE("Basic appender tests", "[appender]") {
    // Test code
}

TEST_CASE("Appender with transactions", "[appender][transaction]") {
    // Test code
}
```

### Marking Slow Tests

Use `[.]` tag to mark slow tests:

```cpp
TEST_CASE("Large dataset test", "[api][.]") {
    // Slow test code
}
```

Slow tests are excluded from `make unit` but included in `make allunit`.

### Connection and Database Setup

```cpp
// In-memory database
DuckDB db(nullptr);
Connection con(db);

// Database file
auto db = make_uniq<DuckDB>("test.db");
auto con = make_uniq<Connection>(*db);

// Enable query verification
con.EnableQueryVerification();

// Disable profiling (for performance tests)
con.DisableProfiling();

// Force parallel execution
con.ForceParallelism();
```

### Testing with Appender

```cpp
TEST_CASE("Appender example", "[appender]") {
    DuckDB db(nullptr);
    Connection con(db);

    REQUIRE_NO_FAIL(con.Query("CREATE TABLE test(i INTEGER)"));

    {
        Appender appender(con, "test");
        for (idx_t i = 0; i < 1000; i++) {
            appender.BeginRow();
            appender.Append<int32_t>(i);
            appender.EndRow();
        }
        appender.Close();
    }

    auto result = con.Query("SELECT COUNT(*) FROM test");
    REQUIRE(CHECK_COLUMN(result, 0, {1000}));
}
```

### Testing Type Casting

```cpp
template <class SRC, class DST>
static void TestNumericCast(vector<SRC> &working_values,
                           vector<SRC> &broken_values) {
    DST result;
    for (auto value : working_values) {
        REQUIRE_NOTHROW(Cast::Operation<SRC, DST>(value));
        REQUIRE(TryCast::Operation<SRC, DST>(value, result));
    }
    for (auto value : broken_values) {
        REQUIRE_THROWS(Cast::Operation<SRC, DST>(value));
        REQUIRE(!TryCast::Operation<SRC, DST>(value, result));
    }
}
```

### Helper Functions

```cpp
// Test directory management
string TestCreatePath(string suffix);
void TestDeleteDirectory(string path);
void TestCreateDirectory(string path);

// Database cleanup
void DeleteDatabase(string path);

// Error checking
bool NO_FAIL(QueryResult &result);
bool NO_FAIL(unique_ptr<QueryResult> result);
```

---

## Running Tests

### Quick Reference

```bash
# Fast unit tests only (~1 minute)
make unit

# All unit tests (~1 hour)
make allunit

# Run specific test by name
build/debug/test/unittest "TestName"

# Run tests with tag
build/debug/test/unittest "[tag]"

# Run one test at a time (for CI)
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest
```

### Building Tests

```bash
# Debug build (default for testing)
make debug

# Debug with specific parallelism
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make debug

# Release build for performance testing
make release
```

### Using the unittest Binary

The `unittest` binary uses Catch2, which provides powerful filtering:

```bash
# Run all tests (fast only)
build/debug/test/unittest

# Run specific test
build/debug/test/unittest "Test sum aggregate"

# Run all tests with tag
build/debug/test/unittest "[appender]"

# Run tests matching pattern
build/debug/test/unittest "*unicode*"

# Exclude slow tests (default in make unit)
build/debug/test/unittest "~[.]"

# Run ONLY slow tests
build/debug/test/unittest "[.]"

# Combine filters
build/debug/test/unittest "[api][!.]"  # API tests, exclude slow
```

### Advanced Test Execution

```bash
# Run tests one by one with timing
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --time_execution

# Continue on error (don't stop at first failure)
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --no-exit

# Fast fail (stop at first error)
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --fast-fail

# List tests without running
build/debug/test/unittest --list-tests

# Show successful tests
build/debug/test/unittest -s
```

### Running Specific Test Files

For SQL logic tests, use the path as the filter:

```bash
# Run all tests in a directory
build/debug/test/unittest "test/sql/aggregate/*"

# Run specific .test file
build/debug/test/unittest "test/sql/aggregate/aggregates/test_sum.test"
```

### Using Make Targets

```bash
# Build debug and run fast tests
make unit

# Build debug and run all tests
make allunit

# Build release and run fast tests
make unittest_release

# Clean and rebuild
make clean
make unit
```

---

## Test Organization

### Directory Structure

```
test/
├── api/                    # C++ API tests
│   ├── capi/              # C API tests
│   └── test_api.cpp       # General API tests
├── appender/              # Appender tests
├── sql/                   # SQL logic tests (most tests here)
│   ├── aggregate/         # Aggregate function tests
│   ├── types/             # Type system tests
│   ├── join/              # Join tests
│   ├── copy/              # COPY tests
│   └── ...
├── common/                # Common utility tests
├── optimizer/             # Optimizer tests
├── storage/               # Storage tests
├── extension/             # Extension tests
└── unittest.cpp           # Main test runner
```

### Naming Conventions

#### SQL Logic Tests
- Use descriptive names: `test_sum.test`, `test_unicode.test`
- Prefix with feature: `test_cast_timestamp.test`
- Slow tests: `test_name.test_slow`

#### C++ Tests
- Use descriptive test names: `TEST_CASE("Test sum aggregate", "[aggregate]")`
- File names: `test_feature.cpp`
- Slow tests: Use `[.]` tag

### Grouping Tests

SQL tests use the `group` header:
```sql
# group: [aggregates]
# group: [string]
# group: [join]
```

C++ tests use tags:
```cpp
TEST_CASE("Description", "[tag1][tag2]")
```

Common tags:
- `[api]` - API tests
- `[appender]` - Appender tests
- `[aggregate]` - Aggregate tests
- `[.]` - Slow tests

---

## Slow Tests vs Fast Tests

### What Makes a Test Slow?

A test is slow if it:
- Takes more than ~100ms to run
- Processes large datasets (>100K rows)
- Performs multiple iterations
- Uses external resources
- Performs complex operations

### Marking Tests as Slow

#### SQL Logic Tests
Add `.test_slow` extension:
```
test/sql/aggregate/aggregates/test_sum.test          # Fast
test/sql/aggregate/aggregates/test_sum_large.test_slow  # Slow
```

#### C++ Tests
Add `[.]` tag:
```cpp
TEST_CASE("Fast test", "[api]") {
    // Fast test
}

TEST_CASE("Slow test", "[api][.]") {
    // Slow test
}
```

### Examples

#### Fast Test
```sql
# name: test/sql/aggregate/aggregates/test_sum.test
# description: Test sum aggregate
# group: [aggregates]

statement ok
CREATE TABLE integers(i INTEGER);

statement ok
INSERT INTO integers SELECT * FROM range(0, 1000, 1);

query I
SELECT SUM(i) FROM integers;
----
499500
```

#### Slow Test
```sql
# name: test/sql/aggregate/aggregates/first_memory_usage.test_slow
# description: Test memory usage with large aggregations
# group: [aggregates]

load __TEST_DIR__/first_memory_usage.db

statement ok
set threads=1;

statement ok
set memory_limit='500mb';

statement ok
select distinct on (a) b
from (select s a, md5(s::text) b from generate_series(1,5_000_000) as g(s))
limit 10;
```

---

## Writing Effective Tests

### Coverage Principles

1. **Normal Cases**: Test typical usage
2. **Edge Cases**: Test boundaries, empty inputs, NULL values
3. **Error Cases**: Test expected failures
4. **Type Variations**: Test with different data types
5. **Scale Variations**: Test small and large datasets

### Example: Comprehensive Test

```sql
# name: test/sql/aggregate/aggregates/test_sum_comprehensive.test
# description: Comprehensive sum aggregate tests
# group: [aggregates]

statement ok
CREATE TABLE integers(i INTEGER);

# Test with positive numbers
statement ok
INSERT INTO integers SELECT * FROM range(0, 1000, 1);

query I
SELECT SUM(i) FROM integers;
----
499500

# Test with negative numbers
statement ok
INSERT INTO integers SELECT * FROM range(0, -1000, -1);

query I
SELECT SUM(i) FROM integers;
----
0

# Test with NULL values
statement ok
INSERT INTO integers VALUES (NULL), (NULL);

query I
SELECT SUM(i) FROM integers;
----
0

# Test empty result
query I
SELECT SUM(i) FROM integers WHERE i > 10000;
----
NULL

# Test with constant
query I
SELECT SUM(1) FROM integers;
----
2002

# Test overflow handling
statement ok
CREATE TABLE bigints(b BIGINT);

statement ok
INSERT INTO bigints SELECT * FROM range(4611686018427387904, 4611686018427388904, 1);

# This should succeed (result is hugeint)
query I
SELECT SUM(b) FROM bigints
----
4611686018427388403500

# This should fail (overflow bigint)
statement error
SELECT SUM(b)::BIGINT FROM bigints
----
<REGEX>:.*Conversion Error.*
```

### Testing Best Practices

#### 1. Test Independence
Each test should be self-contained:
```sql
# Good: Create own tables
statement ok
CREATE TABLE test(i INTEGER);

statement ok
INSERT INTO test VALUES (1), (2), (3);

query I
SELECT SUM(i) FROM test;
----
6
```

#### 2. Clean Test Names
```sql
# Good names
test_sum_with_nulls.test
test_unicode_substring.test
test_join_inner_basic.test

# Avoid
test1.test
test_temp.test
```

#### 3. Test One Thing
```sql
# Good: Focused test
# name: test/sql/aggregate/aggregates/test_sum_nulls.test
# description: Test SUM with NULL values
# group: [aggregates]

# Bad: Testing too many things
# name: test/sql/aggregate/aggregates/test_all_aggregates.test
```

#### 4. Use Meaningful Data
```sql
# Good: Clear values
statement ok
CREATE TABLE employees(salary INTEGER);

statement ok
INSERT INTO employees VALUES (50000), (60000), (70000);

# Bad: Arbitrary values
statement ok
CREATE TABLE t(x INTEGER);

statement ok
INSERT INTO t VALUES (1), (2), (3);
```

---

## Testing Different Data Types

### Numeric Types

```sql
# Test all integer types
foreach type tinyint smallint integer bigint hugeint utinyint usmallint uinteger ubigint uhugeint

statement ok
CREATE TABLE test_${type}(i ${type});

statement ok
INSERT INTO test_${type} VALUES (1), (2), (3);

query I
SELECT SUM(i) FROM test_${type};
----
6

statement ok
DROP TABLE test_${type};

endloop
```

### String Types

```sql
# name: test/sql/types/string/test_unicode.test
# description: Test unicode strings
# group: [string]

statement ok
CREATE TABLE emojis(id INTEGER, s VARCHAR);

statement ok
INSERT INTO emojis VALUES (1, '🦆'), (2, '🦆🍞🦆')

# Test substring with unicode
query TT
SELECT substring(s, 1, 1), substring(s, 2, 1) FROM emojis ORDER BY id
----
🦆	(empty)
🦆	🍞

# Test length with unicode
query I
SELECT length(s) FROM emojis ORDER BY id
----
1
3
```

### Date/Time Types

```sql
statement ok
CREATE TABLE dates(d DATE, t TIME, ts TIMESTAMP);

statement ok
INSERT INTO dates VALUES
    ('2024-01-01', '12:30:00', '2024-01-01 12:30:00'),
    ('2024-12-31', '23:59:59', '2024-12-31 23:59:59');

query TTT
SELECT * FROM dates ORDER BY d;
----
2024-01-01	12:30:00	2024-01-01 12:30:00
2024-12-31	23:59:59	2024-12-31 23:59:59
```

### Nested Types

```sql
# Lists
statement ok
CREATE TABLE lists(l INTEGER[]);

statement ok
INSERT INTO lists VALUES ([1, 2, 3]), ([4, 5, 6]), (NULL);

query I
SELECT l FROM lists;
----
[1, 2, 3]
[4, 5, 6]
NULL

# Structs
statement ok
CREATE TABLE structs(s STRUCT(a INTEGER, b VARCHAR));

statement ok
INSERT INTO structs VALUES
    ({'a': 1, 'b': 'hello'}),
    ({'a': 2, 'b': 'world'});

query I
SELECT s.a FROM structs ORDER BY s.a;
----
1
2

# Maps
statement ok
CREATE TABLE maps(m MAP(VARCHAR, INTEGER));

statement ok
INSERT INTO maps VALUES
    (MAP(['a', 'b'], [1, 2])),
    (MAP(['x', 'y'], [10, 20]));

query I
SELECT m FROM maps;
----
{a=1, b=2}
{x=10, y=20}
```

### NULL Handling

```sql
# Test NULL in all contexts
statement ok
CREATE TABLE nulls(i INTEGER);

statement ok
INSERT INTO nulls VALUES (1), (NULL), (3);

# NULL in WHERE
query I
SELECT * FROM nulls WHERE i IS NULL;
----
NULL

# NULL in aggregates
query I
SELECT COUNT(*), COUNT(i), SUM(i) FROM nulls;
----
3	2	4

# NULL comparison
query I
SELECT * FROM nulls WHERE i = NULL;
----

query I
SELECT * FROM nulls WHERE i IS NULL;
----
NULL
```

---

## Testing Error Cases

### Expected Errors

Always test error conditions:

```sql
# Type errors
statement error
SELECT 'abc'::INTEGER;
----
<REGEX>:.*Conversion Error.*

# Division by zero
statement error
SELECT 1/0;
----
<REGEX>:.*Division by zero.*

# Table not found
statement error
SELECT * FROM nonexistent_table;
----
<REGEX>:.*Table.*not found.*

# Binder errors
statement error
SELECT undefined_column FROM test;
----
<REGEX>:.*Binder Error.*

# Constraint violations
statement ok
CREATE TABLE test(i INTEGER PRIMARY KEY);

statement ok
INSERT INTO test VALUES (1);

statement error
INSERT INTO test VALUES (1);
----
<REGEX>:.*PRIMARY KEY.*conflict.*
```

### Error Message Patterns

Use regex for flexible matching:

```sql
# Exact match
statement error
SELECT 1/0;
----
Division by zero

# Regex match
statement error
SELECT 1/0;
----
<REGEX>:.*Division.*zero.*

# Case insensitive
statement error
SELECT 1/0;
----
<REGEX>:(?i)division.*zero
```

### C++ Error Testing

```cpp
TEST_CASE("Test error handling", "[error]") {
    DuckDB db(nullptr);
    Connection con(db);

    // Test exception
    REQUIRE_THROWS(con.Query("SELECT * FROM nonexistent"));

    // Test error result
    auto result = con.Query("SELECT * FROM nonexistent");
    REQUIRE(result->HasError());
    REQUIRE(result->GetError().find("Table") != string::npos);

    // Test REQUIRE_FAIL macro
    REQUIRE_FAIL(con.Query("SELECT 'abc'::INTEGER"));
}
```

---

## Debugging Failing Tests

### Basic Debugging Steps

1. **Run the failing test individually**:
```bash
build/debug/test/unittest "test/sql/aggregate/aggregates/test_sum.test"
```

2. **Enable verbose output**:
```bash
build/debug/test/unittest "TestName" -s
```

3. **Check the error message**: The test output shows expected vs actual results

4. **Run in debugger**:
```bash
lldb build/debug/test/unittest
> run "TestName"
> bt  # backtrace on crash
```

### Common Issues

#### Issue: Test Passes Locally, Fails in CI
```bash
# Check if it's a slow test issue
build/debug/test/unittest "[.]"

# Check if it's platform-specific
require notwindows
require notmingw
```

#### Issue: Flaky Test (Non-deterministic)
```sql
# Disable verification for non-deterministic tests
statement ok
PRAGMA disable_verification

# Use ORDER BY for deterministic results
query I
SELECT * FROM test ORDER BY id;
```

#### Issue: Test Times Out
```bash
# Mark as slow test
mv test.test test.test_slow

# Or in C++
TEST_CASE("Slow test", "[tag][.]") { ... }
```

### Debugging SQL Logic Tests

Add temporary output:
```sql
# Debug: Print intermediate results
query I
SELECT COUNT(*) FROM test;
----
<expected_count>

query I
SELECT * FROM test;
----
<all_rows>
```

### Debugging C++ Tests

```cpp
TEST_CASE("Debug test", "[debug]") {
    DuckDB db(nullptr);
    Connection con(db);

    // Print intermediate results
    auto result = con.Query("SELECT * FROM test");
    std::cout << result->ToString() << std::endl;

    // Break in debugger
    // Add breakpoint here
    REQUIRE(condition);
}
```

### Using Sanitizers

Debug builds enable sanitizers by default:
```bash
# Build with sanitizers (default)
make debug

# Build without sanitizers
DISABLE_SANITIZER=1 make debug

# Run with AddressSanitizer
ASAN_OPTIONS=detect_leaks=1 build/debug/test/unittest "TestName"
```

---

## CI/CD and Automated Testing

### Continuous Integration

DuckDB runs tests automatically on:
- Every pull request
- Every commit to main
- Nightly builds

### CI Test Workflow

1. **Fast tests** run first (~5 minutes)
2. **All tests** run in parallel (~30 minutes)
3. **Platform-specific** tests (Windows, macOS, Linux)
4. **Extension tests**
5. **Fuzzer tests**

### Pre-commit Testing

Before committing:
```bash
# 1. Format code
make format-fix

# 2. Run fast tests
make unit

# 3. Build release
make release

# 4. Run all tests (optional but recommended)
make allunit
```

### Test Requirements for PRs

All PRs must:
1. Pass all unit tests
2. Be properly formatted (`make format-fix`)
3. Include tests for new features
4. Include tests for bug fixes
5. Not introduce flaky tests

### Running Tests Like CI

```bash
# Run tests one by one (like CI)
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --time_execution

# Check for timing issues
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --time_execution | grep "SLOW"
```

---

## Test-Driven Development Workflow

### TDD Cycle

1. **Write a failing test**
2. **Implement the feature**
3. **Make the test pass**
4. **Refactor**
5. **Repeat**

### Example: Adding a New Function

#### Step 1: Write Failing Test
```sql
# name: test/sql/function/string/test_reverse.test
# description: Test REVERSE function
# group: [string]

# This will fail initially
query T
SELECT REVERSE('hello');
----
olleh
```

```bash
# Verify it fails
build/debug/test/unittest "test/sql/function/string/test_reverse.test"
# Output: Error: Function REVERSE not found
```

#### Step 2: Implement Feature
```cpp
// In src/function/scalar/string/reverse.cpp
// Implement the REVERSE function
```

#### Step 3: Make Test Pass
```bash
# Build and test
make debug
build/debug/test/unittest "test/sql/function/string/test_reverse.test"
# Output: All tests passed
```

#### Step 4: Add More Tests
```sql
# Test edge cases
query T
SELECT REVERSE('');
----
(empty)

query T
SELECT REVERSE(NULL);
----
NULL

# Test unicode
query T
SELECT REVERSE('🦆🍞');
----
🍞🦆

# Test error case
statement error
SELECT REVERSE(123);
----
<REGEX>:.*No function matches.*
```

#### Step 5: Run All Tests
```bash
make unit
```

### Bug Fix Workflow

1. **Create regression test that reproduces the bug**:
```sql
# name: test/issues/general/test_issue_12345.test
# description: Test fix for issue #12345
# group: [general]

statement ok
CREATE TABLE test(i INTEGER);

# This query crashes in version X.Y.Z
query I
SELECT * FROM test WHERE i > NULL;
----
```

2. **Verify test fails**:
```bash
build/debug/test/unittest "test/issues/general/test_issue_12345.test"
```

3. **Fix the bug**

4. **Verify test passes**:
```bash
build/debug/test/unittest "test/issues/general/test_issue_12345.test"
```

5. **Run all tests**:
```bash
make unit
```

### Development Best Practices

1. **Write tests first** (when possible)
2. **Keep tests fast** (use `.test_slow` sparingly)
3. **Test incrementally** (run tests frequently)
4. **Use descriptive test names**
5. **Test edge cases early**
6. **Clean up test databases** (use in-memory when possible)

---

## Quick Reference

### Common Commands
```bash
# Build and run fast tests
make unit

# Run all tests
make allunit

# Run specific test
build/debug/test/unittest "TestName"

# Run tests with tag
build/debug/test/unittest "[tag]"

# Format code
make format-fix

# Run tests one by one
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest
```

### Test File Template (SQL)
```sql
# name: test/path/to/test_feature.test
# description: Test feature description
# group: [group_name]

statement ok
CREATE TABLE test(i INTEGER);

statement ok
INSERT INTO test VALUES (1), (2), (3);

query I
SELECT SUM(i) FROM test;
----
6
```

### Test File Template (C++)
```cpp
#include "catch.hpp"
#include "test_helpers.hpp"

using namespace duckdb;

TEST_CASE("Test description", "[tag]") {
    DuckDB db(nullptr);
    Connection con(db);

    REQUIRE_NO_FAIL(con.Query("CREATE TABLE test(i INTEGER)"));

    auto result = con.Query("SELECT * FROM test");
    REQUIRE(CHECK_COLUMN(result, 0, {}));
}
```

---

## Additional Resources

- **Test Directory**: `test/`
- **Test README**: `test/README.md`
- **CLAUDE.md**: `CLAUDE.md`
- **Catch2 Documentation**: https://github.com/catchorg/Catch2
- **DuckDB Documentation**: https://duckdb.org/docs/

---

## Summary

Testing in DuckDB is straightforward:

1. **Prefer SQL logic tests** (`.test` files) for most testing
2. **Use C++ tests** only when testing internal APIs
3. **Mark slow tests** appropriately (`.test_slow` or `[.]`)
4. **Test comprehensively**: normal cases, edge cases, errors
5. **Run tests frequently** during development
6. **Follow TDD** when adding features
7. **Always include tests** with bug fixes

The key to effective testing is writing simple, focused tests that are easy to understand and maintain.
