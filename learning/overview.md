# DuckDB Project Overview

A comprehensive guide for new developers getting started with DuckDB development.

## What is DuckDB?

DuckDB is a high-performance analytical database system (OLAP) designed to be fast, reliable, portable, and easy to use. Think of it as "SQLite for analytics" - it's an embeddable database that excels at complex analytical queries with a rich SQL dialect supporting window functions, nested types (arrays, structs, maps), and multiple extensions.

## Development Dependencies

Before you start, you'll need:
- **Python 3** (required for build scripts)
- **C++11 compliant compiler** (GCC, Clang, or MSVC)
- **CMake** (build system)
- **Git** (version control)
- **Optional but recommended:**
  - Ninja build system (faster parallel builds): `pip install ninja`
  - clang-format 11.0.1 (code formatting): `python3 -m pip install clang-format==11.0.1`

## Build Commands

DuckDB uses CMake with a Makefile wrapper. Here are the essential commands:

### Basic Builds
```bash
make                    # Build optimized release version
make release           # Same as above
make debug             # Build debug version (with sanitizers)
make reldebug          # Release with debug info
make relassert         # Release with assertions enabled
make clean             # Remove build directory
```

### Faster Builds
```bash
GEN=ninja make                              # Use Ninja for parallel builds
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make # Limit parallel processes
```

### Specialized Builds
```bash
BUILD_BENCHMARK=1 BUILD_TPCH=1 make         # Build with benchmarks
DUCKDB_EXTENSIONS="parquet;json" make       # Build with specific extensions
DISABLE_UNITY=1 make                        # Disable unity builds (slower but better for debugging)
DISABLE_SANITIZER=1 make debug              # Debug without sanitizers
```

## Testing

DuckDB has two test frameworks:

### 1. C++ Unit Tests (Catch2)
```bash
make unit                # Run fast unit tests (~1 minute)
make unittest            # Same as above
make allunit            # Run ALL unit tests (~1 hour)
make unittest_release   # Run tests on release build

# Run tests directly for more control:
build/debug/test/unittest                    # All fast tests
build/debug/test/unittest "[arrow]"          # Tests with specific tag
build/debug/test/unittest "TestName"         # Specific test by name
```

### 2. SQL Logic Tests (Preferred for new tests!)
Located in `test/sql/` directory with `.test` extensions. These are SQL-based tests that are easier to write and maintain.

### Test Organization
- **Fast tests**: Regular test files
- **Slow tests**: Name files `.test_slow` or add `[.]` tag in C++ tests
- **Running tests individually**: `python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --time_execution`

## Architecture Overview

DuckDB follows a classic layered database architecture. Here's the query execution flow:

### 1. Parser (`src/parser/`)
- **Entry point** for SQL queries
- Uses PostgreSQL's `libpg_query` parser
- Transforms SQL text into a parse tree with:
  - `SQLStatements` (SELECT, INSERT, etc.)
  - `Expressions` (columns, functions, operators)
  - `TableRefs` (table references, joins)

### 2. Planner (`src/planner/`)
- Converts parse tree → **Logical Query Plan**
- Represented as a tree of `LogicalOperator` nodes
- The `Binder` resolves symbols (tables, columns) using the Catalog
- Example operators: LogicalGet, LogicalFilter, LogicalJoin

### 3. Optimizer (`src/optimizer/`)
- Transforms logical plans into faster equivalent plans
- **Rule-based optimizations**: Predicate pushdown, expression rewriting
- **Cost-based optimizations**: Join ordering, index selection
- Multiple optimization passes for different aspects

### 4. Execution (`src/execution/`)
- Converts Logical Plan → **Physical Query Plan**
- Uses `PhysicalOperator` nodes
- **Push-based vectorized execution** (processes data in batches/vectors)
- Highly parallel and cache-efficient

### 5. Supporting Components

- **Catalog** (`src/catalog/`): Database metadata (tables, schemas, functions)
- **Storage** (`src/storage/`): Physical data management (in-memory & on-disk)
- **Transaction** (`src/transaction/`): MVCC, ACID compliance
- **Common** (`src/common/`): Shared utilities, types, data structures
- **Function** (`src/function/`): Built-in functions (scalar, aggregate, table, window)
- **Parallel** (`src/parallel/`): Threading and parallelization primitives
- **Main** (`src/main/`): Database instance, connection management, client APIs

## Code Structure

Here's the directory layout:

```
duckdb/
├── src/                    # Core source code
│   ├── catalog/           # Schema management
│   ├── common/            # Utilities, types, vectors
│   ├── execution/         # Query execution engine
│   ├── function/          # Built-in SQL functions
│   ├── main/              # Database & connection APIs
│   ├── optimizer/         # Query optimization
│   ├── parser/            # SQL parsing
│   ├── planner/           # Query planning & binding
│   ├── storage/           # Data storage layer
│   ├── transaction/       # Transaction management
│   └── parallel/          # Parallelization
│
├── extension/             # Extensions (Parquet, JSON, ICU, etc.)
├── test/                  # C++ unit tests
│   └── sql/              # SQL logic tests (.test files)
├── tools/                 # Tools (shell, JDBC, ODBC)
├── scripts/               # Build & utility scripts
├── benchmark/             # Benchmark suite
└── third_party/           # External dependencies
```

## C++ Development Guidelines

### Type System
```cpp
// Use fixed-width types
int32_t, int64_t, uint8_t     // NOT: int, long
idx_t                          // NOT: size_t (for indices/counts)

// Prefer references over pointers
void MyFunction(const Vector &input, Vector &output);
```

### Naming Conventions
```cpp
// Files
my_source_file.cpp

// Classes/Types
class MyAwesomeClass { };
struct DataChunk { };
enum class PhysicalType { };

// Variables
int my_variable;
idx_t row_count;

// Functions
void MyFunction();
void ProcessData();
```

### Memory Management
```cpp
// NEVER use malloc, new, delete directly
// BAD:
int *data = new int[100];

// GOOD:
auto data = make_uniq<int[]>(100);           // unique_ptr
vector<int> data(100);                       // STL container
```

### Code Style
- **Indentation**: Tabs for indentation, spaces for alignment
- **Max line length**: 120 characters
- **NO namespace imports**: Never use `using namespace std;`
- **Always use braces** for if/loops
- **Use range-based for loops**: `for (const auto &item : items) { }`

### Error Handling
```cpp
// Exceptions: For query-terminating errors
throw InvalidInputException("Table 'foo' not found");

// Assertions: For programmer errors (never user-triggered)
D_ASSERT(index < vector_size); // "Index out of bounds"

// Return values: For expected errors
bool TryParse(const string &input, int64_t &result);
```

## Code Formatting

**ALWAYS format before committing!**

```bash
make format-fix      # Format all code
make format-check    # Check without fixing
make format-main     # Format changes vs main branch
make format-head     # Format HEAD commit changes
```

## Extensions

Located in `extension/` directory. Core extensions include:
- `parquet` - Parquet file support
- `json` - JSON functions
- `icu` - International Components for Unicode
- `tpch`/`tpcds` - Benchmark data generators
- `core_functions` - Core function implementations

## Development Workflow

1. **Start with debug build**: `make debug`
2. **Make changes** to source files
3. **Test frequently**: `make unit` (fast feedback)
4. **Format code**: `make format-fix`
5. **Test thoroughly**: `make allunit` (before PR)
6. **Check git status**: Ensure only intended files changed

## Useful Scripts

```bash
scripts/format.py                          # Code formatting
scripts/run_tests_one_by_one.py           # Sequential test execution
scripts/amalgamation.py                    # Single-file build
scripts/coverage_check.sh                  # Code coverage
```

## Tips for Getting Started

1. **Start small**: Read existing code in the area you're working on
2. **Use debug builds**: Easier to debug with sanitizers enabled
3. **Write tests first**: Prefer `.test` files in `test/sql/`
4. **Ask questions**: The codebase is large, it's okay to ask for guidance
5. **Read CLAUDE.md**: The project guide at the root has more details

## Common Pitfalls to Avoid

- Don't commit without formatting (`make format-fix`)
- Don't use raw pointers for ownership
- Don't import namespaces (`using namespace`)
- Don't use `int` or `size_t` (use `int32_t`, `idx_t`)
- Don't skip tests before submitting
- Avoid commented-out code in commits

## Pull Request Requirements

- All unit tests must pass (`make allunit`)
- Code must be formatted (`make format-fix`)
- Add tests for new features/bug fixes (prefer sqllogictest)
- PRs must not be in "Draft" state
- Keep PRs small and focused
- Do not commit commented-out code

## Next Steps

To get hands-on experience:

1. **Build the project**: `make debug`
2. **Run tests**: `make unit`
3. **Explore the code**: Start with `src/main/` or a simple function in `src/function/`
4. **Try making a small change**: Add a simple test or fix a small issue
5. **Read existing tests**: Look at `.test` files to understand the testing patterns

## Additional Resources

- **CLAUDE.md**: Project guide at repository root
- **DuckDB Documentation**: https://duckdb.org/docs/
- **GitHub Repository**: https://github.com/duckdb/duckdb
- **Community Discord**: Join for questions and discussions
