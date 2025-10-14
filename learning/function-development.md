# DuckDB Function Development Guide

This guide provides a comprehensive overview of developing SQL functions in DuckDB for developers who want to add new built-in functions or understand the function architecture.

## Table of Contents
1. [Function Types Overview](#function-types-overview)
2. [Step-by-Step: Adding a Scalar Function](#step-by-step-adding-a-scalar-function)
3. [Step-by-Step: Adding an Aggregate Function](#step-by-step-adding-an-aggregate-function)
4. [Step-by-Step: Adding a Table Function](#step-by-step-adding-a-table-function)
5. [Step-by-Step: Adding a Window Function](#step-by-step-adding-a-window-function)
6. [Function Binding and Overloading](#function-binding-and-overloading)
7. [NULL Handling in Functions](#null-handling-in-functions)
8. [Vectorized Function Implementation](#vectorized-function-implementation)
9. [Function Statistics](#function-statistics)
10. [Testing Functions](#testing-functions)
11. [Documentation Requirements](#documentation-requirements)
12. [Common Patterns and Best Practices](#common-patterns-and-best-practices)

---

## Function Types Overview

DuckDB supports four main types of functions:

### 1. Scalar Functions
- **Purpose**: Transform individual values (row-by-row operations)
- **Examples**: `upper()`, `length()`, `abs()`, `+`, `-`
- **Input**: One or more values from a single row
- **Output**: Single value per row
- **Header**: `/src/include/duckdb/function/scalar_function.hpp`

### 2. Aggregate Functions
- **Purpose**: Combine multiple rows into a single result
- **Examples**: `count()`, `sum()`, `avg()`, `min()`, `max()`
- **Input**: Multiple rows
- **Output**: Single value per group
- **Requires**: State management for accumulation
- **Header**: `/src/include/duckdb/function/aggregate_function.hpp`

### 3. Table Functions
- **Purpose**: Generate or transform entire tables
- **Examples**: `range()`, `read_csv()`, `unnest()`
- **Input**: Parameters (values, files, etc.)
- **Output**: Entire table (multiple rows and columns)
- **Header**: `/src/include/duckdb/function/table_function.hpp`

### 4. Window Functions
- **Purpose**: Compute values over a window of rows
- **Examples**: `row_number()`, `rank()`, `lag()`, `lead()`
- **Input**: Window of rows with partitioning and ordering
- **Output**: One value per row in the window
- **Note**: Often implemented as special aggregate functions
- **Header**: `/src/include/duckdb/function/window/`

---

## Step-by-Step: Adding a Scalar Function

Scalar functions transform individual values. Here's how to add one:

### Example: Simple Unary Function

Let's implement a function `double_value(x)` that doubles a number.

#### Step 1: Create the Operator Struct

Create a file: `/src/function/scalar/math/double_value.cpp`

```cpp
#include "duckdb/function/scalar/math_functions.hpp"
#include "duckdb/common/vector_operations/vector_operations.hpp"

namespace duckdb {

// Define the operation as a template struct
struct DoubleOperator {
    template <class TA, class TR>
    static inline TR Operation(TA input) {
        return input * 2;
    }
};

} // namespace duckdb
```

#### Step 2: Register the Function

Add the registration function in the same file:

```cpp
namespace duckdb {

ScalarFunction DoubleValueFun::GetFunction() {
    return ScalarFunction("double_value",
                         {LogicalType::INTEGER},  // Input type(s)
                         LogicalType::INTEGER,     // Return type
                         ScalarFunction::UnaryFunction<int32_t, int32_t, DoubleOperator>);
}

} // namespace duckdb
```

#### Step 3: Add to Function List

In `/src/include/duckdb/function/scalar/math_functions.hpp`:

```cpp
struct DoubleValueFun {
    static ScalarFunction GetFunction();
};
```

Register in the appropriate registration file (e.g., `/extension/core_functions/scalar/math/math_functions.cpp`).

### Example: Binary Function with Multiple Types

For more complex functions that work with multiple types:

```cpp
struct AddOperator {
    template <class TA, class TB, class TR>
    static inline TR Operation(TA left, TB right) {
        return left + right;
    }
};

ScalarFunctionSet AddFun::GetFunctions() {
    ScalarFunctionSet add("+");

    // Add overload for each type combination
    add.AddFunction(ScalarFunction({LogicalType::INTEGER, LogicalType::INTEGER},
                                   LogicalType::INTEGER,
                                   ScalarFunction::BinaryFunction<int32_t, int32_t, int32_t, AddOperator>));

    add.AddFunction(ScalarFunction({LogicalType::BIGINT, LogicalType::BIGINT},
                                   LogicalType::BIGINT,
                                   ScalarFunction::BinaryFunction<int64_t, int64_t, int64_t, AddOperator>));

    add.AddFunction(ScalarFunction({LogicalType::DOUBLE, LogicalType::DOUBLE},
                                   LogicalType::DOUBLE,
                                   ScalarFunction::BinaryFunction<double, double, double, AddOperator>));

    return add;
}
```

### Example: Function with Custom Logic (Not Templated)

For functions requiring special logic:

```cpp
void MyCustomFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    // Get the input vector
    auto &input = args.data[0];
    auto count = args.size();

    // Use UnaryExecutor with a lambda for custom logic
    UnaryExecutor::Execute<string_t, int64_t>(
        input, result, count,
        [&](string_t input) {
            // Custom logic here
            if (input.GetSize() == 0) {
                return int64_t(0);
            }
            // More complex processing...
            return static_cast<int64_t>(input.GetSize());
        }
    );
}

ScalarFunction CustomFun::GetFunction() {
    return ScalarFunction("my_custom",
                         {LogicalType::VARCHAR},
                         LogicalType::BIGINT,
                         MyCustomFunction);  // Pass function pointer directly
}
```

### Example: Function with Bind Phase

For functions that need to determine return type at bind time:

```cpp
unique_ptr<FunctionData> ArrayLengthBind(ClientContext &context,
                                         ScalarFunction &bound_function,
                                         vector<unique_ptr<Expression>> &arguments) {
    // Check parameter availability
    if (arguments[0]->HasParameter() ||
        arguments[0]->return_type.id() == LogicalTypeId::UNKNOWN) {
        throw ParameterNotResolvedException();
    }

    // Set the function based on input type
    const auto &arg_type = arguments[0]->return_type.id();
    if (arg_type == LogicalTypeId::ARRAY) {
        bound_function.function = ArrayLengthFunction;
    } else if (arg_type == LogicalTypeId::LIST) {
        bound_function.function = ListLengthFunction;
    } else {
        throw BinderException("length can only be used on arrays or lists");
    }

    // Update argument types
    bound_function.arguments[0] = arguments[0]->return_type;

    // Return bind data (or nullptr if not needed)
    return nullptr;
}

ScalarFunction ArrayLengthFun::GetFunction() {
    return ScalarFunction("array_length",
                         {LogicalType::LIST(LogicalType::ANY)},
                         LogicalType::BIGINT,
                         nullptr,              // function (set in bind)
                         ArrayLengthBind);     // bind function
}
```

---

## Step-by-Step: Adding an Aggregate Function

Aggregate functions combine multiple rows. They require state management.

### Architecture

Aggregate functions have four main phases:
1. **Initialize**: Set up the initial state
2. **Update**: Process each input value and update state
3. **Combine**: Merge states from parallel execution
4. **Finalize**: Convert state to final result

### Example: Simple Aggregate (SUM)

Create a file: `/src/function/aggregate/distributive/my_sum.cpp`

```cpp
#include "duckdb/function/aggregate/distributive_functions.hpp"
#include "duckdb/common/types/null_value.hpp"

namespace duckdb {

// Step 1: Define the state structure
struct SumState {
    int64_t sum;
    bool is_set;
};

// Step 2: Define the aggregate operations
struct SumFunction {
    // Initialize state
    template <class STATE>
    static void Initialize(STATE &state) {
        state.sum = 0;
        state.is_set = false;
    }

    // Process a single value (called for each row)
    template <class STATE, class OP>
    static void Operation(STATE &state, AggregateInputData &aggr_input_data,
                         idx_t idx) {
        state.sum += /* get value */;
        state.is_set = true;
    }

    // Simpler version with typed input
    template <class INPUT_TYPE, class STATE, class OP>
    static void Operation(STATE &state, INPUT_TYPE input) {
        state.sum += input;
        state.is_set = true;
    }

    // Combine two states (for parallel execution)
    template <class STATE, class OP>
    static void Combine(const STATE &source, STATE &target,
                       AggregateInputData &aggr_input_data) {
        if (source.is_set) {
            target.sum += source.sum;
            target.is_set = true;
        }
    }

    // Finalize: convert state to result
    template <class T, class STATE>
    static void Finalize(STATE &state, T &target,
                        AggregateFinalizeData &finalize_data) {
        if (!state.is_set) {
            finalize_data.ReturnNull();
        } else {
            target = state.sum;
        }
    }
};

// Step 3: Register the function
AggregateFunction SumFun::GetFunction() {
    return AggregateFunction::UnaryAggregate<SumState, int64_t, int64_t, SumFunction>(
        LogicalType::BIGINT,  // Input type
        LogicalType::BIGINT   // Return type
    );
}

} // namespace duckdb
```

### Example: Aggregate with String State

For aggregates that store non-trivial types (strings, vectors, etc.), you need a destructor:

```cpp
struct FirstStringState {
    string_t value;
    bool is_set;

    // Destructor for cleanup
    void Destroy() {
        if (is_set && !value.IsInlined()) {
            delete[] value.GetData();
        }
    }
};

struct FirstFunction {
    template <class STATE>
    static void Initialize(STATE &state) {
        state.is_set = false;
    }

    template <class INPUT_TYPE, class STATE, class OP>
    static void Operation(STATE &state, INPUT_TYPE input) {
        if (!state.is_set) {
            // Store the string value
            state.value = StringVector::AddStringOrBlob(/* allocator */, input);
            state.is_set = true;
        }
    }

    template <class STATE, class OP>
    static void Combine(const STATE &source, STATE &target,
                       AggregateInputData &aggr_input_data) {
        if (source.is_set && !target.is_set) {
            target.value = source.value;
            target.is_set = true;
        }
    }

    template <class T, class STATE>
    static void Finalize(STATE &state, T &target,
                        AggregateFinalizeData &finalize_data) {
        if (!state.is_set) {
            finalize_data.ReturnNull();
        } else {
            target = state.value;
        }
    }
};

AggregateFunction FirstFun::GetFunction() {
    // Use UnaryAggregateDestructor for non-trivial destructors
    auto func = AggregateFunction::UnaryAggregateDestructor<FirstStringState, string_t,
                                                             string_t, FirstFunction>(
        LogicalType::VARCHAR, LogicalType::VARCHAR
    );
    // Set the destructor
    func.destructor = AggregateFunction::StateDestroy<FirstStringState, FirstFunction>;
    return func;
}
```

### Example: COUNT Implementation

Here's the actual COUNT implementation from DuckDB:

```cpp
struct CountFunction : public BaseCountFunction {
    using STATE = int64_t;

    static void Operation(STATE &state) {
        state += 1;
    }

    static void ConstantOperation(STATE &state, idx_t count) {
        state += count;
    }

    static bool IgnoreNull() {
        return true;  // COUNT ignores NULL values
    }
};

AggregateFunction CountFun::GetFunction() {
    auto fun = AggregateFunction(
        {LogicalType(LogicalTypeId::ANY)},  // Input type
        LogicalType::BIGINT,                 // Return type
        AggregateFunction::StateSize<int64_t>,
        AggregateFunction::StateInitialize<int64_t, CountFunction>,
        CountFunction::CountScatter,         // Custom scatter update
        AggregateFunction::StateCombine<int64_t, CountFunction>,
        AggregateFunction::StateFinalize<int64_t, int64_t, CountFunction>,
        FunctionNullHandling::SPECIAL_HANDLING,  // Handle NULLs specially
        CountFunction::CountUpdate           // Custom simple update
    );
    fun.name = "count";
    fun.order_dependent = AggregateOrderDependent::NOT_ORDER_DEPENDENT;
    return fun;
}
```

---

## Step-by-Step: Adding a Table Function

Table functions generate or transform entire tables.

### Architecture

Table functions have:
1. **Bind**: Determine output schema and create bind data
2. **Init Global**: Initialize global state (shared across threads)
3. **Init Local**: Initialize per-thread local state
4. **Main Function**: Generate output rows

### Example: Simple Range Function

```cpp
#include "duckdb/function/table_function.hpp"

namespace duckdb {

// Step 1: Define bind data (stores function parameters)
struct RangeFunctionBindData : public TableFunctionData {
    int64_t start;
    int64_t end;
    int64_t increment;
    idx_t cardinality;

    RangeFunctionBindData(int64_t start, int64_t end, int64_t increment)
        : start(start), end(end), increment(increment) {
        cardinality = (end - start) / increment;
    }
};

// Step 2: Define local state (per-thread state)
struct RangeFunctionLocalState : public LocalTableFunctionState {
    idx_t current_idx;

    RangeFunctionLocalState() : current_idx(0) {}
};

// Step 3: Bind function - determines output schema
static unique_ptr<FunctionData> RangeBind(ClientContext &context,
                                         TableFunctionBindInput &input,
                                         vector<LogicalType> &return_types,
                                         vector<string> &names) {
    // Define output schema
    return_types.push_back(LogicalType::BIGINT);
    names.push_back("range");

    // Parse parameters
    if (input.inputs.empty() || input.inputs.size() > 3) {
        throw BinderException("range requires 1-3 arguments");
    }

    int64_t start = 0, end = 0, increment = 1;
    if (input.inputs.size() == 1) {
        end = input.inputs[0].GetValue<int64_t>();
    } else if (input.inputs.size() == 2) {
        start = input.inputs[0].GetValue<int64_t>();
        end = input.inputs[1].GetValue<int64_t>();
    } else {
        start = input.inputs[0].GetValue<int64_t>();
        end = input.inputs[1].GetValue<int64_t>();
        increment = input.inputs[2].GetValue<int64_t>();
    }

    return make_uniq<RangeFunctionBindData>(start, end, increment);
}

// Step 4: Initialize local state
static unique_ptr<LocalTableFunctionState> RangeLocalInit(
    ExecutionContext &context,
    TableFunctionInitInput &input,
    GlobalTableFunctionState *global_state) {
    return make_uniq<RangeFunctionLocalState>();
}

// Step 5: Main function - generate output
static void RangeFunction(ClientContext &context, TableFunctionInput &data_p,
                         DataChunk &output) {
    auto &bind_data = data_p.bind_data->Cast<RangeFunctionBindData>();
    auto &state = data_p.local_state->Cast<RangeFunctionLocalState>();

    idx_t remaining = (bind_data.end - bind_data.start) / bind_data.increment
                      - state.current_idx;
    idx_t this_count = MinValue<idx_t>(remaining, STANDARD_VECTOR_SIZE);

    if (this_count == 0) {
        // No more data
        output.SetCardinality(0);
        return;
    }

    // Generate values using Sequence vector
    int64_t start_value = bind_data.start +
                         (state.current_idx * bind_data.increment);
    output.data[0].Sequence(start_value, bind_data.increment, this_count);
    output.SetCardinality(this_count);

    state.current_idx += this_count;
}

// Step 6: Optional - Cardinality function for optimization
unique_ptr<NodeStatistics> RangeCardinality(ClientContext &context,
                                            const FunctionData *bind_data_p) {
    auto &bind_data = bind_data_p->Cast<RangeFunctionBindData>();
    return make_uniq<NodeStatistics>(bind_data.cardinality, bind_data.cardinality);
}

// Step 7: Register the function
void RangeFun::Register(BuiltinFunctions &set) {
    TableFunction range_func("range",
                            {LogicalType::BIGINT},
                            RangeFunction,
                            RangeBind,
                            nullptr,  // init_global (optional)
                            RangeLocalInit);
    range_func.cardinality = RangeCardinality;
    set.AddFunction(range_func);
}

} // namespace duckdb
```

### Example: Table Function with In-Out Pattern

For functions that process input rows and produce output rows (streaming):

```cpp
static OperatorResultType RangeInOutFunction(ExecutionContext &context,
                                            TableFunctionInput &data_p,
                                            DataChunk &input,
                                            DataChunk &output) {
    auto &state = data_p.local_state->Cast<RangeFunctionLocalState>();

    while (true) {
        if (state.current_input_row >= input.size()) {
            // Need more input
            state.current_input_row = 0;
            return OperatorResultType::NEED_MORE_INPUT;
        }

        // Process current input row and generate output
        // ... generate output rows ...

        if (output.size() > 0) {
            return OperatorResultType::HAVE_MORE_OUTPUT;
        }

        state.current_input_row++;
    }
}

void RangeFun::Register(BuiltinFunctions &set) {
    TableFunction range_func({LogicalType::BIGINT}, nullptr, RangeBind,
                            nullptr, RangeLocalInit);
    range_func.in_out_function = RangeInOutFunction;  // Use in-out pattern
    set.AddFunction(range_func);
}
```

---

## Step-by-Step: Adding a Window Function

Window functions compute values over a window of rows. They're typically implemented as special aggregate functions.

### Example: Simple Window Function (ROW_NUMBER)

```cpp
#include "duckdb/function/aggregate_function.hpp"
#include "duckdb/function/window/window_aggregate_function.hpp"

namespace duckdb {

struct RowNumberState {
    int64_t row_number;
};

struct RowNumberFunction {
    template <class STATE>
    static void Initialize(STATE &state) {
        state.row_number = 0;
    }

    // Window-specific function
    static void Window(AggregateInputData &aggr_input_data,
                      const WindowPartitionInput &partition,
                      const_data_ptr_t g_state,
                      data_ptr_t l_state,
                      const SubFrames &subframes,
                      Vector &result,
                      idx_t rid) {
        // For row_number, just return the current row index + 1
        auto data = FlatVector::GetData<int64_t>(result);
        data[rid] = rid + 1;
    }
};

AggregateFunction RowNumberFun::GetFunction() {
    // Window-only constructor
    return AggregateFunction(
        {},                                          // No inputs
        LogicalType::BIGINT,                        // Return type
        AggregateFunction::StateSize<RowNumberState>,
        AggregateFunction::StateInitialize<RowNumberState, RowNumberFunction>,
        nullptr,                                     // window_init
        RowNumberFunction::Window                   // window function
    );
}

} // namespace duckdb
```

### Example: Window Aggregate (SUM OVER)

Many window functions reuse aggregate logic:

```cpp
static void CountStarWindow(AggregateInputData &aggr_input_data,
                           const WindowPartitionInput &partition,
                           const_data_ptr_t g_state,
                           data_ptr_t l_state,
                           const SubFrames &subframes,
                           Vector &result,
                           idx_t rid) {
    auto data = FlatVector::GetData<int64_t>(result);
    int64_t total = 0;

    // Iterate over all frames
    for (const auto &frame : subframes) {
        const auto begin = frame.start;
        const auto end = frame.end;

        // Count rows in frame
        if (partition.filter_mask.AllValid()) {
            total += end - begin;
        } else {
            for (auto i = begin; i < end; ++i) {
                total += partition.filter_mask.RowIsValid(i);
            }
        }
    }

    data[rid] = total;
}

AggregateFunction CountStarFun::GetFunction() {
    auto fun = AggregateFunction::NullaryAggregate<int64_t, int64_t, CountStarFunction>(
        LogicalType::BIGINT
    );
    fun.window = CountStarWindow;  // Add window support
    return fun;
}
```

---

## Function Binding and Overloading

Function binding is the process of matching a function call to its implementation.

### Function Sets

Use `FunctionSet` to provide multiple overloads:

```cpp
ScalarFunctionSet LengthFun::GetFunctions() {
    ScalarFunctionSet length("length");

    // Overload 1: VARCHAR -> BIGINT
    length.AddFunction(ScalarFunction(
        {LogicalType::VARCHAR},
        LogicalType::BIGINT,
        ScalarFunction::UnaryFunction<string_t, int64_t, StringLengthOperator>
    ));

    // Overload 2: BIT -> BIGINT
    length.AddFunction(ScalarFunction(
        {LogicalType::BIT},
        LogicalType::BIGINT,
        ScalarFunction::UnaryFunction<string_t, int64_t, BitStringLenOperator>
    ));

    // Overload 3: LIST(ANY) -> BIGINT (with bind)
    length.AddFunction(ScalarFunction(
        {LogicalType::LIST(LogicalType::ANY)},
        LogicalType::BIGINT,
        nullptr,
        ArrayOrListLengthBind  // Custom bind resolves LIST vs ARRAY
    ));

    return length;
}
```

### Custom Bind Function

Bind functions can:
- Validate arguments
- Determine return type dynamically
- Select the actual implementation function
- Store bind-time data

```cpp
struct MyFunctionBindData : public FunctionData {
    int64_t constant_value;

    unique_ptr<FunctionData> Copy() const override {
        auto copy = make_uniq<MyFunctionBindData>();
        copy->constant_value = constant_value;
        return copy;
    }

    bool Equals(const FunctionData &other) const override {
        auto &other_data = other.Cast<MyFunctionBindData>();
        return constant_value == other_data.constant_value;
    }
};

unique_ptr<FunctionData> MyFunctionBind(ClientContext &context,
                                       ScalarFunction &bound_function,
                                       vector<unique_ptr<Expression>> &arguments) {
    // Validate arguments
    if (arguments.empty()) {
        throw BinderException("Function requires at least one argument");
    }

    // Check for parameter expressions (prepared statements)
    for (auto &arg : arguments) {
        if (arg->HasParameter()) {
            throw ParameterNotResolvedException();
        }
    }

    // Determine return type based on input
    if (arguments[0]->return_type == LogicalType::INTEGER) {
        bound_function.return_type = LogicalType::BIGINT;
    }

    // Create and return bind data
    auto bind_data = make_uniq<MyFunctionBindData>();
    bind_data->constant_value = 42;
    return bind_data;
}
```

### Type Resolution

DuckDB automatically handles type casting for overloaded functions:

```cpp
AggregateFunctionSet SumFun::GetFunctions() {
    AggregateFunctionSet sum("sum");

    // Add overloads for all numeric types
    sum.AddFunction(AggregateFunction::UnaryAggregate<int64_t, int64_t, int64_t, SumFunction>(
        LogicalType::TINYINT, LogicalType::HUGEINT
    ));
    sum.AddFunction(AggregateFunction::UnaryAggregate<int64_t, int64_t, int64_t, SumFunction>(
        LogicalType::SMALLINT, LogicalType::HUGEINT
    ));
    // ... more overloads ...

    return sum;
}
```

---

## NULL Handling in Functions

DuckDB provides two modes for NULL handling:

### 1. Default NULL Handling

**Rule**: NULL in, NULL out (automatically)

```cpp
ScalarFunction MyFunc::GetFunction() {
    return ScalarFunction("my_func",
                         {LogicalType::INTEGER},
                         LogicalType::INTEGER,
                         ScalarFunction::UnaryFunction<int32_t, int32_t, MyOperator>,
                         nullptr, nullptr, nullptr, nullptr,
                         LogicalType::INVALID,
                         FunctionNullHandling::DEFAULT_NULL_HANDLING);
}
```

With default NULL handling:
- If any input is NULL, the result is NULL
- Your function doesn't need to check for NULLs
- This is the most common case

### 2. Special NULL Handling

**Rule**: Function handles NULLs explicitly

```cpp
void CoalesceFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    // This function must handle NULL values explicitly

    UnifiedVectorFormat data[2];
    args.data[0].ToUnifiedFormat(args.size(), data[0]);
    args.data[1].ToUnifiedFormat(args.size(), data[1]);

    auto result_data = FlatVector::GetData<int32_t>(result);
    auto &result_validity = FlatVector::Validity(result);

    for (idx_t i = 0; i < args.size(); i++) {
        auto idx0 = data[0].sel->get_index(i);
        auto idx1 = data[1].sel->get_index(i);

        // Return first non-NULL value
        if (data[0].validity.RowIsValid(idx0)) {
            result_data[i] = /* first value */;
        } else if (data[1].validity.RowIsValid(idx1)) {
            result_data[i] = /* second value */;
        } else {
            result_validity.SetInvalid(i);
        }
    }
}

ScalarFunction CoalesceFun::GetFunction() {
    return ScalarFunction("coalesce",
                         {LogicalType::INTEGER, LogicalType::INTEGER},
                         LogicalType::INTEGER,
                         CoalesceFunction,
                         nullptr, nullptr, nullptr, nullptr,
                         LogicalType::INVALID,
                         FunctionNullHandling::SPECIAL_HANDLING);  // Handle NULLs
}
```

### NULL Handling in Aggregates

Aggregates typically ignore NULL values by default:

```cpp
struct CountFunction {
    static bool IgnoreNull() {
        return true;  // Don't count NULL values
    }

    template <class INPUT_TYPE, class STATE, class OP>
    static void Operation(STATE &state, INPUT_TYPE input) {
        // This is only called for non-NULL values
        state++;
    }
};
```

For special NULL handling in aggregates:

```cpp
AggregateFunction MyAggregate::GetFunction() {
    auto func = AggregateFunction(
        {LogicalType::INTEGER},
        LogicalType::BIGINT,
        // ... state functions ...
        FunctionNullHandling::SPECIAL_HANDLING  // Handle NULLs specially
    );
    return func;
}
```

### Working with Validity Masks

When using special NULL handling:

```cpp
void MyFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto count = args.size();

    // Get input data in unified format
    UnifiedVectorFormat input_data;
    args.data[0].ToUnifiedFormat(count, input_data);

    auto input_values = UnifiedVectorFormat::GetData<int32_t>(input_data);
    auto &input_validity = input_data.validity;

    // Prepare output
    auto result_data = FlatVector::GetData<int32_t>(result);
    auto &result_validity = FlatVector::Validity(result);

    for (idx_t i = 0; i < count; i++) {
        auto idx = input_data.sel->get_index(i);

        if (input_validity.RowIsValid(idx)) {
            // Process non-NULL value
            result_data[i] = Process(input_values[idx]);
        } else {
            // Handle NULL input
            result_validity.SetInvalid(i);
        }
    }
}
```

---

## Vectorized Function Implementation

DuckDB uses vectorized execution for performance. Functions should process entire vectors at once.

### Using Executors

DuckDB provides executor helpers:

#### UnaryExecutor

For functions with one input:

```cpp
struct AbsOperator {
    template <class TA, class TR>
    static inline TR Operation(TA input) {
        return input < 0 ? -input : input;
    }
};

void AbsFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    UnaryExecutor::Execute<int32_t, int32_t, AbsOperator>(
        args.data[0],  // Input vector
        result,        // Output vector
        args.size()    // Count
    );
}
```

With lambda:

```cpp
void LengthFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    UnaryExecutor::Execute<string_t, int64_t>(
        args.data[0], result, args.size(),
        [](string_t input) {
            return static_cast<int64_t>(input.GetSize());
        }
    );
}
```

#### BinaryExecutor

For functions with two inputs:

```cpp
struct AddOperator {
    template <class TA, class TB, class TR>
    static inline TR Operation(TA left, TB right) {
        return left + right;
    }
};

void AddFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    BinaryExecutor::ExecuteStandard<int32_t, int32_t, int32_t, AddOperator>(
        args.data[0],  // Left input
        args.data[1],  // Right input
        result,        // Output
        args.size()    // Count
    );
}
```

#### TernaryExecutor

For functions with three inputs:

```cpp
void SubstringFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    TernaryExecutor::ExecuteStandard<string_t, int64_t, int64_t, string_t, SubstringOp>(
        args.data[0],  // String input
        args.data[1],  // Start position
        args.data[2],  // Length
        result,
        args.size()
    );
}
```

### Vector Types

DuckDB uses different vector types for optimization:

```cpp
void MyFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input = args.data[0];

    switch (input.GetVectorType()) {
        case VectorType::CONSTANT_VECTOR: {
            // Handle constant (all values the same)
            if (ConstantVector::IsNull(input)) {
                result.SetVectorType(VectorType::CONSTANT_VECTOR);
                ConstantVector::SetNull(result, true);
                return;
            }
            auto value = ConstantVector::GetData<int32_t>(input)[0];
            auto result_value = Process(value);
            result.SetVectorType(VectorType::CONSTANT_VECTOR);
            ConstantVector::GetData<int32_t>(result)[0] = result_value;
            break;
        }
        case VectorType::FLAT_VECTOR: {
            // Handle flat (contiguous) vector - most common
            auto input_data = FlatVector::GetData<int32_t>(input);
            auto result_data = FlatVector::GetData<int32_t>(result);
            for (idx_t i = 0; i < args.size(); i++) {
                result_data[i] = Process(input_data[i]);
            }
            break;
        }
        default: {
            // Use UnifiedFormat for other types
            UnifiedVectorFormat input_data;
            input.ToUnifiedFormat(args.size(), input_data);
            // ... process ...
            break;
        }
    }
}
```

### Sequence Vectors

For generating sequential values efficiently:

```cpp
void RangeFunction(DataChunk &output, int64_t start, int64_t increment,
                  idx_t count) {
    // Creates vector [start, start+increment, start+2*increment, ...]
    output.data[0].Sequence(start, increment, count);
    output.SetCardinality(count);
}
```

### String Vectors

Special handling for strings:

```cpp
void ConcatFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    UnifiedVectorFormat left_data, right_data;
    args.data[0].ToUnifiedFormat(args.size(), left_data);
    args.data[1].ToUnifiedFormat(args.size(), right_data);

    auto left_strings = UnifiedVectorFormat::GetData<string_t>(left_data);
    auto right_strings = UnifiedVectorFormat::GetData<right_data);

    auto result_data = FlatVector::GetData<string_t>(result);

    for (idx_t i = 0; i < args.size(); i++) {
        auto left_idx = left_data.sel->get_index(i);
        auto right_idx = right_data.sel->get_index(i);

        if (!left_data.validity.RowIsValid(left_idx) ||
            !right_data.validity.RowIsValid(right_idx)) {
            FlatVector::SetNull(result, i, true);
            continue;
        }

        auto left = left_strings[left_idx];
        auto right = right_strings[right_idx];

        // Allocate result string
        auto total_len = left.GetSize() + right.GetSize();
        result_data[i] = StringVector::EmptyString(result, total_len);

        // Copy data
        auto result_ptr = result_data[i].GetDataWriteable();
        memcpy(result_ptr, left.GetData(), left.GetSize());
        memcpy(result_ptr + left.GetSize(), right.GetData(), right.GetSize());
        result_data[i].Finalize();
    }
}
```

---

## Function Statistics

Statistics help the optimizer make better decisions.

### Scalar Function Statistics

```cpp
unique_ptr<BaseStatistics> LengthPropagateStats(ClientContext &context,
                                               FunctionStatisticsInput &input) {
    auto &child_stats = input.child_stats;
    auto &expr = input.expr;

    D_ASSERT(child_stats.size() == 1);

    // Optimize: if input cannot contain unicode, use byte length
    if (!StringStats::CanContainUnicode(child_stats[0])) {
        // Switch to faster byte-length function
        expr.function.function = ScalarFunction::UnaryFunction<string_t, int64_t,
                                                               StrLenOperator>;
    }

    // Can also return statistics about the result
    return nullptr;  // Or return statistics object
}

ScalarFunction LengthFun::GetFunction() {
    return ScalarFunction("length",
                         {LogicalType::VARCHAR},
                         LogicalType::BIGINT,
                         ScalarFunction::UnaryFunction<string_t, int64_t, StringLengthOperator>,
                         nullptr,
                         nullptr,
                         LengthPropagateStats);  // Statistics function
}
```

### Aggregate Function Statistics

```cpp
unique_ptr<BaseStatistics> CountPropagateStats(ClientContext &context,
                                              BoundAggregateExpression &expr,
                                              AggregateStatisticsInput &input) {
    // Optimize: COUNT on column without nulls -> COUNT(*)
    if (!expr.IsDistinct() && !input.child_stats[0].CanHaveNull()) {
        expr.function = CountStarFun::GetFunction();
        expr.function.name = "count_star";
        expr.children.clear();  // Remove children (no longer needed)
    }
    return nullptr;
}

AggregateFunction CountFun::GetFunction() {
    auto count_function = CountFunctionBase::GetFunction();
    count_function.statistics = CountPropagateStats;  // Add statistics
    return count_function;
}
```

### Table Function Cardinality

```cpp
unique_ptr<NodeStatistics> RangeCardinality(ClientContext &context,
                                           const FunctionData *bind_data_p) {
    if (!bind_data_p) {
        return nullptr;
    }
    auto &bind_data = bind_data_p->Cast<RangeFunctionBindData>();

    // Return exact cardinality (min and max are the same)
    return make_uniq<NodeStatistics>(bind_data.cardinality,
                                    bind_data.cardinality);
}

TableFunction RangeFun::GetFunction() {
    TableFunction func("range", {LogicalType::BIGINT}, RangeFunction, RangeBind);
    func.cardinality = RangeCardinality;  // Provide cardinality
    return func;
}
```

---

## Testing Functions

DuckDB strongly prefers **sqllogictest** (`.test` files) for testing.

### Creating a Test File

Create a file: `/test/sql/function/test_my_function.test`

```sql
# name: test/sql/function/test_my_function.test
# description: Test my_function
# group: [function]

# Test basic functionality
statement ok
CREATE TABLE test (a INTEGER);

statement ok
INSERT INTO test VALUES (1), (2), (3), (NULL);

# Test with integers
query I
SELECT my_function(a) FROM test ORDER BY a;
----
2
4
6
NULL

# Test with different types
query I
SELECT my_function(42);
----
84

# Test error cases
statement error
SELECT my_function('invalid');
----
Cannot convert

# Test with constants
query I
SELECT my_function(NULL);
----
NULL

# Test edge cases
query I
SELECT my_function(0);
----
0

query I
SELECT my_function(-5);
----
-10

# Test with large values
query I
SELECT my_function(2147483647);
----
Overflow

# Test in WHERE clause
query I
SELECT a FROM test WHERE my_function(a) > 3;
----
2
3

# Test in aggregates
query I
SELECT SUM(my_function(a)) FROM test;
----
12
```

### Test File Conventions

- **Name**: `test_<function_name>.test`
- **Location**: `/test/sql/function/` or appropriate subdirectory
- **Slow tests**: Use `.test_slow` extension for tests that take >1 second
- **Groups**: Add `# group: [function]` for organization

### Common Test Patterns

```sql
# Test NULL handling
query I
SELECT my_function(NULL);
----
NULL

# Test multiple rows
query II
SELECT a, my_function(a) FROM test ORDER BY a;
----
1    2
2    4
3    6

# Test error with pattern matching
statement error
SELECT my_function('bad');
----
.*type.*

# Test with prepared statements
statement ok
PREPARE p AS SELECT my_function($1);

query I
EXECUTE p(5);
----
10

# Test aggregate function
query I
SELECT my_aggregate(a) FROM test;
----
42

# Test window function
query II
SELECT a, my_window_func() OVER (ORDER BY a) FROM test;
----
1    1
2    2
3    3
```

### Running Tests

```bash
# Run specific test
build/debug/test/unittest "test/sql/function/test_my_function.test"

# Run all function tests
build/debug/test/unittest "[function]"

# Run test with specific tag
build/debug/test/unittest "[my_tag]"
```

### C++ Tests (Less Common)

Only use C++ tests when you need to:
- Test internal APIs not exposed to SQL
- Test specific edge cases requiring C++ setup

```cpp
// test/sql/function/test_my_function.cpp
#include "catch.hpp"
#include "test_helpers.hpp"

using namespace duckdb;
using namespace std;

TEST_CASE("Test my_function", "[function]") {
    DuckDB db(nullptr);
    Connection con(db);

    REQUIRE_NO_FAIL(con.Query("SELECT my_function(42)"));

    auto result = con.Query("SELECT my_function(21)");
    REQUIRE(result->RowCount() == 1);
    REQUIRE(result->GetValue(0, 0).GetValue<int32_t>() == 42);
}
```

### Test Coverage

Ensure you test:
1. **Basic functionality**: Normal inputs and outputs
2. **NULL handling**: NULL inputs and outputs
3. **Type variants**: All supported input types
4. **Edge cases**: 0, negative, max values, empty strings
5. **Error cases**: Invalid inputs, overflows
6. **Integration**: Use in WHERE, JOIN, GROUP BY, etc.
7. **Performance**: Mark slow tests appropriately

---

## Documentation Requirements

Every function needs documentation for users.

### Inline Documentation

Add documentation in the function registration:

```cpp
// In the header file or registration code
ScalarFunctionSet MyFun::GetFunctions() {
    ScalarFunctionSet functions("my_function");

    functions.AddFunction(ScalarFunction(
        {LogicalType::INTEGER},
        LogicalType::INTEGER,
        MyFunctionImpl
    ));

    return functions;
}

// Documentation is typically added via the function list
// See src/function/function_list.cpp for examples
```

### Function Descriptions

DuckDB uses structured function descriptions:

```cpp
{
    "my_function",           // Function name
    "description",           // Brief description
    "example",              // SQL example
    "category",             // Function category
    {                       // Parameter descriptions
        {"parameter1", "Description of parameter1"},
        {"parameter2", "Description of parameter2"}
    }
}
```

### Documentation Location

- Function list: `/src/function/function_list.cpp`
- Online docs: https://duckdb.org/docs/
- The function description system is used to auto-generate documentation

---

## Common Patterns and Best Practices

### Pattern 1: Simple Unary Scalar Function

```cpp
struct MyOperator {
    template <class TA, class TR>
    static inline TR Operation(TA input) {
        return /* process input */;
    }
};

ScalarFunction MyFun::GetFunction() {
    return ScalarFunction("my_func",
                         {LogicalType::INTEGER},
                         LogicalType::INTEGER,
                         ScalarFunction::UnaryFunction<int32_t, int32_t, MyOperator>);
}
```

**Use when**: Simple transformation, no special state needed

### Pattern 2: Function with Multiple Overloads

```cpp
ScalarFunctionSet MyFun::GetFunctions() {
    ScalarFunctionSet set("my_func");

    for (auto &type : {LogicalType::TINYINT, LogicalType::SMALLINT,
                       LogicalType::INTEGER, LogicalType::BIGINT}) {
        set.AddFunction(ScalarFunction({type}, type,
            ScalarFunction::GetScalarUnaryFunction<MyOperator>(type)));
    }

    return set;
}
```

**Use when**: Function works on multiple types with same logic

### Pattern 3: Function with Bind Phase

```cpp
unique_ptr<FunctionData> MyBind(ClientContext &context,
                               ScalarFunction &bound_function,
                               vector<unique_ptr<Expression>> &arguments) {
    // Validate, determine types, select implementation
    bound_function.return_type = /* determine return type */;
    bound_function.function = /* select implementation */;
    return /* bind data or nullptr */;
}

ScalarFunction MyFun::GetFunction() {
    return ScalarFunction("my_func",
                         {LogicalType::ANY},
                         LogicalType::ANY,
                         nullptr,
                         MyBind);
}
```

**Use when**: Return type depends on input, need validation, multiple implementations

### Pattern 4: Aggregate with Simple State

```cpp
struct SumFunction {
    template <class STATE>
    static void Initialize(STATE &state) {
        state = 0;
    }

    template <class INPUT_TYPE, class STATE, class OP>
    static void Operation(STATE &state, INPUT_TYPE input) {
        state += input;
    }

    template <class STATE, class OP>
    static void Combine(const STATE &source, STATE &target, AggregateInputData &) {
        target += source;
    }

    template <class T, class STATE>
    static void Finalize(STATE &state, T &target, AggregateFinalizeData &) {
        target = state;
    }
};

AggregateFunction SumFun::GetFunction() {
    return AggregateFunction::UnaryAggregate<int64_t, int64_t, int64_t, SumFunction>(
        LogicalType::BIGINT, LogicalType::BIGINT
    );
}
```

**Use when**: Simple accumulation, trivial state

### Pattern 5: Aggregate with Complex State

```cpp
struct ComplexState {
    vector<int32_t> values;

    void Destroy() {
        // Cleanup if needed
    }
};

struct ComplexFunction {
    template <class STATE>
    static void Initialize(STATE &state) {
        new (&state) STATE();  // Placement new for complex types
    }

    // ... other methods ...
};

AggregateFunction ComplexFun::GetFunction() {
    auto func = AggregateFunction::UnaryAggregateDestructor<ComplexState, int32_t,
                                                             int32_t, ComplexFunction>(
        LogicalType::INTEGER, LogicalType::INTEGER
    );
    func.destructor = AggregateFunction::StateDestroy<ComplexState, ComplexFunction>;
    return func;
}
```

**Use when**: State needs heap allocation, non-trivial destructor

### Pattern 6: Table Function with Simple Generation

```cpp
static unique_ptr<FunctionData> Bind(ClientContext &context,
                                    TableFunctionBindInput &input,
                                    vector<LogicalType> &return_types,
                                    vector<string> &names) {
    return_types.push_back(LogicalType::INTEGER);
    names.push_back("value");
    return nullptr;
}

static void Execute(ClientContext &context, TableFunctionInput &data,
                   DataChunk &output) {
    // Generate rows
    output.SetCardinality(/* count */);
}

TableFunction MyTableFun::GetFunction() {
    return TableFunction("my_table", {}, Execute, Bind);
}
```

**Use when**: Simple row generation, no state needed

### Best Practices

#### 1. Always Use Vectorized Execution

```cpp
// GOOD: Vectorized
void MyFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    UnaryExecutor::Execute<int32_t, int32_t>(args.data[0], result, args.size(),
        [](int32_t input) { return input * 2; }
    );
}

// BAD: Row-by-row (only for complex logic)
void MyFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    for (idx_t i = 0; i < args.size(); i++) {
        // Process each row individually - avoid this!
    }
}
```

#### 2. Handle NULLs Correctly

```cpp
// GOOD: Use default NULL handling when possible
ScalarFunction("my_func", {LogicalType::INTEGER}, LogicalType::INTEGER,
               MyFunctionImpl,
               nullptr, nullptr, nullptr, nullptr,
               LogicalType::INVALID,
               FunctionNullHandling::DEFAULT_NULL_HANDLING);

// GOOD: Only use special handling when necessary
FunctionNullHandling::SPECIAL_HANDLING  // Only for COALESCE, etc.
```

#### 3. Use Appropriate Types

```cpp
// GOOD: Use proper integer types
ScalarFunction({LogicalType::INTEGER}, LogicalType::BIGINT, ...)

// GOOD: Use idx_t for sizes/counts
idx_t count = args.size();

// BAD: Use int for sizes
int count = args.size();  // Don't use int
```

#### 4. Memory Management

```cpp
// GOOD: Use make_uniq
auto data = make_uniq<MyData>();

// GOOD: Use StringVector for string allocation
auto str = StringVector::AddString(vector, "hello");

// BAD: Use new/delete
auto data = new MyData();  // Don't use new
delete data;               // Don't use delete
```

#### 5. Error Handling

```cpp
// GOOD: Throw descriptive exceptions
if (invalid_input) {
    throw InvalidInputException("Function 'my_func' requires positive values");
}

// GOOD: Use specific exception types
throw BinderException("...");
throw OutOfRangeException("...");
throw ConversionException("...");

// BAD: Generic exceptions
throw Exception("Error");  // Too generic
```

#### 6. Function Stability

```cpp
// For functions that always return the same result for same input
FunctionStability::CONSISTENT

// For functions like NOW() that are constant within a query
FunctionStability::CONSISTENT_WITHIN_QUERY

// For functions like RANDOM() that vary per row
FunctionStability::VOLATILE
```

#### 7. Testing

```sql
# GOOD: Test all aspects
query I
SELECT my_function(42);
----
84

query I
SELECT my_function(NULL);
----
NULL

statement error
SELECT my_function('invalid');
----
.*type.*

# BAD: Only test happy path
query I
SELECT my_function(42);
----
84
# Missing NULL handling, error cases, edge cases
```

#### 8. Code Organization

```cpp
// GOOD: Anonymous namespace for internal helpers
namespace duckdb {
namespace {
    struct InternalHelper { /* ... */ };
}

// Public API
ScalarFunction MyFun::GetFunction() { /* ... */ }

} // namespace duckdb
```

#### 9. Performance Considerations

```cpp
// GOOD: Use sequence vectors for simple sequences
output.data[0].Sequence(start, increment, count);

// GOOD: Use reference vectors to avoid copies
output.data[0].Reference(input.data[0]);

// GOOD: Use statistics to optimize
if (!stats.CanContainNull()) {
    // Use faster path without NULL checks
}
```

#### 10. Documentation

```cpp
// GOOD: Clear comments for complex logic
// This function implements the Frobnitz algorithm which requires
// special handling for negative values as described in...
void FrobnitzFunction(...) { /* ... */ }

// GOOD: Document assumptions
D_ASSERT(input.size() > 0);  // Caller must ensure non-empty input

// GOOD: Document ownership
// Returns a newly allocated string - caller must free
```

---

## Summary

This guide covered:

1. **Four function types**: Scalar, Aggregate, Table, Window
2. **Implementation patterns**: From simple operators to complex stateful functions
3. **Binding**: Type resolution and function overloading
4. **NULL handling**: Default vs special handling
5. **Vectorization**: Using executors and working with vectors
6. **Statistics**: Helping the optimizer make better decisions
7. **Testing**: Comprehensive test coverage with sqllogictest
8. **Documentation**: Inline and structured documentation
9. **Best practices**: Performance, correctness, maintainability

### Key Takeaways

- **Start simple**: Use templated operators and executors when possible
- **Test thoroughly**: Use sqllogictest for comprehensive coverage
- **Follow conventions**: Use DuckDB's type system, naming, and patterns
- **Optimize smart**: Use vectorization, statistics, and appropriate data structures
- **Document well**: Make your functions discoverable and understandable

### Next Steps

1. Review existing function implementations in `/src/function/`
2. Start with a simple scalar function
3. Write comprehensive tests
4. Run `make format-fix` before committing
5. Ensure `make allunit` passes

For more information:
- Browse `/src/function/` for examples
- Read CLAUDE.md for build and test instructions
- Check the DuckDB documentation at https://duckdb.org/docs/
