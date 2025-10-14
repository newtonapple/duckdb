# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

DuckDB is a high-performance analytical database system designed to be fast, reliable, portable, and easy to use. It is an embeddable SQL OLAP database management system with a rich SQL dialect supporting complex queries, window functions, nested types (arrays, structs, maps), and multiple extensions.

## Build System

DuckDB uses CMake with a Makefile wrapper. Python 3 and a C++11 compliant compiler are required.

### Common Build Commands

- `make` or `make release` - Build optimized release version
- `make debug` - Build debug version with sanitizers enabled
- `make clean` - Remove build directory
- `GEN=ninja make` - Use Ninja for parallel builds (faster)
- `CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make` - Limit parallel build processes

### Build with Benchmarks/Extensions

- `BUILD_BENCHMARK=1 BUILD_TPCH=1 make` - Build with benchmarks
- `BUILD_BENCHMARK=1 make` then `./build/release/benchmark/benchmark_runner` - Run benchmarks
- `DUCKDB_EXTENSIONS="extension1;extension2" make` - Build with specific extensions

### Additional Build Targets

- `make reldebug` - Release with debug info (RelWithDebInfo)
- `make relassert` - Release with assertions enabled
- `make unittest_release` - Run unit tests on release build

## Testing

DuckDB has two test frameworks: sqllogictest (`.test` files) and C++ tests. **Strongly prefer sqllogictest** for new tests.

### Running Tests

- `make unit` or `make unittest` - Run fast unit tests (~1 minute)
- `make allunit` - Run all unit tests (~1 hour)
- `build/debug/test/unittest` - Run all fast tests directly
- `build/debug/test/unittest "[tag]"` - Run tests with specific tag (e.g., `"[arrow]"`)
- `build/debug/test/unittest "TestName"` - Run specific test
- `python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --time_execution` - Run tests one-by-one for CI

### Test Organization

- **Slow tests**: Name test files `.test_slow` or add `[.]` tag in C++ tests
- **Test files**: Located in `test/sql/` (sqllogictest) or `test/` (C++)
- Tests should cover different types (numerics, strings, nested types) and error cases

## Code Formatting

- `make format-fix` - Format all code using clang-format and black
- `make format-check` - Check formatting without fixing
- `make format-main` - Format changes compared to main branch
- `make format-head` - Format changes in HEAD commit

**Important**: Always run `make format-fix` before submitting PRs. Use clang-format version 11.0.1:
```bash
python3 -m pip install clang-format==11.0.1
```

## Architecture

DuckDB follows a layered database architecture:

### Parser (`src/parser/`)
Entry point for queries. Uses PostgreSQL's libpg_query parser and transforms tokens into a custom parse tree with `SQLStatements`, `Expressions`, and `TableRefs`.

### Planner (`src/planner/`)
Converts parsed tokens into a Logical Query Plan represented as a tree of `LogicalOperator` nodes. The `Binder` resolves symbols using the `Catalog`.

### Optimizer (`src/optimizer/`)
Transforms Logical Query Plans into faster equivalent plans. Performs both rule-based and cost-based optimizations including predicate pushdown, expression rewriting, and join ordering.

### Execution (`src/execution/`)
Converts Logical Query Plans to Physical Query Plans (`PhysicalOperators`) and executes them using a push-based (vectorized) execution model.

### Catalog (`src/catalog/`)
Manages database metadata (tables, schemas, functions). Used by the Binder during planning to resolve table/column references.

### Storage (`src/storage/`)
Manages physical data both in-memory and on-disk. Handles base table scans and data modification operations (INSERT, UPDATE, DELETE).

### Transaction (`src/transaction/`)
Manages concurrent transactions and handles COMMIT/ROLLBACK operations.

### Other Components
- `src/common/` - Shared utilities, data structures, types
- `src/function/` - Built-in functions (scalar, aggregate, table, window)
- `src/parallel/` - Parallel execution primitives
- `src/main/` - Database instance, connection management, client APIs

## Extensions

Extensions are in the `extension/` directory. Core extensions include:
- `parquet` - Parquet file support
- `json` - JSON functions
- `icu` - International Components for Unicode
- `tpch`/`tpcds` - Benchmark data generators
- `core_functions` - Core function implementations

## C++ Guidelines

### Type System
- Use `[u]int(8|16|32|64)_t` instead of `int`, `long`, etc.
- Use `idx_t` instead of `size_t` for offsets/indices/counts
- Prefer references over pointers for function arguments
- Use `const` references for non-trivial objects

### Memory Management
- **Do not use** `malloc`, `new`, or `delete`
- Prefer `unique_ptr` over `shared_ptr`
- Use smart pointers for all heap allocations

### Naming Conventions
- **Files**: lowercase_with_underscores.cpp
- **Types** (classes/structs/enums): CamelCase (UpperCamelCase)
- **Variables**: lowercase_with_underscores
- **Functions**: CamelCase (UpperCamelCase)
- Avoid single-letter variables in nested loops (use descriptive names like `column_idx`)

### Code Style
- Use **tabs for indentation, spaces for alignment**
- Maximum line length: 120 characters
- Do **not** import namespaces (no `using namespace std`)
- All core functions should be in `duckdb` namespace
- Use C++11 range-based for loops: `for (const auto& item : items) {...}`
- Always use braces for if statements and loops
- Use `override` or `final` for virtual methods (not `virtual`)

### Error Handling
- **Exceptions**: Only for query-terminating errors (parser errors, table not found)
- **Return values**: For expected errors that don't break execution flow
- **Assertions**: Use `D_ASSERT` for programmer errors (never triggered by user input)
- Add comments explaining what assert failures mean

### Class Layout
```cpp
class MyClass {
public:
    MyClass();
    int my_public_variable;

public:
    void MyFunction();

private:
    void MyPrivateFunction();

private:
    int my_private_variable;
};
```

## Scripts

- `scripts/format.py` - Code formatting
- `scripts/amalgamation.py` - Create single-file amalgamation build
- `scripts/generate_*.py` - Code generation for C API, functions, settings, etc.
- `scripts/run_tests_one_by_one.py` - Sequential test execution
- `scripts/coverage_check.sh` - Code coverage analysis

## Development Workflow

1. **Before coding**: Read relevant source files to understand architecture
2. **Build**: Use `make debug` for development
3. **Test frequently**: Run `make unit` to verify changes
4. **Format code**: Run `make format-fix` before committing
5. **Test thoroughly**: Run `make allunit` before submitting PR
6. **Check coverage**: Ensure tests cover new code paths

## Pull Request Requirements

- All unit tests must pass (`make allunit`)
- Code must be formatted (`make format-fix`)
- Add tests for new features/bug fixes (prefer sqllogictest)
- PRs must not be in "Draft" state
- Keep PRs small and focused
- Do not commit commented-out code

## Important Notes

- **No generative AI PRs**: Do not submit AI-generated pull requests
- **Unity builds**: Default enabled; disable with `DISABLE_UNITY=1 make` if needed
- **Sanitizers**: Enabled by default in debug builds; disable with `DISABLE_SANITIZER=1 make`
- **Main branch**: Default branch for PRs is `main`
