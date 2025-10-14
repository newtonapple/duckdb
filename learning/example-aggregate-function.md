# Complete Walkthrough: Adding an Aggregate Function to DuckDB

This guide provides a comprehensive, step-by-step walkthrough for adding a custom aggregate function to DuckDB. We'll implement a simple but useful function: **GEOMETRIC_MEAN**, which computes the geometric mean of a set of positive numbers.

## Table of Contents
1. [Goal: What We're Building](#goal-what-were-building)
2. [Understanding Aggregate Function Structure](#understanding-aggregate-function-structure)
3. [Step 1: Find the Right Location](#step-1-find-the-right-location)
4. [Step 2: Define the State Structure](#step-2-define-the-state-structure)
5. [Step 3: Implement Update Function](#step-3-implement-update-function)
6. [Step 4: Implement Combine Function](#step-4-implement-combine-function-for-parallelism)
7. [Step 5: Implement Finalize Function](#step-5-implement-finalize-function)
8. [Step 6: Register the Function](#step-6-register-the-function)
9. [Step 7: Build and Test Interactively](#step-7-build-and-test-interactively)
10. [Step 8: Write Comprehensive Tests](#step-8-write-comprehensive-tests)
11. [Complete Code Listing](#complete-code-listing)

---

## Goal: What We're Building

We'll implement `GEOMETRIC_MEAN(x)`, which computes the geometric mean of positive numbers:

```
geometric_mean = (x1 * x2 * ... * xn)^(1/n)
```

To avoid overflow with large products, we'll use the logarithmic formula:

```
geometric_mean = exp((log(x1) + log(x2) + ... + log(xn)) / n)
```

**Example usage:**
```sql
SELECT GEOMETRIC_MEAN(value) FROM numbers;
-- Input: 2, 8, 32
-- Output: 8.0  (because 2^(1/3) * 8^(1/3) * 32^(1/3) = 8)
```

---

## Understanding Aggregate Function Structure

Aggregate functions in DuckDB process data in four key phases:

### 1. State
A struct that holds intermediate computation results. Each group in a `GROUP BY` gets its own state.

### 2. Update (Accumulation)
Called for each input row to accumulate data into the state. This is where the main computation happens.

### 3. Combine (Parallel Merging)
Merges two states together. Essential for parallel execution where different threads process different chunks of data.

### 4. Finalize (Result Production)
Converts the accumulated state into the final result value.

### Types of Aggregates

DuckDB categorizes aggregates into three types (see `extension/core_functions/aggregate/README.md`):

- **Distributive**: Can be computed in parallel and combined (e.g., SUM, COUNT, MIN, MAX)
- **Algebraic**: Computed from multiple distributive aggregates (e.g., AVG = SUM/COUNT)
- **Holistic**: Require all data to compute (e.g., MEDIAN, MODE)

Our GEOMETRIC_MEAN is **algebraic** because it's computed as `exp(sum_log / count)`.

---

## Step 1: Find the Right Location

Aggregate functions are organized in `extension/core_functions/aggregate/`:

```
extension/core_functions/aggregate/
├── distributive/     # SUM, COUNT, MIN, MAX, PRODUCT, etc.
├── algebraic/        # AVG, STDDEV, COVAR, etc.
├── holistic/         # MEDIAN, MODE, QUANTILE, etc.
├── nested/           # LIST, HISTOGRAM, etc.
└── regression/       # Linear regression functions
```

Since GEOMETRIC_MEAN is algebraic (similar to AVG), we'll place it in:

```
extension/core_functions/aggregate/algebraic/geometric_mean.cpp
```

**Key files to know:**
- `src/include/duckdb/function/aggregate_function.hpp` - Core aggregate function interface
- `extension/core_functions/include/core_functions/aggregate/algebraic_functions.hpp` - Function declarations
- `extension/core_functions/aggregate/README.md` - Detailed aggregate documentation

---

## Step 2: Define the State Structure

The state holds all data needed during accumulation. For geometric mean using logarithms:

```cpp
template <class T>
struct GeometricMeanState {
    uint64_t count;    // Number of values processed
    T sum_log;         // Sum of logarithms

    void Initialize() {
        this->count = 0;
        this->sum_log = 0;
    }

    void Combine(const GeometricMeanState<T> &other) {
        this->count += other.count;
        this->sum_log += other.sum_log;
    }
};
```

**Key points:**
- Template on type `T` to support different numeric types (float, double)
- `Initialize()` resets state to empty
- `Combine()` merges another state into this one (for parallel execution)
- Keep state simple and small for efficiency

---

## Step 3: Implement Update Function

The update function processes input values and accumulates them into the state.

### 3.1 Create the Operation Class

DuckDB uses a template-based approach where you define an "Operation" class with static methods:

```cpp
struct GeometricMeanOperation {
    template <class STATE>
    static void Initialize(STATE &state) {
        state.Initialize();
    }

    template <class INPUT_TYPE, class STATE, class OP>
    static void Operation(STATE &state, const INPUT_TYPE &input, AggregateUnaryInput &) {
        if (input <= 0) {
            throw InvalidInputException("GEOMETRIC_MEAN requires positive values");
        }
        state.count += 1;
        state.sum_log += std::log(static_cast<double>(input));
    }

    template <class INPUT_TYPE, class STATE, class OP>
    static void ConstantOperation(STATE &state, const INPUT_TYPE &input,
                                  AggregateUnaryInput &, idx_t count) {
        if (input <= 0) {
            throw InvalidInputException("GEOMETRIC_MEAN requires positive values");
        }
        state.count += count;
        state.sum_log += std::log(static_cast<double>(input)) * count;
    }

    template <class STATE, class OP>
    static void Combine(const STATE &source, STATE &target, AggregateInputData &) {
        target.Combine(source);
    }

    static bool IgnoreNull() {
        return true;  // Skip NULL values
    }
};
```

**Key points:**
- `Operation()` - processes a single input value
- `ConstantOperation()` - optimized path when the same value appears multiple times
- `IgnoreNull()` - return true to skip NULL values automatically
- Validate input (geometric mean requires positive numbers)

---

## Step 4: Implement Combine Function (for Parallelism)

The combine function is already handled by our `Combine()` method in the state and operation class.

**Why this matters:**
- DuckDB processes data in parallel across multiple threads
- Each thread maintains its own state
- After processing, states must be merged
- Without combine, aggregation would be single-threaded

```cpp
void Combine(const GeometricMeanState<T> &other) {
    this->count += other.count;
    this->sum_log += other.sum_log;
}
```

This simple addition works because:
1. Sum of logs is associative: `(log(a) + log(b)) + log(c) = log(a) + (log(b) + log(c))`
2. Count is distributive: `count(A) + count(B) = count(A ∪ B)`

---

## Step 5: Implement Finalize Function

The finalize function converts the accumulated state into the final result:

```cpp
template <class T, class STATE>
static void Finalize(STATE &state, T &target, AggregateFinalizeData &finalize_data) {
    if (state.count == 0) {
        finalize_data.ReturnNull();
        return;
    }
    // geometric_mean = exp(sum_log / count)
    double average_log = state.sum_log / state.count;
    target = std::exp(average_log);
}
```

**Key points:**
- Return NULL for empty input (standard SQL behavior)
- Convert sum of logs back to geometric mean using `exp()`
- Cast result to appropriate output type

---

## Step 6: Register the Function

### 6.1 Create the Aggregate Function

Use the helper method `UnaryAggregate` to create the function:

```cpp
AggregateFunction GetGeometricMeanFunction(const LogicalType &type) {
    switch (type.InternalType()) {
    case PhysicalType::DOUBLE:
        return AggregateFunction::UnaryAggregate<GeometricMeanState<double>, double, double,
                                                 GeometricMeanOperation>(
            LogicalType::DOUBLE, LogicalType::DOUBLE);
    case PhysicalType::FLOAT:
        return AggregateFunction::UnaryAggregate<GeometricMeanState<double>, float, double,
                                                 GeometricMeanOperation>(
            LogicalType::FLOAT, LogicalType::DOUBLE);
    default:
        throw NotImplementedException("GEOMETRIC_MEAN only supports FLOAT and DOUBLE");
    }
}
```

### 6.2 Create Function Set and Registration

```cpp
namespace duckdb {

AggregateFunctionSet GeometricMeanFun::GetFunctions() {
    AggregateFunctionSet geometric_mean("geometric_mean");
    geometric_mean.AddFunction(GetGeometricMeanFunction(LogicalType::FLOAT));
    geometric_mean.AddFunction(GetGeometricMeanFunction(LogicalType::DOUBLE));
    return geometric_mean;
}

} // namespace duckdb
```

### 6.3 Add to Header File

Add declaration to `extension/core_functions/include/core_functions/aggregate/algebraic_functions.hpp`:

```cpp
struct GeometricMeanFun {
    static constexpr const char *Name = "geometric_mean";
    static constexpr const char *Parameters = "x";
    static constexpr const char *Description = "Computes the geometric mean of positive values.";
    static constexpr const char *Example = "geometric_mean(A)";
    static constexpr const char *Categories = "";

    static AggregateFunctionSet GetFunctions();
};
```

### 6.4 Update CMakeLists.txt

Add the new file to `extension/core_functions/aggregate/algebraic/CMakeLists.txt`:

```cmake
add_library(
  core_functions_aggregate_algebraic OBJECT
  avg.cpp
  covar.cpp
  corr.cpp
  stddev.cpp
  geometric_mean.cpp  # Add this line
)
```

### 6.5 Register with Core Functions

The function registration happens automatically through the function list generation system. You may need to run:

```bash
python3 scripts/generate_functions.py
```

---

## Step 7: Build and Test Interactively

### 7.1 Build DuckDB

```bash
# Debug build for development
make debug

# Or with ninja for faster builds
GEN=ninja make debug
```

### 7.2 Interactive Testing

Launch the DuckDB CLI and test your function:

```bash
./build/debug/duckdb

D SELECT GEOMETRIC_MEAN(value) FROM (VALUES (2.0), (8.0), (32.0)) AS t(value);
┌────────────────────────┐
│ geometric_mean(value)  │
│        double          │
├────────────────────────┤
│                    8.0 │
└────────────────────────┘

D SELECT GEOMETRIC_MEAN(value) FROM (VALUES (1.0), (2.0), (4.0), (8.0)) AS t(value);
┌────────────────────────┐
│ geometric_mean(value)  │
│        double          │
├────────────────────────┤
│     2.8284271247461903 │
└────────────────────────┘

D SELECT GEOMETRIC_MEAN(value) FROM (VALUES (NULL)) AS t(value);
┌────────────────────────┐
│ geometric_mean(value)  │
│        double          │
├────────────────────────┤
│                   NULL │
└────────────────────────┘

D SELECT region, GEOMETRIC_MEAN(price)
  FROM products
  GROUP BY region;
```

### 7.3 Test Error Cases

```sql
-- Should throw error (negative values)
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (-1.0)) AS t(value);

-- Should throw error (zero)
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (0.0)) AS t(value);

-- Should handle empty table
SELECT GEOMETRIC_MEAN(value) FROM (SELECT 1.0 AS value WHERE FALSE) AS t;
```

---

## Step 8: Write Comprehensive Tests

Create `test/sql/aggregate/aggregates/test_geometric_mean.test`:

```sql
# name: test/sql/aggregate/aggregates/test_geometric_mean.test
# description: Test GEOMETRIC_MEAN aggregate function
# group: [aggregates]

# Test basic functionality
query R
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (2.0), (8.0), (32.0)) AS t(value);
----
8.0

# Test with different values
query R
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (1.0), (2.0), (4.0), (8.0)) AS t(value);
----
2.8284271247461903

# Test single value
query R
SELECT GEOMETRIC_MEAN(5.0);
----
5.0

# Test with NULLs (should ignore them)
query R
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (4.0), (NULL), (16.0)) AS t(value);
----
8.0

# Test all NULLs (should return NULL)
query R
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (NULL), (NULL)) AS t(value);
----
NULL

# Test empty result set (should return NULL)
query R
SELECT GEOMETRIC_MEAN(value) FROM (SELECT 1.0 AS value WHERE FALSE) AS t;
----
NULL

# Test with GROUP BY
statement ok
CREATE TABLE products(region VARCHAR, price DOUBLE);

statement ok
INSERT INTO products VALUES
    ('North', 10.0), ('North', 20.0), ('North', 40.0),
    ('South', 5.0), ('South', 15.0), ('South', 45.0);

query TR
SELECT region, GEOMETRIC_MEAN(price) FROM products GROUP BY region ORDER BY region;
----
North	20.0
South	15.0

# Test FLOAT type
query R
SELECT GEOMETRIC_MEAN(value::FLOAT) FROM (VALUES (2), (8), (32)) AS t(value);
----
8.0

# Test with DISTINCT
query R
SELECT GEOMETRIC_MEAN(DISTINCT value) FROM (VALUES (2.0), (2.0), (8.0), (32.0)) AS t(value);
----
8.0

# Test negative values (should error)
statement error
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (-5.0)) AS t(value);
----
GEOMETRIC_MEAN requires positive values

# Test zero (should error)
statement error
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (0.0)) AS t(value);
----
GEOMETRIC_MEAN requires positive values

# Test mixed positive and zero (should error)
statement error
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (1.0), (0.0), (3.0)) AS t(value);
----
GEOMETRIC_MEAN requires positive values

# Test with WHERE clause
query R
SELECT GEOMETRIC_MEAN(price) FROM products WHERE region = 'North';
----
20.0

# Test precision with small numbers
query R
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (0.1), (0.01), (0.001)) AS t(value);
----
0.01

# Test precision with large numbers
query R
SELECT GEOMETRIC_MEAN(value) FROM (VALUES (1000.0), (10000.0), (100000.0)) AS t(value);
----
10000.0

# Test window function (if supported)
query RR
SELECT price,
       GEOMETRIC_MEAN(price) OVER (PARTITION BY region) as geom_mean
FROM products
WHERE region = 'North'
ORDER BY price;
----
10.0	20.0
20.0	20.0
40.0	20.0

# Cleanup
statement ok
DROP TABLE products;
```

### Run Tests

```bash
# Run just your test
./build/debug/test/unittest "test_geometric_mean"

# Run all aggregate tests
./build/debug/test/unittest "[aggregates]"

# Run the test file with sqllogictest
python3 scripts/run_tests.py test/sql/aggregate/aggregates/test_geometric_mean.test
```

### Test Coverage

Ensure your tests cover:
1. Basic functionality with simple inputs
2. Edge cases (single value, empty set, all NULLs)
3. NULL handling (mixed with valid values)
4. GROUP BY scenarios
5. Different data types (FLOAT, DOUBLE)
6. Error conditions (negative numbers, zero)
7. DISTINCT modifier
8. WHERE clause filtering
9. Precision with very small and very large numbers
10. Window functions (if applicable)

---

## Complete Code Listing

### File: `extension/core_functions/aggregate/algebraic/geometric_mean.cpp`

```cpp
#include "core_functions/aggregate/algebraic_functions.hpp"
#include "duckdb/common/exception.hpp"
#include "duckdb/common/types/hugeint.hpp"
#include "duckdb/function/function_set.hpp"
#include "duckdb/planner/expression.hpp"
#include <cmath>

namespace duckdb {

namespace {

template <class T>
struct GeometricMeanState {
	uint64_t count;
	T sum_log;

	void Initialize() {
		this->count = 0;
		this->sum_log = 0;
	}

	void Combine(const GeometricMeanState<T> &other) {
		this->count += other.count;
		this->sum_log += other.sum_log;
	}
};

struct GeometricMeanOperation {
	template <class STATE>
	static void Initialize(STATE &state) {
		state.Initialize();
	}

	template <class INPUT_TYPE, class STATE, class OP>
	static void Operation(STATE &state, const INPUT_TYPE &input, AggregateUnaryInput &) {
		if (input <= 0) {
			throw InvalidInputException("GEOMETRIC_MEAN requires positive values");
		}
		state.count += 1;
		state.sum_log += std::log(static_cast<double>(input));
	}

	template <class INPUT_TYPE, class STATE, class OP>
	static void ConstantOperation(STATE &state, const INPUT_TYPE &input,
	                              AggregateUnaryInput &, idx_t count) {
		if (input <= 0) {
			throw InvalidInputException("GEOMETRIC_MEAN requires positive values");
		}
		state.count += count;
		state.sum_log += std::log(static_cast<double>(input)) * count;
	}

	template <class STATE, class OP>
	static void Combine(const STATE &source, STATE &target, AggregateInputData &) {
		target.Combine(source);
	}

	template <class T, class STATE>
	static void Finalize(STATE &state, T &target, AggregateFinalizeData &finalize_data) {
		if (state.count == 0) {
			finalize_data.ReturnNull();
			return;
		}
		// geometric_mean = exp(sum_log / count)
		double average_log = state.sum_log / state.count;
		target = std::exp(average_log);
	}

	static bool IgnoreNull() {
		return true;
	}
};

AggregateFunction GetGeometricMeanFunction(const LogicalType &type) {
	switch (type.InternalType()) {
	case PhysicalType::FLOAT: {
		auto func = AggregateFunction::UnaryAggregate<GeometricMeanState<double>, float, double,
		                                               GeometricMeanOperation>(
		    LogicalType::FLOAT, LogicalType::DOUBLE);
		func.order_dependent = AggregateOrderDependent::NOT_ORDER_DEPENDENT;
		return func;
	}
	case PhysicalType::DOUBLE: {
		auto func = AggregateFunction::UnaryAggregate<GeometricMeanState<double>, double, double,
		                                               GeometricMeanOperation>(
		    LogicalType::DOUBLE, LogicalType::DOUBLE);
		func.order_dependent = AggregateOrderDependent::NOT_ORDER_DEPENDENT;
		return func;
	}
	default:
		throw NotImplementedException("GEOMETRIC_MEAN only supports FLOAT and DOUBLE types");
	}
}

} // namespace

AggregateFunctionSet GeometricMeanFun::GetFunctions() {
	AggregateFunctionSet geometric_mean;
	geometric_mean.AddFunction(GetGeometricMeanFunction(LogicalType::FLOAT));
	geometric_mean.AddFunction(GetGeometricMeanFunction(LogicalType::DOUBLE));
	return geometric_mean;
}

} // namespace duckdb
```

### File: Add to `extension/core_functions/include/core_functions/aggregate/algebraic_functions.hpp`

```cpp
struct GeometricMeanFun {
	static constexpr const char *Name = "geometric_mean";
	static constexpr const char *Parameters = "x";
	static constexpr const char *Description = "Computes the geometric mean of positive values using logarithmic computation.";
	static constexpr const char *Example = "geometric_mean(A)";
	static constexpr const char *Categories = "";

	static AggregateFunctionSet GetFunctions();
};
```

### File: Update `extension/core_functions/aggregate/algebraic/CMakeLists.txt`

```cmake
add_library(
  core_functions_aggregate_algebraic OBJECT
  avg.cpp
  covar.cpp
  corr.cpp
  stddev.cpp
  geometric_mean.cpp
)
```

---

## Key Takeaways

### Design Principles

1. **State Design**: Keep state small and efficient. Our state only has `count` and `sum_log`.

2. **Null Handling**: Use `IgnoreNull()` to skip NULLs automatically. Return NULL for empty sets.

3. **Type Support**: Support common numeric types (FLOAT, DOUBLE). Use templates for code reuse.

4. **Parallelism**: Implement `Combine()` for multi-threaded execution. Essential for large datasets.

5. **Validation**: Check input constraints early. Geometric mean requires positive values.

6. **Numerical Stability**: Use logarithms to avoid overflow with large products.

### Common Patterns

**Distributive aggregates** (SUM, COUNT, MIN, MAX):
- State is simple (single value or accumulator)
- Combine is straightforward merging
- Order doesn't matter

**Algebraic aggregates** (AVG, STDDEV):
- State holds multiple values (e.g., sum and count)
- Final computation in Finalize
- Compose from distributive pieces

**Holistic aggregates** (MEDIAN, MODE):
- State holds all or sampled data
- More complex combine logic
- May require sorting or hash tables

### Testing Strategy

1. **Correctness**: Verify mathematical correctness with known inputs
2. **Edge Cases**: Empty sets, single values, all NULLs
3. **Data Types**: Test all supported types
4. **Grouping**: Test with GROUP BY and PARTITION BY
5. **Errors**: Validate error conditions
6. **Performance**: Test with large datasets (not in regular tests)

### Further Reading

- `extension/core_functions/aggregate/README.md` - Detailed aggregate documentation
- `src/include/duckdb/function/aggregate_function.hpp` - Core interfaces
- Existing implementations in `extension/core_functions/aggregate/` - Real examples
- DuckDB documentation on custom functions

### Next Steps

After implementing your aggregate function:

1. Format code: `make format-fix`
2. Run tests: `make unit`
3. Build release: `make release`
4. Test performance with real data
5. Consider adding function aliases if needed
6. Update documentation

---

## Variations and Extensions

### Supporting Integer Types

To support integers, add automatic casting to double:

```cpp
case PhysicalType::INT32: {
    auto func = AggregateFunction::UnaryAggregate<GeometricMeanState<double>, int32_t, double,
                                                   GeometricMeanOperation>(
        LogicalType::INTEGER, LogicalType::DOUBLE);
    func.order_dependent = AggregateOrderDependent::NOT_ORDER_DEPENDENT;
    return func;
}
```

### Adding a Bind Function

For more complex aggregates, you might need custom binding:

```cpp
unique_ptr<FunctionData> BindGeometricMean(ClientContext &context, AggregateFunction &function,
                                           vector<unique_ptr<Expression>> &arguments) {
    // Custom validation or configuration
    auto &input_type = arguments[0]->return_type;
    if (!input_type.IsNumeric()) {
        throw BinderException("GEOMETRIC_MEAN requires numeric input");
    }
    function = GetGeometricMeanFunction(input_type);
    function.name = "geometric_mean";
    return nullptr;
}
```

### Window Function Support

Our implementation automatically works with window functions because we implement all required operations. Test with:

```sql
SELECT value,
       GEOMETRIC_MEAN(value) OVER (ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM data;
```

This guide provides everything needed to understand, implement, test, and debug aggregate functions in DuckDB. The patterns shown here apply to most aggregate functions you might want to add.
