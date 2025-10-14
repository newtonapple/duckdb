# DuckDB Extension Development Guide

This guide provides comprehensive information for developers building DuckDB extensions. It covers architecture, setup, development workflow, testing, and distribution.

## Table of Contents

1. [What are DuckDB Extensions](#what-are-duckdb-extensions)
2. [In-tree vs Out-of-tree Extensions](#in-tree-vs-out-of-tree-extensions)
3. [Using the Extension Template](#using-the-extension-template)
4. [Setting Up a New Extension](#setting-up-a-new-extension)
5. [Extension Structure and Organization](#extension-structure-and-organization)
6. [Building Extensions](#building-extensions)
7. [Adding Functions to Extensions](#adding-functions-to-extensions)
8. [Testing Extensions](#testing-extensions)
9. [Debugging Extensions](#debugging-extensions)
10. [Distribution Options](#distribution-options)
11. [Versioning and DuckDB Compatibility](#versioning-and-duckdb-compatibility)
12. [Complete Walkthrough: Creating a Simple Extension](#complete-walkthrough-creating-a-simple-extension)

---

## What are DuckDB Extensions

DuckDB extensions are libraries containing additional DuckDB functionality separate from the main codebase. Extensions provide a way to add features that:

- Should not live in the core DuckDB codebase (to keep core small and focused)
- Have licensing requirements incompatible with core DuckDB
- Are experimental or domain-specific
- Depend on external libraries
- Are maintained by external developers

Extensions can be used in two ways:

1. **Statically linked**: Built directly into DuckDB executables (CLI, unittest binary, etc.)
2. **Dynamically loaded**: Loaded at runtime using `INSTALL` and `LOAD` commands

### Common Extension Use Cases

- **Data format support**: Parquet, JSON, Excel, Avro
- **External database connectors**: PostgreSQL, MySQL, SQLite scanners
- **Specialized functions**: ICU (internationalization), full-text search
- **Cloud storage**: AWS S3, Azure Blob Storage, HTTP file system
- **Benchmarking**: TPC-H, TPC-DS data generators

---

## In-tree vs Out-of-tree Extensions

### In-tree Extensions

**Location**: `extension/` (in DuckDB repository)

In-tree extensions live in the main DuckDB repository. They are considered fundamental to DuckDB or tie in so deeply that DuckDB changes regularly break them.

**Examples**:
- `parquet` - Parquet file format support
- `json` - JSON functions and types
- `icu` - International Components for Unicode
- `tpch`/`tpcds` - Benchmark data generators
- `core_functions` - Core function implementations
- `autocomplete` - Shell autocomplete functionality

**Characteristics**:
- Maintained by the DuckDB team
- Tested with every DuckDB commit
- Changes coordinated with core DuckDB changes
- Built using the same CI as DuckDB core

The DuckDB project aims to keep in-tree extensions to a minimum and moves extensions out-of-tree where possible.

### Out-of-tree Extensions (OOTEs)

Out-of-tree extensions live in separate repositories outside the main DuckDB repository. There are two main categories:

#### 1. DuckDB Managed OOTEs

These are distributed through the main DuckDB CI and signed using DuckDB's signing key.

**Examples**:
- `sqlite_scanner` - Read SQLite databases
- `postgres_scanner` - Query PostgreSQL databases
- `spatial` - Geospatial data support
- `aws` - AWS services integration
- `azure` - Azure services integration

**Configuration**: Listed in `.github/config/out_of_tree_extensions.cmake`

**Characteristics**:
- Maintained by the DuckDB team
- Distributed automatically with every DuckDB release
- Signed with DuckDB's signing key
- Built and tested through DuckDB CI

#### 2. External OOTEs

These are maintained by external developers in their own repositories.

**Characteristics**:
- Independent CI/CD in their own repositories
- Maintainer responsible for testing and distribution
- May or may not be signed (depending on maintainer)
- Can be distributed via:
  - Community extensions repository (recommended)
  - Custom extension repositories
  - GitHub artifacts (unsigned)

---

## Using the Extension Template

The extension template repository provides a complete starting point for building DuckDB extensions.

**Repository**: https://github.com/duckdb/extension-template

### Key Features

1. **Complete build system** using CMake and Make wrappers
2. **VCPKG integration** for dependency management
3. **CI/CD workflows** for building and distributing extensions
4. **Example implementation** showing scalar function registration
5. **Test framework** set up with SQLLogicTest examples
6. **CLion configuration** guide for IDE development

### Template Structure

```
extension-template/
├── duckdb/                    # DuckDB submodule (specific version)
├── extension-ci-tools/        # CI/CD tools submodule
├── src/
│   ├── include/
│   │   └── quack_extension.hpp    # Extension header
│   └── quack_extension.cpp        # Extension implementation
├── test/
│   └── sql/
│       └── quack.test             # SQL tests
├── scripts/
│   ├── bootstrap-template.py      # Rename template script
│   └── extension-upload.sh        # Upload helper
├── .github/workflows/
│   └── MainDistributionPipeline.yml  # CI/CD workflow
├── CMakeLists.txt             # Build configuration
├── extension_config.cmake     # Extension loading config
├── vcpkg.json                 # Dependency manifest
├── Makefile                   # Build wrapper
└── README.md                  # Documentation
```

---

## Setting Up a New Extension

### Step 1: Create Repository from Template

1. Go to https://github.com/duckdb/extension-template
2. Click "Use this template" to create your repository
3. Choose a repository name (e.g., `duckdb-myextension`)

### Step 2: Clone with Submodules

```bash
git clone --recurse-submodules https://github.com/<you>/<your-extension-repo>.git
cd <your-extension-repo>
```

The `--recurse-submodules` flag is **critical** - it ensures the DuckDB source code and CI tools are cloned.

### Step 3: Set Up VCPKG (Optional)

VCPKG is only required if your extension has dependencies. The template includes OpenSSL as an example.

```bash
# In a directory OUTSIDE your extension repository
cd <your-working-dir-not-the-plugin-repo>
git clone https://github.com/Microsoft/vcpkg.git
./vcpkg/bootstrap-vcpkg.sh -disableMetrics
export VCPKG_TOOLCHAIN_PATH=`pwd`/vcpkg/scripts/buildsystems/vcpkg.cmake
```

To skip VCPKG:
- Remove dependencies from `vcpkg.json`
- Remove `find_package()` and `target_link_libraries()` calls in `CMakeLists.txt`

### Step 4: Rename the Extension

```bash
python3 ./scripts/bootstrap-template.py <your_extension_name>
```

Use snake_case for the extension name (e.g., `my_extension`, `json_parser`, `spatial_utils`).

This script:
- Renames files and directories
- Updates function names and class names
- Modifies CMake configuration
- Updates test files

After running, you can delete the bootstrap script.

### Step 5: Verify the Build

```bash
make
```

For faster builds (recommended):

```bash
# Install ccache and ninja first
# macOS: brew install ccache ninja
# Ubuntu: apt-get install ccache ninja-build

GEN=ninja make
```

This creates:
- `./build/release/duckdb` - DuckDB shell with extension pre-loaded
- `./build/release/test/unittest` - Test runner
- `./build/release/extension/<name>/<name>.duckdb_extension` - Loadable binary

### Step 6: Test the Extension

```bash
make test
```

Or run the shell:

```bash
./build/release/duckdb
```

---

## Extension Structure and Organization

### Required Components

Every extension must implement three core components:

#### 1. Extension Class

Your extension class inherits from `duckdb::Extension`:

```cpp
// src/include/myext_extension.hpp
#pragma once

#include "duckdb.hpp"

namespace duckdb {

class MyExtExtension : public Extension {
public:
    void Load(ExtensionLoader &loader) override;
    std::string Name() override;
    std::string Version() const override;
};

} // namespace duckdb
```

#### 2. Load Function

The `Load()` function registers all extension functionality:

```cpp
// src/myext_extension.cpp
#include "myext_extension.hpp"

namespace duckdb {

static void LoadInternal(ExtensionLoader &loader) {
    // Register functions, types, etc.
    auto my_function = ScalarFunction("my_func",
        {LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        MyFunctionImpl
    );
    loader.RegisterFunction(my_function);
}

void MyExtExtension::Load(ExtensionLoader &loader) {
    LoadInternal(loader);
}

std::string MyExtExtension::Name() {
    return "myext";
}

std::string MyExtExtension::Version() const {
#ifdef EXT_VERSION_MYEXT
    return EXT_VERSION_MYEXT;
#else
    return "";
#endif
}

} // namespace duckdb
```

#### 3. Dynamic Loading Entry Point

The C entry point for loadable extensions:

```cpp
extern "C" {

DUCKDB_CPP_EXTENSION_ENTRY(myext, loader) {
    duckdb::LoadInternal(loader);
}

}
```

### File Organization

**Minimal structure**:
```
src/
├── include/
│   └── myext_extension.hpp
└── myext_extension.cpp
```

**Larger extensions** (see JSON extension):
```
extension/json/
├── include/
│   ├── json_common.hpp
│   ├── json_functions.hpp
│   ├── json_reader.hpp
│   └── ...
├── json_functions/
│   ├── json_extract.cpp
│   ├── json_transform.cpp
│   └── ...
├── json_extension.cpp      # Entry point
├── json_common.cpp
├── json_functions.cpp
└── CMakeLists.txt
```

**Best practices**:
- Keep headers in `include/` subdirectory
- Group related functionality in subdirectories
- One file per major function or function family
- Separate complex logic from registration code

### CMakeLists.txt Configuration

```cmake
cmake_minimum_required(VERSION 3.5)

# Set extension name
set(TARGET_NAME myext)

# Dependencies (optional)
find_package(SomeLibrary REQUIRED)

set(EXTENSION_NAME ${TARGET_NAME}_extension)
set(LOADABLE_EXTENSION_NAME ${TARGET_NAME}_loadable_extension)

project(${TARGET_NAME})
include_directories(src/include)

# List all source files
set(EXTENSION_SOURCES
    src/myext_extension.cpp
    src/myext_functions.cpp
)

# Build both static and loadable versions
build_static_extension(${TARGET_NAME} ${EXTENSION_SOURCES})
build_loadable_extension(${TARGET_NAME} " " ${EXTENSION_SOURCES})

# Link dependencies to both versions
target_link_libraries(${EXTENSION_NAME} SomeLibrary::SomeLibrary)
target_link_libraries(${LOADABLE_EXTENSION_NAME} SomeLibrary::SomeLibrary)

install(
  TARGETS ${EXTENSION_NAME}
  EXPORT "${DUCKDB_EXPORT_SET}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}")
```

### extension_config.cmake

This file tells DuckDB how to load your extension:

```cmake
# Extension from this repo
duckdb_extension_load(myext
    SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}
    LOAD_TESTS
)

# Optional: Load additional extensions for testing
# duckdb_extension_load(parquet)
```

---

## Building Extensions

### Build Modes

#### 1. Release Build (Optimized)

```bash
make
# or
make release
```

Best for:
- Performance testing
- Distribution
- Benchmarking

#### 2. Debug Build

```bash
make debug
```

Features:
- Debug symbols included
- Sanitizers enabled (AddressSanitizer, UBSanitizer)
- Assertions enabled
- No optimization

Best for:
- Development
- Debugging issues
- Finding memory errors

#### 3. RelWithDebInfo Build

```bash
make reldebug
```

Features:
- Optimizations enabled
- Debug symbols included
- Good balance for debugging performance issues

#### 4. RelWithAssert Build

```bash
make relassert
```

Features:
- Optimizations enabled
- Assertions enabled (but no sanitizers)

### Parallel Builds

For faster builds, use Ninja and limit parallel jobs:

```bash
# Use Ninja (much faster than Make)
GEN=ninja make

# Limit parallel jobs (prevent system overload)
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make
```

### Clean Builds

```bash
# Remove build directory
make clean

# Clean and rebuild
make clean && make
```

### Building with Specific Extensions

When developing in the main DuckDB repository:

```bash
# Build DuckDB with specific extensions
DUCKDB_EXTENSIONS='json;icu' make

# Skip specific extensions
SKIP_EXTENSIONS=parquet make
```

### VCPKG and Dependencies

#### Adding Dependencies

1. Add to `vcpkg.json`:
```json
{
    "dependencies": [
        "openssl",
        "boost-algorithm",
        "nlohmann-json"
    ],
    "vcpkg-configuration": {
        "overlay-ports": [
            "./extension-ci-tools/vcpkg_ports"
        ],
        "overlay-triplets": [
            "./extension-ci-tools/toolchains"
        ]
    }
}
```

2. Find and link in `CMakeLists.txt`:
```cmake
find_package(OpenSSL REQUIRED)
find_package(Boost REQUIRED COMPONENTS algorithm)
find_package(nlohmann_json REQUIRED)

target_link_libraries(${EXTENSION_NAME}
    OpenSSL::SSL
    OpenSSL::Crypto
    Boost::algorithm
    nlohmann_json::nlohmann_json
)
target_link_libraries(${LOADABLE_EXTENSION_NAME}
    OpenSSL::SSL
    OpenSSL::Crypto
    Boost::algorithm
    nlohmann_json::nlohmann_json
)
```

3. Build with VCPKG:
```bash
VCPKG_TOOLCHAIN_PATH=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake make
```

#### Building with Multiple VCPKG Extensions

When building DuckDB with multiple extensions that use VCPKG, manifests must be merged:

1. Configure extensions in `extension/extension_config_local.cmake`:
```cmake
duckdb_extension_load(extension_1
    GIT_URL https://github.com/example/extension_1
    GIT_TAG some_git_hash
)
duckdb_extension_load(extension_2
    GIT_URL https://github.com/example/extension_2
    GIT_TAG some_git_hash
)
```

2. Merge manifests:
```bash
make extension_configuration
```

This creates `./build/extension_configuration/vcpkg.json`.

3. Build with merged manifest:
```bash
USE_MERGED_VCPKG_MANIFEST=1 VCPKG_TOOLCHAIN_PATH="/path/to/vcpkg" make
```

---

## Adding Functions to Extensions

Extensions can register various types of functionality through the `ExtensionLoader` API.

### Scalar Functions

Scalar functions operate on individual values and return a single value.

#### Basic Scalar Function

```cpp
#include "duckdb.hpp"
#include "duckdb/function/scalar_function.hpp"
#include "duckdb/common/exception.hpp"

namespace duckdb {

// Function implementation
inline void MyScalarFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input_vector = args.data[0];

    // Use UnaryExecutor for functions with one argument
    UnaryExecutor::Execute<string_t, string_t>(
        input_vector,
        result,
        args.size(),
        [&](string_t input) {
            string input_str = input.GetString();
            string output_str = "Processed: " + input_str;
            return StringVector::AddString(result, output_str);
        }
    );
}

// Registration in LoadInternal()
static void LoadInternal(ExtensionLoader &loader) {
    auto scalar_fn = ScalarFunction(
        "my_function",                    // SQL function name
        {LogicalType::VARCHAR},           // Input types
        LogicalType::VARCHAR,             // Return type
        MyScalarFunction                  // Implementation
    );
    loader.RegisterFunction(scalar_fn);
}

} // namespace duckdb
```

#### Multiple Overloads

```cpp
static void LoadInternal(ExtensionLoader &loader) {
    // Create function set for overloads
    ScalarFunctionSet my_function_set("my_function");

    // Overload 1: VARCHAR input
    my_function_set.AddFunction(ScalarFunction(
        {LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        MyStringImplementation
    ));

    // Overload 2: INTEGER input
    my_function_set.AddFunction(ScalarFunction(
        {LogicalType::INTEGER},
        LogicalType::INTEGER,
        MyIntegerImplementation
    ));

    // Overload 3: Two arguments
    my_function_set.AddFunction(ScalarFunction(
        {LogicalType::VARCHAR, LogicalType::INTEGER},
        LogicalType::VARCHAR,
        MyTwoArgImplementation
    ));

    loader.RegisterFunction(my_function_set);
}
```

#### Binary Functions

```cpp
inline void ConcatFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &left_vector = args.data[0];
    auto &right_vector = args.data[1];

    // Use BinaryExecutor for two arguments
    BinaryExecutor::Execute<string_t, string_t, string_t>(
        left_vector,
        right_vector,
        result,
        args.size(),
        [&](string_t left, string_t right) {
            string result_str = left.GetString() + right.GetString();
            return StringVector::AddString(result, result_str);
        }
    );
}

// Register
auto concat_fn = ScalarFunction(
    "my_concat",
    {LogicalType::VARCHAR, LogicalType::VARCHAR},
    LogicalType::VARCHAR,
    ConcatFunction
);
loader.RegisterFunction(concat_fn);
```

#### Ternary and More Arguments

```cpp
inline void TernaryFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &arg1 = args.data[0];
    auto &arg2 = args.data[1];
    auto &arg3 = args.data[2];

    // Use TernaryExecutor
    TernaryExecutor::Execute<int32_t, int32_t, int32_t, int32_t>(
        arg1, arg2, arg3,
        result,
        args.size(),
        [&](int32_t a, int32_t b, int32_t c) {
            return a + b + c;
        }
    );
}

// For 4+ arguments, use generic executor
inline void QuaternaryFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    // Process args.data[0], args.data[1], args.data[2], args.data[3]
    // Use loops or specialized logic
}
```

### Aggregate Functions

Aggregate functions operate on groups of rows (e.g., SUM, AVG, COUNT).

```cpp
#include "duckdb/function/aggregate_function.hpp"

namespace duckdb {

// State structure for aggregate
struct SumState {
    int64_t sum;
    bool is_set;
};

// Initialize state
static void SumInitialize(data_ptr_t state_ptr) {
    auto state = reinterpret_cast<SumState *>(state_ptr);
    state->sum = 0;
    state->is_set = false;
}

// Update state with new value
static void SumUpdate(Vector inputs[], AggregateInputData &aggr_input_data,
                      idx_t input_count, Vector &state_vector, idx_t count) {
    auto &input = inputs[0];
    UnifiedVectorFormat input_data;
    input.ToUnifiedFormat(count, input_data);

    auto states = FlatVector::GetData<SumState *>(state_vector);
    auto input_values = UnifiedVectorFormat::GetData<int64_t>(input_data);

    for (idx_t i = 0; i < count; i++) {
        auto state = states[i];
        auto idx = input_data.sel->get_index(i);
        if (!input_data.validity.RowIsValid(idx)) {
            continue; // Skip NULL values
        }
        state->sum += input_values[idx];
        state->is_set = true;
    }
}

// Combine two states (for parallel execution)
static void SumCombine(Vector &state_vector, Vector &combined_vector,
                       AggregateInputData &aggr_input_data, idx_t count) {
    auto combined_states = FlatVector::GetData<SumState *>(combined_vector);
    auto states = FlatVector::GetData<SumState *>(state_vector);

    for (idx_t i = 0; i < count; i++) {
        if (states[i]->is_set) {
            combined_states[i]->sum += states[i]->sum;
            combined_states[i]->is_set = true;
        }
    }
}

// Finalize and return result
static void SumFinalize(Vector &state_vector, AggregateInputData &aggr_input_data,
                        Vector &result, idx_t count, idx_t offset) {
    auto states = FlatVector::GetData<SumState *>(state_vector);
    auto result_data = FlatVector::GetData<int64_t>(result);

    for (idx_t i = 0; i < count; i++) {
        if (!states[i]->is_set) {
            FlatVector::SetNull(result, i, true);
        } else {
            result_data[i] = states[i]->sum;
        }
    }
}

// Register aggregate function
static void LoadInternal(ExtensionLoader &loader) {
    auto sum_function = AggregateFunction(
        "my_sum",                         // SQL function name
        {LogicalType::BIGINT},            // Input types
        LogicalType::BIGINT,              // Return type
        AggregateFunction::StateSize<SumState>,
        AggregateFunction::StateInitialize<SumState, SumInitialize>,
        SumUpdate,
        SumCombine,
        SumFinalize
    );
    loader.RegisterFunction(sum_function);
}

} // namespace duckdb
```

### Table Functions

Table functions return tables (multiple rows and columns).

```cpp
#include "duckdb/function/table_function.hpp"

namespace duckdb {

// Bind function - validates inputs and prepares execution
static unique_ptr<FunctionData> MyTableBind(ClientContext &context,
                                            TableFunctionBindInput &input,
                                            vector<LogicalType> &return_types,
                                            vector<string> &names) {
    // Validate input parameters
    if (input.inputs.size() != 1) {
        throw BinderException("my_table requires exactly 1 argument");
    }

    // Define return columns
    return_types.push_back(LogicalType::INTEGER);
    return_types.push_back(LogicalType::VARCHAR);
    names.push_back("id");
    names.push_back("name");

    // Return bind data if needed
    return nullptr;
}

// Initialize state for execution
static unique_ptr<GlobalTableFunctionState> MyTableInit(ClientContext &context,
                                                        TableFunctionInitInput &input) {
    // Initialize global state if needed
    return nullptr;
}

// Execute and produce output
static void MyTableFunction(ClientContext &context, TableFunctionInput &data,
                           DataChunk &output) {
    // Generate rows
    idx_t current_row = 0;
    idx_t max_rows = 10;

    while (current_row < max_rows && output.size() < STANDARD_VECTOR_SIZE) {
        auto id_vector = FlatVector::GetData<int32_t>(output.data[0]);
        auto name_vector = FlatVector::GetData<string_t>(output.data[1]);

        idx_t this_count = MinValue<idx_t>(
            max_rows - current_row,
            STANDARD_VECTOR_SIZE - output.size()
        );

        for (idx_t i = 0; i < this_count; i++) {
            id_vector[output.size()] = current_row + i;
            name_vector[output.size()] = StringVector::AddString(
                output.data[1],
                "Item " + to_string(current_row + i)
            );
            output.size()++;
        }

        current_row += this_count;
    }

    // If output.size() == 0, function is done
}

// Register table function
static void LoadInternal(ExtensionLoader &loader) {
    TableFunction my_table(
        "my_table",                       // SQL function name
        {LogicalType::INTEGER},           // Input parameters
        MyTableFunction,                  // Execute function
        MyTableBind,                      // Bind function
        MyTableInit                       // Init function
    );
    loader.RegisterFunction(my_table);
}

} // namespace duckdb
```

### Custom Types

```cpp
#include "duckdb/common/types/value.hpp"

static void LoadInternal(ExtensionLoader &loader) {
    // Register custom type
    auto json_type = LogicalType::JSON();
    loader.RegisterType(LogicalType::JSON_TYPE_NAME, std::move(json_type));
}
```

### Cast Functions

```cpp
#include "duckdb/function/cast/cast_function_set.hpp"

// Cast function implementation
static bool CastVarcharToJSON(Vector &source, Vector &result, idx_t count,
                              CastParameters &parameters) {
    // Implement casting logic
    // Return false on cast failure
    return true;
}

// Register cast
static void LoadInternal(ExtensionLoader &loader) {
    loader.RegisterCastFunction(
        LogicalType::VARCHAR,             // Source type
        LogicalType::JSON(),              // Target type
        BoundCastInfo(CastVarcharToJSON), // Cast function
        100                               // Cost (lower = preferred)
    );
}
```

### Macro Functions

```cpp
#include "duckdb/parser/expression/function_expression.hpp"
#include "duckdb/catalog/default/default_functions.hpp"

static const DefaultMacro MY_MACROS[] = {
    {DEFAULT_SCHEMA,
     "my_macro",                          // Macro name
     {"x", "y", nullptr},                 // Parameters
     {{nullptr, nullptr}},
     "x + y + 1"},                        // Macro body (SQL expression)
    {nullptr, nullptr, {nullptr}, {{nullptr, nullptr}}, nullptr}  // Sentinel
};

static void LoadInternal(ExtensionLoader &loader) {
    for (idx_t index = 0; MY_MACROS[index].name != nullptr; index++) {
        auto info = DefaultFunctionGenerator::CreateInternalMacroInfo(MY_MACROS[index]);
        loader.RegisterFunction(*info);
    }
}
```

### Copy Functions

Copy functions define how to read/write data in custom formats:

```cpp
#include "duckdb/function/copy_function.hpp"

// Register copy function for custom format
static void LoadInternal(ExtensionLoader &loader) {
    CopyFunction copy_fun = GetMyFormatCopyFunction();
    loader.RegisterFunction(copy_fun);

    // Can register with alternative names
    copy_fun.extension = "myformat2";
    copy_fun.name = "myformat2";
    loader.RegisterFunction(copy_fun);
}
```

---

## Testing Extensions

DuckDB strongly prefers **SQLLogicTest** for extension testing. C++ unit tests should be used sparingly for low-level functionality.

### SQLLogicTest

SQLLogicTests are located in `test/sql/` and use the `.test` extension.

#### Basic Test Structure

```sql
# name: test/sql/myext.test
# description: Test my extension functionality
# group: [sql]

# Test that function doesn't exist before loading
statement error
SELECT my_function('test');
----
Catalog Error: Scalar Function with name my_function does not exist!

# Require the extension
require myext

# Test basic functionality
query I
SELECT my_function('hello');
----
Processed: hello

# Test with NULL
query I
SELECT my_function(NULL);
----
NULL

# Test multiple rows
query I
SELECT my_function(name) FROM (VALUES ('Alice'), ('Bob')) t(name);
----
Processed: Alice
Processed: Bob
```

#### Test Directives

**statement ok** - SQL should execute successfully:
```sql
statement ok
CREATE TABLE t (x INTEGER);

statement ok
INSERT INTO t VALUES (1), (2), (3);
```

**statement error** - SQL should fail with error:
```sql
statement error
SELECT my_function();
----
Binder Error: No function matches
```

**query** - Execute query and check results:
```sql
# query [type code] [result_mode]
# Types: I=INTEGER, T=TEXT, R=REAL, etc.
# Modes: (none)=row-by-row, rowsort=sorted, valuesort=all sorted

query I
SELECT 1;
----
1

query IT
SELECT 1, 'hello';
----
1	hello

query I rowsort
SELECT x FROM t;
----
1
2
3
```

**require** - Load extension before tests:
```sql
require myext
require parquet
```

**load** - Load additional extension (deprecated, use require):
```sql
load myext
```

#### Testing Different Data Types

```sql
# Integers
query I
SELECT my_sum(42);
----
42

# Floating point
query R
SELECT my_avg(3.14);
----
3.14

# Strings
query T
SELECT my_concat('hello', 'world');
----
helloworld

# Dates
query T
SELECT my_format_date(DATE '2024-01-15');
----
2024-01-15

# Lists
query I
SELECT len(my_create_list(1, 2, 3));
----
3

# Structs
query T
SELECT my_struct_field({'a': 1, 'b': 2}, 'a');
----
1
```

#### Testing Error Conditions

```sql
# Test invalid input
statement error
SELECT my_function('invalid input');
----
Invalid Format Error: Expected valid input

# Test NULL handling
query I
SELECT my_nullable_function(NULL);
----
NULL

# Test type errors
statement error
SELECT my_function(123);
----
Binder Error: Cannot bind function
```

#### Testing Aggregate Functions

```sql
statement ok
CREATE TABLE numbers (x INTEGER);

statement ok
INSERT INTO numbers VALUES (1), (2), (3), (4), (5);

query I
SELECT my_sum(x) FROM numbers;
----
15

query I
SELECT my_sum(x) FROM numbers WHERE x > 10;
----
NULL

query II
SELECT x % 2, my_sum(x) FROM numbers GROUP BY x % 2 ORDER BY 1;
----
0	6
1	9
```

#### Testing Table Functions

```sql
query IT
SELECT * FROM my_table_func(5);
----
0	Item 0
1	Item 1
2	Item 2
3	Item 3
4	Item 4

query I
SELECT COUNT(*) FROM my_table_func(100);
----
100
```

#### Complex Multi-Statement Tests

```sql
# Test complete workflow
statement ok
CREATE TABLE source (id INTEGER, data VARCHAR);

statement ok
INSERT INTO source VALUES (1, 'foo'), (2, 'bar');

query IT
SELECT id, my_transform(data) FROM source ORDER BY id;
----
1	transformed: foo
2	transformed: bar

statement ok
DROP TABLE source;
```

### Running Tests

```bash
# Run all SQL tests (release build)
make test

# Run all SQL tests (debug build)
make test_debug

# Run specific test file
./build/release/test/unittest "test/sql/myext.test"

# Run all fast tests
./build/debug/test/unittest

# Run tests matching a tag
./build/debug/test/unittest "[myext]"

# Run specific test by name
./build/debug/test/unittest "MySpecificTest"
```

### Slow Tests

Tests that take more than a few seconds should be marked as slow:

**SQLLogicTest**: Use `.test_slow` extension:
```bash
mv test/sql/longtest.test test/sql/longtest.test_slow
```

### C++ Unit Tests (Discouraged)

Only use C++ tests for low-level functionality that's hard to test via SQL.

Location: `test/extension/` (if building in main DuckDB repo)

```cpp
#include "catch.hpp"
#include "test_helpers.hpp"

using namespace duckdb;

TEST_CASE("Test my extension functionality", "[myext]") {
    DuckDB db(nullptr);
    Connection con(db);

    // Load extension
    con.Query("LOAD myext");

    // Test basic functionality
    auto result = con.Query("SELECT my_function('test')");
    REQUIRE(result->success);
    REQUIRE(result->RowCount() == 1);

    auto value = result->GetValue(0, 0);
    REQUIRE(value.ToString() == "Processed: test");
}
```

---

## Debugging Extensions

### Debug Build

Always use debug builds when debugging:

```bash
make debug
```

Debug builds include:
- Debug symbols
- AddressSanitizer (detects memory errors)
- UBSanitizer (detects undefined behavior)
- Assertions enabled

### Using GDB/LLDB

```bash
# Build debug version
make debug

# Run with debugger
lldb ./build/debug/duckdb
(lldb) run

# Or debug tests
lldb ./build/debug/test/unittest
(lldb) run "test/sql/myext.test"
```

Set breakpoints in your extension:

```bash
(lldb) breakpoint set --name MyScalarFunction
(lldb) breakpoint set --file myext_extension.cpp --line 42
(lldb) run
```

### CLion Setup

#### Opening the Project

1. Open `./duckdb/CMakeLists.txt` (NOT the root CMakeLists.txt)
2. Go to `Tools -> CMake -> Change Project Root`
3. Set project root to your extension repository root

#### Configure CMake

In `Settings -> Build, Execution, Deploy -> CMake`:

1. Add Debug profile:
   - Name: Debug
   - Build type: Debug
   - Build directory: `../build/debug`

2. Add Release profile:
   - Name: Release
   - Build type: Release
   - Build directory: `../build/release`

3. Run `make debug` once to initialize CMake directory

#### Create Run Configurations

**For testing**:
1. Go to `Run -> Edit Configurations`
2. Click `+ -> CMake Application`
3. Name: "Extension Tests"
4. Target: `unittest`
5. Executable: `unittest`
6. Program arguments: `--test-dir ../../.. [sql]`
7. Working directory: `<project-root>/build/debug/test`

**For DuckDB shell** (less reliable):
1. Click `+ -> CMake Application`
2. Target: `duckdb`
3. Executable: `duckdb`

#### Debugging

1. Set breakpoints in your extension code
2. Select "Extension Tests" run configuration
3. Click Debug button (or press Cmd+D / Ctrl+D)
4. Step through code, inspect variables

### Print Debugging

```cpp
#include "duckdb/common/printer.hpp"

void MyFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    // Print to stdout
    Printer::Print("MyFunction called with " + to_string(args.size()) + " rows\n");

    // Print variable
    string debug_info = "Processing: " + some_value;
    Printer::Print(debug_info + "\n");

    // ... function implementation
}
```

### Assertions

Use `D_ASSERT` for invariants (removed in release builds):

```cpp
void MyFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    // This should never happen in correct code
    D_ASSERT(args.ColumnCount() > 0);  // Comment explains what this checks
    D_ASSERT(args.data[0].GetType() == LogicalType::VARCHAR);

    // ... implementation
}
```

**Important**: Assertions should check programmer errors, not user input errors.

### Memory Debugging

AddressSanitizer (enabled in debug builds) catches:
- Use-after-free
- Heap buffer overflow
- Stack buffer overflow
- Memory leaks
- Double-free

Example output:
```
=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free
READ of size 4 at 0x60300000eff0 thread T0
    #0 0x12345 in MyFunction myext_extension.cpp:42
    ...
```

To disable sanitizers (not recommended):
```bash
DISABLE_SANITIZER=1 make debug
```

### Performance Profiling

For performance debugging, use RelWithDebInfo:

```bash
make reldebug
```

Then profile with standard tools:

```bash
# Linux - perf
perf record ./build/reldebug/duckdb < query.sql
perf report

# macOS - Instruments
instruments -t "Time Profiler" ./build/reldebug/duckdb

# Generic - gprof
# (requires ENABLE_PROFILING=1 cmake flag)
```

---

## Distribution Options

Extension binaries are version-specific and platform-specific. You need separate binaries for each combination of:
- DuckDB version (e.g., v1.4.1, v1.4.2)
- Platform (e.g., linux_amd64, osx_arm64, windows_amd64)

### Extension URL Format

Extensions are served from URLs with this structure:

```
http://extensions.duckdb.org/v1.4.1/osx_arm64/myext.duckdb_extension.gz
│                              │       │       │      └─ File extension + compression
│                              │       │       └─ Extension name
│                              │       └─ Platform
│                              └─ DuckDB version
└─ Extension registry
```

Local installation path:
```
~/.duckdb/extensions/v1.4.1/osx_arm64/myext.duckdb_extension
│          │          │       │       └─ Extension name + file extension
│          │          │       └─ Platform
│          │          └─ DuckDB version
│          └─ Extensions directory
└─ Configuration folder
```

### Option 1: Community Extensions (Recommended)

**Repository**: https://github.com/duckdb/community-extensions

Community extensions are:
- Built automatically by DuckDB infrastructure
- Signed with community extensions signature
- Discoverable through DuckDB
- Easy to install for users

#### Submission Process

1. **Ensure your extension builds with the template CI**:
   - The template's GitHub Actions workflow must pass
   - All platforms should build successfully

2. **Create descriptor file**:

Create `extensions/<your_extension>.yaml`:

```yaml
extension:
  name: myext
  description: Brief description of what myext does
  version: 1.0.0
  language: C++
  build: cmake
  license: MIT
  maintainers:
    - yourusername

repo:
  github: yourusername/duckdb-myext
  ref: v1.0.0
```

3. **Submit PR to community-extensions**:
```bash
git clone https://github.com/duckdb/community-extensions
cd community-extensions
# Create descriptor file
git checkout -b add-myext
git add extensions/myext.yaml
git commit -m "Add myext extension"
git push origin add-myext
# Open PR on GitHub
```

4. **CI builds your extension**:
   - Automatically builds for all platforms
   - Runs tests
   - Signs binaries

5. **After merge, users can install**:
```sql
INSTALL myext FROM community;
LOAD myext;
```

### Option 2: Custom Extension Repository

For more control, you can host your own extension repository.

#### Setting Up Custom Repository

1. **Build extension binaries** (via GitHub Actions or locally)

2. **Upload to hosting** with correct structure:
```
https://my-extensions.example.com/
├── v1.4.1/
│   ├── linux_amd64/
│   │   └── myext.duckdb_extension.gz
│   ├── osx_arm64/
│   │   └── myext.duckdb_extension.gz
│   └── windows_amd64/
│       └── myext.duckdb_extension.gz
└── v1.4.2/
    └── ...
```

3. **Configure GitHub Actions for deployment**:

The template includes `.github/workflows/MainDistributionPipeline.yml`. To enable deployment:

```yaml
jobs:
  duckdb-stable-build:
    name: Build extension binaries
    uses: duckdb/extension-ci-tools/.github/workflows/_extension_distribution.yml@v1.4.1
    with:
      duckdb_version: v1.4.1
      ci_tools_version: v1.4.1
      extension_name: myext
      # Enable deployment
      deploy_latest: ${{ github.ref == 'refs/heads/main' }}
      deploy_version: ${{ startsWith(github.ref, 'refs/tags/v') }}
    secrets:
      # Add deployment secrets
      s3_id_dev: ${{ secrets.S3_ID_DEV }}
      s3_key_dev: ${{ secrets.S3_KEY_DEV }}
      s3_bucket_dev: my-extensions-bucket
```

4. **Users install with custom repository**:
```sql
SET custom_extension_repository='https://my-extensions.example.com';
INSTALL myext;
LOAD myext;
```

**Note**: Custom repositories require `allow_unsigned_extensions` unless you implement signing.

### Option 3: GitHub Artifacts (Development)

For development and testing, use GitHub Actions artifacts:

1. **Push to GitHub** - Artifacts built automatically by template CI

2. **Download artifact** from Actions tab

3. **Load unsigned extension**:
```bash
# Start DuckDB with unsigned extensions allowed
duckdb -unsigned

# Or set in SQL
SET allow_unsigned_extensions=true;
```

```sql
-- Load from file path
LOAD '/path/to/myext.duckdb_extension';
```

**Use case**: Testing, internal distribution, development

### Distribution Comparison

| Method | Signing | Discovery | Maintenance | Use Case |
|--------|---------|-----------|-------------|----------|
| Community Extensions | Yes | Automatic | DuckDB team | Public extensions |
| Custom Repository | Optional | Manual | You | Private/commercial |
| GitHub Artifacts | No | Manual | You | Development |

---

## Versioning and DuckDB Compatibility

### Extension ABI Types

DuckDB extensions use different ABI types:

1. **CPP** (C++ ABI):
   - Most extensions use this
   - Version must match DuckDB exactly
   - Breaks with every DuckDB release
   - Requires recompilation for each DuckDB version

2. **C_STRUCT** (Stable C ABI):
   - Uses C API via `duckdb_ext_api_v1` struct
   - Version must be equal or higher
   - More stable across versions
   - Limited functionality compared to C++ API

3. **C_STRUCT_UNSTABLE** (Unstable C ABI):
   - Uses C API including unstable functions
   - Version must match precisely

Most extensions use the **C++ ABI** because it provides full access to DuckDB internals.

### DuckDB Submodule

The template includes DuckDB as a git submodule at `./duckdb/`. This pins your extension to a specific DuckDB version.

Check current version:
```bash
cd duckdb
git describe --tags
# Output: v1.4.1
```

### Updating to New DuckDB Versions

When a new DuckDB version is released, you need to update your extension:

#### Step 1: Update Submodules

```bash
# Update duckdb submodule to latest release
cd duckdb
git fetch --tags
git checkout v1.4.2  # New version
cd ..

# Update extension-ci-tools to matching branch
cd extension-ci-tools
git fetch
git checkout v1.4.2  # Must match DuckDB version
cd ..

# Commit changes
git add duckdb extension-ci-tools
git commit -m "Update to DuckDB v1.4.2"
```

#### Step 2: Update CI Workflows

Edit `.github/workflows/MainDistributionPipeline.yml`:

```yaml
jobs:
  duckdb-stable-build:
    name: Build extension binaries
    uses: duckdb/extension-ci-tools/.github/workflows/_extension_distribution.yml@v1.4.2  # Update version
    with:
      duckdb_version: v1.4.2  # Update version
      ci_tools_version: v1.4.2  # Update version
      extension_name: myext
```

#### Step 3: Handle API Changes

DuckDB's C++ API is **not stable**. Expect breaking changes between versions.

**Check for changes**:

1. **Read release notes**: https://github.com/duckdb/duckdb/releases
2. **Check core extension patches**: https://github.com/duckdb/duckdb/commits/main/.github/patches/extensions
3. **Review header file history**: Check git history of changed headers

**Common API changes**:
- Function signature changes
- Renamed types or classes
- Changed namespace organization
- New required parameters
- Deprecated functions removed

**Example migration**:

```cpp
// DuckDB v1.4.1
auto result = StringVector::AddString(vector, str);

// DuckDB v1.4.2 (hypothetical change)
auto result = StringVector::AddString(vector, str, length);
```

#### Step 4: Build and Test

```bash
# Clean build
make clean

# Build with new version
make

# Run tests
make test
```

#### Step 5: Update Documentation

Update `README.md` to reflect supported DuckDB version(s):

```markdown
## Compatibility

- DuckDB v1.4.1: Use release v1.0.0
- DuckDB v1.4.2: Use release v1.1.0
```

### Supporting Multiple DuckDB Versions

#### Option 1: Separate Branches

```bash
# Branch per DuckDB version
git checkout -b duckdb-v1.4.1
# Configure for v1.4.1

git checkout -b duckdb-v1.4.2
# Configure for v1.4.2
```

Build and release from each branch.

#### Option 2: Version Detection

Use preprocessor macros to support multiple versions:

```cpp
#include "duckdb.hpp"

// Check DuckDB version
#ifndef DUCKDB_PATCH_VERSION
#define DUCKDB_PATCH_VERSION 0
#endif

void MyFunction(DataChunk &args, ExpressionState &state, Vector &result) {
#if DUCKDB_MAJOR_VERSION == 1 && DUCKDB_MINOR_VERSION >= 4
    // v1.4.x and later API
    auto str = StringVector::AddString(result, value, length);
#else
    // Pre-v1.4.x API
    auto str = StringVector::AddString(result, value);
#endif
}
```

**Warning**: This approach is fragile and hard to maintain. Prefer separate branches for major API changes.

#### Option 3: Multiple CI Jobs

In `.github/workflows/MainDistributionPipeline.yml`:

```yaml
jobs:
  duckdb-v1_4_1:
    uses: duckdb/extension-ci-tools/.github/workflows/_extension_distribution.yml@v1.4.1
    with:
      duckdb_version: v1.4.1
      extension_name: myext

  duckdb-v1_4_2:
    uses: duckdb/extension-ci-tools/.github/workflows/_extension_distribution.yml@v1.4.2
    with:
      duckdb_version: v1.4.2
      extension_name: myext
```

This builds binaries for multiple DuckDB versions in a single CI run.

### Version String

Extension version is set via `EXT_VERSION_<NAME>` macro:

```cpp
std::string MyExtExtension::Version() const {
#ifdef EXT_VERSION_MYEXT
    return EXT_VERSION_MYEXT;
#else
    return "";
#endif
}
```

This macro is set by the build system based on git tags. To set a version:

```bash
git tag v1.0.0
git push origin v1.0.0
```

---

## Complete Walkthrough: Creating a Simple Extension

Let's create a complete extension called `text_utils` with functions for text manipulation.

### Step 1: Create Repository from Template

1. Go to https://github.com/duckdb/extension-template
2. Click "Use this template" -> "Create a new repository"
3. Name: `duckdb-text-utils`
4. Create repository

### Step 2: Clone and Set Up

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/yourusername/duckdb-text-utils.git
cd duckdb-text-utils

# Set up VCPKG (if you need dependencies - we don't for this example)
# For this example, we'll skip VCPKG and remove the OpenSSL dependency

# Rename the extension
python3 ./scripts/bootstrap-template.py text_utils

# Remove the bootstrap script
rm ./scripts/bootstrap-template.py
```

### Step 3: Remove OpenSSL Dependency

Since our extension doesn't need OpenSSL, remove it:

**Edit `vcpkg.json`**:
```json
{
    "dependencies": [],
    "vcpkg-configuration": {
        "overlay-ports": [
            "./extension-ci-tools/vcpkg_ports"
        ],
        "overlay-triplets": [
            "./extension-ci-tools/toolchains"
        ]
    }
}
```

**Edit `CMakeLists.txt`**:
```cmake
cmake_minimum_required(VERSION 3.5)

set(TARGET_NAME text_utils)

# Remove: find_package(OpenSSL REQUIRED)

set(EXTENSION_NAME ${TARGET_NAME}_extension)
set(LOADABLE_EXTENSION_NAME ${TARGET_NAME}_loadable_extension)

project(${TARGET_NAME})
include_directories(src/include)

set(EXTENSION_SOURCES src/text_utils_extension.cpp)

build_static_extension(${TARGET_NAME} ${EXTENSION_SOURCES})
build_loadable_extension(${TARGET_NAME} " " ${EXTENSION_SOURCES})

# Remove: target_link_libraries lines for OpenSSL

install(
  TARGETS ${EXTENSION_NAME}
  EXPORT "${DUCKDB_EXPORT_SET}"
  LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}")
```

### Step 4: Implement Extension Functions

**Edit `src/text_utils_extension.cpp`**:

```cpp
#define DUCKDB_EXTENSION_MAIN

#include "text_utils_extension.hpp"
#include "duckdb.hpp"
#include "duckdb/common/exception.hpp"
#include "duckdb/function/scalar_function.hpp"
#include <duckdb/parser/parsed_data/create_scalar_function_info.hpp>

namespace duckdb {

// Function 1: reverse(str) - Reverse a string
inline void ReverseFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input_vector = args.data[0];
    UnaryExecutor::Execute<string_t, string_t>(
        input_vector,
        result,
        args.size(),
        [&](string_t input) {
            string str = input.GetString();
            std::reverse(str.begin(), str.end());
            return StringVector::AddString(result, str);
        }
    );
}

// Function 2: word_count(str) - Count words in a string
inline void WordCountFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input_vector = args.data[0];
    UnaryExecutor::Execute<string_t, int32_t>(
        input_vector,
        result,
        args.size(),
        [&](string_t input) {
            string str = input.GetString();
            if (str.empty()) {
                return 0;
            }

            int32_t count = 0;
            bool in_word = false;

            for (char c : str) {
                if (std::isspace(c)) {
                    in_word = false;
                } else if (!in_word) {
                    in_word = true;
                    count++;
                }
            }

            return count;
        }
    );
}

// Function 3: repeat(str, n) - Repeat a string n times
inline void RepeatFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &str_vector = args.data[0];
    auto &count_vector = args.data[1];

    BinaryExecutor::Execute<string_t, int32_t, string_t>(
        str_vector,
        count_vector,
        result,
        args.size(),
        [&](string_t str, int32_t count) {
            if (count < 0) {
                throw InvalidInputException("Repeat count must be non-negative");
            }
            if (count == 0) {
                return StringVector::AddString(result, "");
            }

            string input = str.GetString();
            string output;
            output.reserve(input.size() * count);

            for (int32_t i = 0; i < count; i++) {
                output += input;
            }

            return StringVector::AddString(result, output);
        }
    );
}

// Function 4: title_case(str) - Convert to title case
inline void TitleCaseFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input_vector = args.data[0];
    UnaryExecutor::Execute<string_t, string_t>(
        input_vector,
        result,
        args.size(),
        [&](string_t input) {
            string str = input.GetString();
            bool new_word = true;

            for (size_t i = 0; i < str.length(); i++) {
                if (std::isspace(str[i])) {
                    new_word = true;
                } else if (new_word) {
                    str[i] = std::toupper(str[i]);
                    new_word = false;
                } else {
                    str[i] = std::tolower(str[i]);
                }
            }

            return StringVector::AddString(result, str);
        }
    );
}

static void LoadInternal(ExtensionLoader &loader) {
    // Register reverse function
    auto reverse_fn = ScalarFunction(
        "reverse",
        {LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        ReverseFunction
    );
    loader.RegisterFunction(reverse_fn);

    // Register word_count function
    auto word_count_fn = ScalarFunction(
        "word_count",
        {LogicalType::VARCHAR},
        LogicalType::INTEGER,
        WordCountFunction
    );
    loader.RegisterFunction(word_count_fn);

    // Register repeat function
    auto repeat_fn = ScalarFunction(
        "repeat",
        {LogicalType::VARCHAR, LogicalType::INTEGER},
        LogicalType::VARCHAR,
        RepeatFunction
    );
    loader.RegisterFunction(repeat_fn);

    // Register title_case function
    auto title_case_fn = ScalarFunction(
        "title_case",
        {LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        TitleCaseFunction
    );
    loader.RegisterFunction(title_case_fn);
}

void TextUtilsExtension::Load(ExtensionLoader &loader) {
    LoadInternal(loader);
}

std::string TextUtilsExtension::Name() {
    return "text_utils";
}

std::string TextUtilsExtension::Version() const {
#ifdef EXT_VERSION_TEXT_UTILS
    return EXT_VERSION_TEXT_UTILS;
#else
    return "";
#endif
}

} // namespace duckdb

extern "C" {

DUCKDB_CPP_EXTENSION_ENTRY(text_utils, loader) {
    duckdb::LoadInternal(loader);
}

}
```

### Step 5: Create Tests

**Edit `test/sql/text_utils.test`**:

```sql
# name: test/sql/text_utils.test
# description: Test text_utils extension
# group: [sql]

# Test that functions don't exist before loading
statement error
SELECT reverse('hello');
----
Catalog Error: Scalar Function with name reverse does not exist!

# Require the extension
require text_utils

# Test reverse function
query I
SELECT reverse('hello');
----
olleh

query I
SELECT reverse('DuckDB');
----
BDkcuD

query I
SELECT reverse(NULL);
----
NULL

# Test word_count function
query I
SELECT word_count('hello world');
----
2

query I
SELECT word_count('one two three four five');
----
5

query I
SELECT word_count('   extra   spaces   ');
----
2

query I
SELECT word_count('');
----
0

query I
SELECT word_count(NULL);
----
NULL

# Test repeat function
query I
SELECT repeat('ab', 3);
----
ababab

query I
SELECT repeat('x', 5);
----
xxxxx

query I
SELECT repeat('hello', 0);
----
<empty>

query I
SELECT repeat('test', 1);
----
test

statement error
SELECT repeat('bad', -1);
----
Invalid Input Error: Repeat count must be non-negative

# Test title_case function
query I
SELECT title_case('hello world');
----
Hello World

query I
SELECT title_case('the quick BROWN fox');
----
The Quick Brown Fox

query I
SELECT title_case('UPPERCASE');
----
Uppercase

query I
SELECT title_case(NULL);
----
NULL

# Test with table data
statement ok
CREATE TABLE texts (id INTEGER, text VARCHAR);

statement ok
INSERT INTO texts VALUES
    (1, 'hello'),
    (2, 'world'),
    (3, 'duckdb is awesome');

query IT
SELECT id, reverse(text) FROM texts ORDER BY id;
----
1	olleh
2	dlrow
3	emosewa si bckud

query II
SELECT id, word_count(text) FROM texts ORDER BY id;
----
1	1
2	1
3	3

statement ok
DROP TABLE texts;

# Test combining functions
query I
SELECT reverse(title_case('hello world'));
----
dlroW olleH

query I
SELECT word_count(repeat('hello world ', 3));
----
6
```

### Step 6: Build the Extension

```bash
# Build release version
make

# Or build with ninja for speed
GEN=ninja make
```

Expected output:
```
...
-- Building extension 'text_utils'
...
[100%] Built target text_utils_loadable_extension
```

### Step 7: Test the Extension

```bash
# Run tests
make test
```

Expected output:
```
...
test/sql/text_utils.test ....................................... Ok (0.01s)
...
All tests passed
```

### Step 8: Try It Interactively

```bash
./build/release/duckdb
```

```sql
D SELECT reverse('Hello DuckDB!');
┌──────────────────────┐
│ reverse('Hello Duc…  │
│       varchar        │
├──────────────────────┤
│ !BDkcuD olleH        │
└──────────────────────┘

D SELECT word_count('DuckDB is an amazing database');
┌─────────────────────────────────────────────┐
│ word_count('DuckDB is an amazing database') │
│                    int32                    │
├─────────────────────────────────────────────┤
│                                           5 │
└─────────────────────────────────────────────┘

D SELECT repeat('Duck', 3);
┌───────────────────┐
│ repeat('Duck', 3) │
│      varchar      │
├───────────────────┤
│ DuckDuckDuck      │
└───────────────────┘

D SELECT title_case('the quick brown fox');
┌─────────────────────────────────────┐
│ title_case('the quick brown fox')   │
│               varchar               │
├─────────────────────────────────────┤
│ The Quick Brown Fox                 │
└─────────────────────────────────────┘
```

### Step 9: Commit and Push

```bash
git add .
git commit -m "Implement text_utils extension with reverse, word_count, repeat, and title_case functions"
git push origin main
```

### Step 10: Distribute

#### Option A: Submit to Community Extensions

1. Fork https://github.com/duckdb/community-extensions

2. Create `extensions/text_utils.yaml`:
```yaml
extension:
  name: text_utils
  description: Text manipulation utilities for DuckDB
  version: 1.0.0
  language: C++
  build: cmake
  license: MIT
  maintainers:
    - yourusername

repo:
  github: yourusername/duckdb-text-utils
  ref: main
```

3. Submit PR

4. After merge, users install with:
```sql
INSTALL text_utils FROM community;
LOAD text_utils;
SELECT reverse('Hello!');
```

#### Option B: Use GitHub Artifacts

Users can:

1. Download artifact from your GitHub Actions
2. Start DuckDB with `-unsigned`
3. Load with `LOAD '/path/to/text_utils.duckdb_extension'`

### Step 11: Update README

```markdown
# DuckDB Text Utils Extension

Text manipulation utilities for DuckDB.

## Functions

- `reverse(str)` - Reverse a string
- `word_count(str)` - Count words in a string
- `repeat(str, n)` - Repeat a string n times
- `title_case(str)` - Convert to title case

## Installation

### From Community Extensions
```sql
INSTALL text_utils FROM community;
LOAD text_utils;
```

### From Source
```bash
git clone --recurse-submodules https://github.com/yourusername/duckdb-text-utils
cd duckdb-text-utils
make
./build/release/duckdb
```

## Examples

```sql
SELECT reverse('hello');  -- 'olleh'
SELECT word_count('hello world');  -- 2
SELECT repeat('ab', 3);  -- 'ababab'
SELECT title_case('hello world');  -- 'Hello World'
```

## Compatibility

Compatible with DuckDB v1.4.1+

## License

MIT License
```

---

## Additional Resources

### Official Documentation

- **Extension Template**: https://github.com/duckdb/extension-template
- **Community Extensions**: https://github.com/duckdb/community-extensions
- **DuckDB Documentation**: https://duckdb.org/docs/
- **Extension Distribution**: `extension/ExtensionDistribution.md` (in DuckDB repository)
- **Building Extensions**: `extension/README.md` (in DuckDB repository)

### Example Extensions

**In-tree examples** (in DuckDB repo):
- **JSON** (`extension/json/`) - Custom types, casts, multiple function types
- **ICU** - Complex external library integration
- **Parquet** - File format support, table functions
- **TPC-H** (`extension/tpch/`) - Table functions, data generation

**Out-of-tree examples**:
- **sqlite_scanner** - External database connector
- **spatial** - Complex geospatial functionality
- **postgres_scanner** - Connection pooling, remote queries

### Build System Documentation

- **Main Makefile**: `Makefile` (in DuckDB repository root)
- **CMake configuration**: `CMakeLists.txt` (in DuckDB repository root)
- **Extension config**: `extension/extension_config.cmake`

### Key Header Files

Extension API:
- `src/include/duckdb/main/extension.hpp`
- `src/include/duckdb/main/extension/extension_loader.hpp`

Function types:
- `duckdb/function/scalar_function.hpp` - Scalar functions
- `duckdb/function/aggregate_function.hpp` - Aggregate functions
- `duckdb/function/table_function.hpp` - Table functions
- `duckdb/function/copy_function.hpp` - Copy functions

### Getting Help

- **GitHub Discussions**: https://github.com/duckdb/duckdb/discussions
- **Discord**: https://discord.duckdb.org
- **GitHub Issues**: https://github.com/duckdb/duckdb/issues

### Best Practices Summary

1. **Use SQLLogicTest** for all testing
2. **Keep extensions small** and focused
3. **Document all functions** in README
4. **Test error conditions** thoroughly
5. **Use debug builds** during development
6. **Format code** with `make format-fix` before committing
7. **Keep up with DuckDB releases** and update promptly
8. **Submit to community extensions** for maximum reach
9. **Follow C++ guidelines** from main CLAUDE.md
10. **Write descriptive commit messages**

---

This guide provides a complete reference for DuckDB extension development. For additional details on specific topics, consult the DuckDB source code and existing extensions as examples.
