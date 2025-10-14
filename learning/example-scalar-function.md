# Adding a Scalar Function to DuckDB: Complete Walkthrough

This guide provides a complete, step-by-step walkthrough of adding a new scalar function to DuckDB. We'll implement a simple `reverse_string` function that reverses the characters in a string.

## Table of Contents

1. [Goal: What We're Building](#goal-what-were-building)
2. [Step 1: Find the Right Location](#step-1-find-the-right-location)
3. [Step 2: Write the Function Implementation](#step-2-write-the-function-implementation)
4. [Step 3: Register the Function](#step-3-register-the-function)
5. [Step 4: Build and Test Interactively](#step-4-build-and-test-interactively)
6. [Step 5: Write SQL Logic Tests](#step-5-write-sql-logic-tests)
7. [Step 6: Write C++ Unit Tests (Optional)](#step-6-write-c-unit-tests-optional)
8. [Step 7: Handle Edge Cases and Errors](#step-7-handle-edge-cases-and-errors)
9. [Complete Code Listing](#complete-code-listing)
10. [Advanced Topics](#advanced-topics)

---

## Goal: What We're Building

We'll implement a `reverse_string()` function that reverses the characters in a string:

```sql
SELECT reverse_string('hello');  -- returns 'olleh'
SELECT reverse_string('DuckDB'); -- returns 'BDkcuD'
```

This function will:
- Take a single VARCHAR argument
- Return a VARCHAR result
- Handle NULL values (NULL input produces NULL output)
- Work correctly with Unicode characters
- Be vectorized for performance

---

## Step 1: Find the Right Location

DuckDB has scalar functions in two main locations:

1. **Core functions** (in `src/function/scalar/`): Basic built-in functions
2. **Extension functions** (in `extension/core_functions/scalar/`): Extended functionality

For string functions, we'll add our implementation to:
```
extension/core_functions/scalar/string/
```

This is where modern string functions like `starts_with`, `levenshtein`, and `bar` are located.

### Understanding the Architecture

Scalar functions in DuckDB follow this pattern:
- **Implementation file** (`.cpp`): Contains the actual function logic
- **Header file** (`.hpp`): Contains function metadata (auto-generated)
- **Registration**: Happens automatically via function list

---

## Step 2: Write the Function Implementation

Create a new file: `extension/core_functions/scalar/string/reverse_string.cpp`

```cpp
#include "core_functions/scalar/string_functions.hpp"
#include "duckdb/common/exception.hpp"
#include "duckdb/common/vector_operations/vector_operations.hpp"

namespace duckdb {

// The actual reverse logic - reverses a string in-place
static void ReverseString(const char *input_data, idx_t input_length, char *output_data) {
	// Simple byte-level reverse
	// Note: This works for ASCII. For proper Unicode support, you'd need to reverse
	// by codepoints or grapheme clusters
	for (idx_t i = 0; i < input_length; i++) {
		output_data[input_length - 1 - i] = input_data[i];
	}
}

// Operator struct that implements the function logic
struct ReverseStringOperator {
	template <class INPUT_TYPE, class RESULT_TYPE>
	static RESULT_TYPE Operation(INPUT_TYPE input, Vector &result) {
		auto input_data = input.GetData();
		auto input_length = input.GetSize();

		// Allocate space for the result string
		auto result_str = StringVector::EmptyString(result, input_length);
		auto result_data = result_str.GetDataWriteable();

		// Perform the reverse operation
		ReverseString(input_data, input_length, result_data);

		// Finalize the string (important!)
		result_str.Finalize();
		return result_str;
	}
};

// GetFunction() is called to register this function with DuckDB
ScalarFunction ReverseStringFun::GetFunction() {
	return ScalarFunction(
		"reverse_string",                                    // function name
		{LogicalType::VARCHAR},                              // input types
		LogicalType::VARCHAR,                                // return type
		ScalarFunction::UnaryFunction<string_t, string_t, ReverseStringOperator> // executor
	);
}

} // namespace duckdb
```

### Key Components Explained

1. **ReverseString()**: The core logic that reverses the string
   - Takes input data pointer and length
   - Writes reversed bytes to output data pointer

2. **ReverseStringOperator**: Template struct that wraps the operation
   - `Operation()` method is called for each value
   - Must handle `string_t` types properly
   - Uses `StringVector::EmptyString()` to allocate result space
   - Must call `Finalize()` on result strings

3. **GetFunction()**: Returns a `ScalarFunction` object
   - Specifies function name, input types, return type
   - Uses `UnaryFunction` helper for single-argument functions
   - For multi-argument functions, use `BinaryFunction` or custom function pointer

---

## Step 3: Register the Function

### 3.1 Add Function Metadata to Header

Add the following to `extension/core_functions/include/core_functions/scalar/string_functions.hpp`:

```cpp
struct ReverseStringFun {
	static constexpr const char *Name = "reverse_string";
	static constexpr const char *Parameters = "string";
	static constexpr const char *Description = "Reverses the characters in the input string.";
	static constexpr const char *Example = "reverse_string('hello')";
	static constexpr const char *Categories = "string";

	static ScalarFunction GetFunction();
};
```

**Note**: In practice, this header file is auto-generated by `scripts/generate_functions.py`. You would typically add metadata to the generation script instead of editing the header directly.

### 3.2 Add to Function List

Add to `extension/core_functions/function_list.cpp`:

```cpp
static const StaticFunctionDefinition core_functions[] = {
	// ... existing functions ...
	DUCKDB_SCALAR_FUNCTION(ReverseStringFun),
	// ... more functions ...
	FINAL_FUNCTION
};
```

The `DUCKDB_SCALAR_FUNCTION` macro automatically wires up the function metadata and registration.

### 3.3 Update CMakeLists.txt

Add your new source file to `extension/core_functions/CMakeLists.txt`:

```cmake
set(EXTENSION_SOURCES
  # ... existing files ...
  scalar/string/reverse_string.cpp
  # ... more files ...
)
```

---

## Step 4: Build and Test Interactively

### 4.1 Build DuckDB

```bash
# From the repository root
cd /path/to/duckdb

# Build in debug mode for development
make debug

# Or use ninja for faster builds
GEN=ninja make debug
```

The build will take several minutes the first time. Subsequent builds are incremental.

### 4.2 Test Interactively

Launch the DuckDB shell:

```bash
./build/debug/duckdb
```

Test your function:

```sql
-- Basic test
SELECT reverse_string('hello');
-- Result: olleh

-- Test with mixed case
SELECT reverse_string('DuckDB');
-- Result: BDkcuD

-- Test with empty string
SELECT reverse_string('');
-- Result: (empty string)

-- Test with NULL
SELECT reverse_string(NULL);
-- Result: NULL

-- Test with numbers
SELECT reverse_string('12345');
-- Result: 54321

-- Test with special characters
SELECT reverse_string('Hello, World!');
-- Result: !dlroW ,olleH

-- Test in a table
CREATE TABLE strings(s VARCHAR);
INSERT INTO strings VALUES ('foo'), ('bar'), ('baz');
SELECT s, reverse_string(s) FROM strings;
```

If something doesn't work:
- Check for compilation errors in the build output
- Ensure all files are properly added to CMakeLists.txt
- Verify function is registered in function_list.cpp

---

## Step 5: Write SQL Logic Tests

Create a test file: `test/sql/function/string/test_reverse_string.test`

```sql
# name: test/sql/function/string/test_reverse_string.test
# description: Test reverse_string function
# group: [string]

# Enable verification (catches vectorization bugs)
statement ok
PRAGMA enable_verification

# Basic tests
query T
SELECT reverse_string('hello')
----
olleh

query T
SELECT reverse_string('DuckDB')
----
BDkcuD

query T
SELECT reverse_string('a')
----
a

# Empty string
query T
SELECT reverse_string('')
----
(empty)

# NULL handling
query T
SELECT reverse_string(NULL)
----
NULL

# Numeric characters
query T
SELECT reverse_string('12345')
----
54321

# Special characters
query T
SELECT reverse_string('Hello, World!')
----
!dlroW ,olleH

# Test with table
statement ok
CREATE TABLE test_strings(s VARCHAR)

statement ok
INSERT INTO test_strings VALUES ('foo'), ('bar'), ('baz'), (NULL)

query TT
SELECT s, reverse_string(s) FROM test_strings ORDER BY s NULLS FIRST
----
NULL	NULL
bar	rab
baz	zab
foo	oof

# Test with longer strings (non-inlined)
query T
SELECT reverse_string('abcdefghijklmnopqrstuvwxyz')
----
zyxwvutsrqponmlkjihgfedcba

# Test with Unicode characters
query T
SELECT reverse_string('café')
----
éfac

query T
SELECT reverse_string('你好世界')
----
界世好你

query T
SELECT reverse_string('🦆🦆🦆')
----
🦆🦆🦆

# Test double reverse returns original
query T
SELECT reverse_string(reverse_string('DuckDB'))
----
DuckDB

# Test in WHERE clause
query T
SELECT s FROM test_strings WHERE reverse_string(s) = 'oof'
----
foo

# Test in GROUP BY
statement ok
CREATE TABLE words(word VARCHAR)

statement ok
INSERT INTO words VALUES ('abc'), ('def'), ('fed'), ('cba')

query TI
SELECT reverse_string(word), COUNT(*) FROM words GROUP BY reverse_string(word) ORDER BY 1
----
abc	1
cba	1
def	1
fed	1
```

### Test File Format

- **Header comments**: Name, description, and group
- **statement ok**: Executes a statement that should succeed
- **query T**: Executes a query expecting text results
  - `T` = text, `I` = integer, `R` = real (float)
  - Multiple columns: `TI` for text and integer
- **----**: Separates query from expected results
- **Expected results**: One value per line

### Running Tests

```bash
# Run all tests
./build/debug/test/unittest

# Run specific test file
./build/debug/test/unittest test/sql/function/string/test_reverse_string.test

# Run all string tests
./build/debug/test/unittest "[string]"

# Run fast unit tests only
make unit
```

---

## Step 6: Write C++ Unit Tests (Optional)

For complex functions, you might want C++ unit tests. Create `test/sql/function/test_reverse_string.cpp`:

```cpp
#include "catch.hpp"
#include "duckdb/common/types/value.hpp"
#include "test_helpers.hpp"

using namespace duckdb;
using namespace std;

TEST_CASE("Test reverse_string function", "[string]") {
	unique_ptr<QueryResult> result;
	DuckDB db(nullptr);
	Connection con(db);

	// Basic functionality
	result = con.Query("SELECT reverse_string('hello')");
	REQUIRE(CHECK_COLUMN(result, 0, {"olleh"}));

	result = con.Query("SELECT reverse_string('DuckDB')");
	REQUIRE(CHECK_COLUMN(result, 0, {"BDkcuD"}));

	// Edge cases
	result = con.Query("SELECT reverse_string('')");
	REQUIRE(CHECK_COLUMN(result, 0, {""}));

	result = con.Query("SELECT reverse_string(NULL)");
	REQUIRE(CHECK_COLUMN(result, 0, {Value()}));

	// Longer strings
	result = con.Query("SELECT reverse_string('abcdefghijklmnopqrstuvwxyz')");
	REQUIRE(CHECK_COLUMN(result, 0, {"zyxwvutsrqponmlkjihgfedcba"}));

	// Unicode
	result = con.Query("SELECT reverse_string('café')");
	REQUIRE(CHECK_COLUMN(result, 0, {"éfac"}));
}

TEST_CASE("Test reverse_string with vectors", "[string]") {
	unique_ptr<QueryResult> result;
	DuckDB db(nullptr);
	Connection con(db);

	con.Query("CREATE TABLE test(s VARCHAR)");
	con.Query("INSERT INTO test VALUES ('foo'), ('bar'), ('baz'), (NULL)");

	result = con.Query("SELECT reverse_string(s) FROM test ORDER BY s NULLS FIRST");
	REQUIRE(CHECK_COLUMN(result, 0, {Value(), "rab", "zab", "oof"}));
}
```

### C++ Test Guidelines

- Use **Catch2** testing framework
- Include `test_helpers.hpp` for convenience macros
- Create `DuckDB` and `Connection` objects
- Use `CHECK_COLUMN` to verify results
- Test both scalar and vector execution
- Tag tests appropriately (`[string]`, `[.]` for slow tests)

### Running C++ Tests

```bash
# Run all C++ unit tests
./build/debug/test/unittest

# Run specific test
./build/debug/test/unittest "Test reverse_string function"

# Run all string tests
./build/debug/test/unittest "[string]"
```

---

## Step 7: Handle Edge Cases and Errors

### 7.1 NULL Handling

DuckDB automatically handles NULL values for most functions. If your function needs custom NULL handling:

```cpp
ScalarFunction ReverseStringFun::GetFunction() {
	auto func = ScalarFunction(
		"reverse_string",
		{LogicalType::VARCHAR},
		LogicalType::VARCHAR,
		ScalarFunction::UnaryFunction<string_t, string_t, ReverseStringOperator>
	);
	// Custom NULL handling if needed
	func.null_handling = FunctionNullHandling::SPECIAL_HANDLING;
	return func;
}
```

### 7.2 Error Handling

Throw exceptions for invalid inputs:

```cpp
struct ReverseStringOperator {
	template <class INPUT_TYPE, class RESULT_TYPE>
	static RESULT_TYPE Operation(INPUT_TYPE input, Vector &result) {
		auto input_length = input.GetSize();

		// Example: limit maximum string length
		if (input_length > 1000000) {
			throw InvalidInputException("String too long for reverse_string (max 1MB)");
		}

		// ... rest of implementation
	}
};
```

### 7.3 Unicode Support

Our simple implementation reverses bytes, not Unicode codepoints. For proper Unicode support:

```cpp
#include "utf8proc_wrapper.hpp"

static string_t ReverseUnicode(const string_t &input, Vector &result) {
	auto input_data = input.GetData();
	auto input_length = input.GetSize();

	// First pass: count codepoints and calculate output size
	vector<int> codepoint_sizes;
	idx_t pos = 0;
	while (pos < input_length) {
		int sz = 0;
		Utf8Proc::UTF8ToCodepoint(input_data + pos, sz);
		codepoint_sizes.push_back(sz);
		pos += sz;
	}

	// Allocate output string
	auto result_str = StringVector::EmptyString(result, input_length);
	auto result_data = result_str.GetDataWriteable();

	// Second pass: copy codepoints in reverse order
	idx_t output_pos = 0;
	for (int i = codepoint_sizes.size() - 1; i >= 0; i--) {
		idx_t input_pos = 0;
		for (int j = 0; j < i; j++) {
			input_pos += codepoint_sizes[j];
		}
		memcpy(result_data + output_pos, input_data + input_pos, codepoint_sizes[i]);
		output_pos += codepoint_sizes[i];
	}

	result_str.Finalize();
	return result_str;
}
```

### 7.4 Performance Considerations

- **Vectorization**: DuckDB processes data in vectors (batches). Your function is automatically vectorized using `UnaryExecutor`.
- **String allocation**: Use `StringVector::EmptyString()` for efficient allocation
- **Avoid copies**: Work directly with `string_t` data pointers when possible
- **Inlined strings**: Strings ≤12 bytes are inlined in `string_t`, no separate allocation

---

## Complete Code Listing

### reverse_string.cpp (Full Implementation)

```cpp
#include "core_functions/scalar/string_functions.hpp"
#include "duckdb/common/exception.hpp"
#include "duckdb/common/vector_operations/vector_operations.hpp"

namespace duckdb {

// Simple byte-level reverse
static void ReverseString(const char *input_data, idx_t input_length, char *output_data) {
	for (idx_t i = 0; i < input_length; i++) {
		output_data[input_length - 1 - i] = input_data[i];
	}
}

struct ReverseStringOperator {
	template <class INPUT_TYPE, class RESULT_TYPE>
	static RESULT_TYPE Operation(INPUT_TYPE input, Vector &result) {
		auto input_data = input.GetData();
		auto input_length = input.GetSize();

		// Allocate result string
		auto result_str = StringVector::EmptyString(result, input_length);
		auto result_data = result_str.GetDataWriteable();

		// Reverse the string
		ReverseString(input_data, input_length, result_data);

		// Finalize and return
		result_str.Finalize();
		return result_str;
	}
};

ScalarFunction ReverseStringFun::GetFunction() {
	return ScalarFunction(
		"reverse_string",
		{LogicalType::VARCHAR},
		LogicalType::VARCHAR,
		ScalarFunction::UnaryFunction<string_t, string_t, ReverseStringOperator>
	);
}

} // namespace duckdb
```

### Header Declaration (in string_functions.hpp)

```cpp
struct ReverseStringFun {
	static constexpr const char *Name = "reverse_string";
	static constexpr const char *Parameters = "string";
	static constexpr const char *Description = "Reverses the characters in the input string.";
	static constexpr const char *Example = "reverse_string('hello')";
	static constexpr const char *Categories = "string";

	static ScalarFunction GetFunction();
};
```

### Registration (in function_list.cpp)

```cpp
static const StaticFunctionDefinition core_functions[] = {
	// ... other functions ...
	DUCKDB_SCALAR_FUNCTION(ReverseStringFun),
	// ... more functions ...
	FINAL_FUNCTION
};
```

---

## Advanced Topics

### Multiple Overloads (Function Sets)

If your function supports multiple argument types, use `ScalarFunctionSet`:

```cpp
ScalarFunctionSet MyFunctionFun::GetFunctions() {
	ScalarFunctionSet set("my_function");

	// Overload for VARCHAR
	set.AddFunction(ScalarFunction(
		{LogicalType::VARCHAR},
		LogicalType::VARCHAR,
		VarcharImplementation
	));

	// Overload for INTEGER
	set.AddFunction(ScalarFunction(
		{LogicalType::INTEGER},
		LogicalType::VARCHAR,
		IntegerImplementation
	));

	return set;
}
```

### Custom Bind Function

For functions that need compile-time analysis:

```cpp
unique_ptr<FunctionData> MyBind(ClientContext &context, ScalarFunction &bound_function,
                                 vector<unique_ptr<Expression>> &arguments) {
	// Analyze arguments at bind time
	if (arguments[0]->HasParameter()) {
		throw ParameterNotResolvedException();
	}

	// Can modify bound_function here
	// Can return custom FunctionData
	return nullptr;
}

ScalarFunction MyFun::GetFunction() {
	return ScalarFunction(
		"my_function",
		{LogicalType::VARCHAR},
		LogicalType::VARCHAR,
		MyImplementation,
		MyBind  // Bind function
	);
}
```

### Statistics Propagation

Optimize query planning with statistics:

```cpp
unique_ptr<BaseStatistics> MyStatsPropagation(ClientContext &context,
                                               FunctionStatisticsInput &input) {
	auto &child_stats = input.child_stats;
	// Analyze input statistics and return output statistics
	return nullptr;
}

ScalarFunction MyFun::GetFunction() {
	return ScalarFunction(
		"my_function",
		{LogicalType::VARCHAR},
		LogicalType::VARCHAR,
		MyImplementation,
		nullptr,  // bind
		nullptr,  // dependency
		MyStatsPropagation  // statistics
	);
}
```

### Binary Functions

For two-argument functions:

```cpp
struct ConcatOperator {
	template <class TA, class TB, class TR>
	static inline TR Operation(TA left, TB right, Vector &result) {
		// Implementation with two arguments
	}
};

ScalarFunction::BinaryFunction<string_t, string_t, string_t, ConcatOperator>
```

### Variadic Functions

For variable number of arguments:

```cpp
ScalarFunction MyFun::GetFunction() {
	auto func = ScalarFunction(
		"my_function",
		{LogicalType::ANY},  // base arguments
		LogicalType::VARCHAR,
		MyImplementation
	);
	func.varargs = LogicalType::ANY;  // allow additional arguments
	return func;
}
```

### Custom Vectorized Implementation

For maximum control:

```cpp
void MyCustomFunction(DataChunk &args, ExpressionState &state, Vector &result) {
	auto &input = args.data[0];
	auto count = args.size();

	// Custom vectorized logic
	UnifiedVectorFormat vdata;
	input.ToUnifiedFormat(count, vdata);

	auto input_data = UnifiedVectorFormat::GetData<string_t>(vdata);
	auto result_data = FlatVector::GetData<string_t>(result);

	for (idx_t i = 0; i < count; i++) {
		auto idx = vdata.sel->get_index(i);
		if (vdata.validity.RowIsValid(idx)) {
			result_data[i] = MyOperation(input_data[idx], result);
		} else {
			FlatVector::SetNull(result, i, true);
		}
	}
}
```

---

## Summary

Adding a scalar function to DuckDB involves:

1. **Create implementation file** in appropriate directory
2. **Define operator struct** with Operation() method
3. **Implement GetFunction()** to return ScalarFunction
4. **Register in header** with metadata (or via generation script)
5. **Add to function list** for automatic registration
6. **Build and test** interactively
7. **Write SQL logic tests** (strongly preferred)
8. **Optionally write C++ tests** for complex cases
9. **Handle edge cases**: NULL, errors, Unicode
10. **Format code** with `make format-fix`

The key insight is that DuckDB's vectorized execution handles most of the complexity. You just need to:
- Implement the core operation logic
- Wrap it in an operator struct
- Register it properly

For more examples, explore:
- `extension/core_functions/scalar/string/` - Modern string functions
- `src/function/scalar/string/` - Core string functions
- Test files in `test/sql/function/string/`

Happy coding!
