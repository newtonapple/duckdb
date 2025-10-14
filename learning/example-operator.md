# Adding a New Operator to DuckDB: Complete Walkthrough

This guide provides a comprehensive, step-by-step walkthrough of how to add a new operator to DuckDB. We'll walk through the architecture, implementation details, and testing for a complete operator implementation.

## Table of Contents

1. [Goal: What We're Building](#goal-what-were-building)
2. [Understanding Operator Architecture](#understanding-operator-architecture)
3. [Step 1: Define the Logical Operator](#step-1-define-the-logical-operator)
4. [Step 2: Update the Planner/Binder](#step-2-update-the-plannerbinder)
5. [Step 3: Define the Physical Operator](#step-3-define-the-physical-operator)
6. [Step 4: Implement the Execution Logic](#step-4-implement-the-execution-logic)
7. [Step 5: Handle State Management](#step-5-handle-state-management)
8. [Step 6: Connect Physical Plan Generator](#step-6-connect-physical-plan-generator)
9. [Step 7: Add Optimizer Support](#step-7-add-optimizer-support)
10. [Step 8: Build and Test](#step-8-build-and-test)
11. [Step 9: Write Comprehensive Tests](#step-9-write-comprehensive-tests)
12. [Complete Code Listings](#complete-code-listings)

---

## Goal: What We're Building

In this tutorial, we'll implement a **FILTER** operator from scratch. The FILTER operator is responsible for filtering rows based on a predicate expression (e.g., `WHERE` clause in SQL). While DuckDB already has a filter operator, we'll recreate it to understand all the components involved.

**What the FILTER operator does:**
- Takes input rows (DataChunks) from child operators
- Evaluates a boolean predicate expression on each row
- Returns only the rows where the predicate evaluates to true
- Uses selection vectors for efficient filtering (no data copying)

**Example SQL:**
```sql
SELECT * FROM table WHERE age > 18 AND status = 'active';
```

The `WHERE age > 18 AND status = 'active'` clause will be translated into a FILTER operator.

---

## Understanding Operator Architecture

DuckDB uses a **two-phase operator design**: Logical and Physical operators.

### Logical Operators (Planning Phase)

**Location:** `src/planner/operator/`

Logical operators represent **what** needs to be done, without worrying about **how** it's executed. They:
- Are created during query planning by the Binder
- Form a tree structure representing the logical query plan
- Contain high-level information (expressions, types, cardinality estimates)
- Are transformed by the optimizer
- Are platform-independent

**Key Base Class:** `LogicalOperator` (`src/include/duckdb/planner/logical_operator.hpp`)

```cpp
class LogicalOperator {
public:
    LogicalOperatorType type;                        // Type of operator
    vector<unique_ptr<LogicalOperator>> children;    // Child operators
    vector<unique_ptr<Expression>> expressions;      // Expressions (e.g., filter predicates)
    vector<LogicalType> types;                       // Output column types
    idx_t estimated_cardinality;                     // Estimated output rows

    virtual void ResolveTypes() = 0;                 // Must implement
    virtual vector<ColumnBinding> GetColumnBindings();
};
```

### Physical Operators (Execution Phase)

**Location:** `src/execution/operator/`

Physical operators represent **how** operations are executed. They:
- Are created from logical operators by the Physical Plan Generator
- Contain actual execution logic (vectorized processing)
- Handle state management for parallel execution
- Process data in chunks (DataChunk objects)
- Use push-based execution model

**Key Base Class:** `PhysicalOperator` (`src/include/duckdb/execution/physical_operator.hpp`)

```cpp
class PhysicalOperator {
public:
    PhysicalOperatorType type;
    vector<LogicalType> types;                       // Output types
    idx_t estimated_cardinality;

    // Core execution interface
    virtual OperatorResultType Execute(ExecutionContext &context,
                                      DataChunk &input,
                                      DataChunk &chunk,
                                      GlobalOperatorState &gstate,
                                      OperatorState &state) const;

    // State management
    virtual unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) const;
    virtual unique_ptr<GlobalOperatorState> GetGlobalOperatorState(ClientContext &context) const;

    // Sink interface (for operators that consume all input before producing output)
    virtual SinkResultType Sink(ExecutionContext &context, DataChunk &chunk,
                               OperatorSinkInput &input) const;
    virtual bool IsSink() const { return false; }

    // Source interface (for operators that generate data)
    virtual SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                                    OperatorSourceInput &input) const;
    virtual bool IsSource() const { return false; }
};
```

### Operator Execution Model

DuckDB uses a **push-based vectorized execution model**:

1. **Vectorized Processing:** Operators process data in batches (DataChunks) of up to 2048 rows
2. **Push-based:** Data flows from child to parent (child pushes data to parent)
3. **Pipeline Parallelism:** Multiple threads can execute the same pipeline on different data

**Operator Categories:**

1. **Stream Operators:** Process data as it arrives (e.g., FILTER, PROJECTION)
   - Input → Transform → Output immediately

2. **Blocking Operators:** Consume all input before producing output (e.g., ORDER BY, HASH GROUP BY)
   - Implement Sink interface (consume data)
   - Implement Source interface (produce results)

3. **Source Operators:** Generate data without input (e.g., TABLE_SCAN)
   - Only implement Source interface

---

## Step 1: Define the Logical Operator

First, we create the logical operator that represents the filtering operation in the query plan.

### 1.1 Create the Header File

**File:** `src/include/duckdb/planner/operator/logical_filter.hpp`

```cpp
//===----------------------------------------------------------------------===//
//                         DuckDB
//
// duckdb/planner/operator/logical_filter.hpp
//
//===----------------------------------------------------------------------===//

#pragma once

#include "duckdb/planner/logical_operator.hpp"

namespace duckdb {

//! LogicalFilter represents a filter operation (e.g. WHERE or HAVING clause)
class LogicalFilter : public LogicalOperator {
public:
	static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_FILTER;

public:
	explicit LogicalFilter(unique_ptr<Expression> expression);
	LogicalFilter();

	//! Optional projection map for column reordering
	vector<idx_t> projection_map;

public:
	vector<ColumnBinding> GetColumnBindings() override;

	bool HasProjectionMap() const override {
		return !projection_map.empty();
	}

	void Serialize(Serializer &serializer) const override;
	static unique_ptr<LogicalOperator> Deserialize(Deserializer &deserializer);

	//! Splits up the predicates of the LogicalFilter into a set of predicates
	//! separated by AND. Returns whether or not any splits were made
	bool SplitPredicates() {
		return SplitPredicates(expressions);
	}
	static bool SplitPredicates(vector<unique_ptr<Expression>> &expressions);

protected:
	void ResolveTypes() override;
};

} // namespace duckdb
```

**Key Components:**

1. **TYPE constant:** Used for operator type checking and casting
2. **Constructor:** Takes filter expression(s)
3. **projection_map:** Optional mapping for column reordering (e.g., `SELECT col2, col1 FROM t WHERE ...`)
4. **SplitPredicates():** Optimization to split `A AND B AND C` into separate predicates for better pushdown
5. **ResolveTypes():** Determines output column types

### 1.2 Implement the Logical Operator

**File:** `src/planner/operator/logical_filter.cpp`

```cpp
#include "duckdb/planner/operator/logical_filter.hpp"
#include "duckdb/planner/expression/bound_conjunction_expression.hpp"

namespace duckdb {

LogicalFilter::LogicalFilter(unique_ptr<Expression> expression)
    : LogicalOperator(LogicalOperatorType::LOGICAL_FILTER) {
	expressions.push_back(std::move(expression));
	SplitPredicates(expressions);
}

LogicalFilter::LogicalFilter()
    : LogicalOperator(LogicalOperatorType::LOGICAL_FILTER) {
}

void LogicalFilter::ResolveTypes() {
	// Filter passes through types from its child, potentially reordered
	types = MapTypes(children[0]->types, projection_map);
}

vector<ColumnBinding> LogicalFilter::GetColumnBindings() {
	// Column bindings pass through from child, potentially reordered
	return MapBindings(children[0]->GetColumnBindings(), projection_map);
}

// Split the predicates separated by AND statements
// These are safe to push down because all of them MUST be true
bool LogicalFilter::SplitPredicates(vector<unique_ptr<Expression>> &expressions) {
	bool found_conjunction = false;
	for (idx_t i = 0; i < expressions.size(); i++) {
		if (expressions[i]->GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
			auto &conjunction = expressions[i]->Cast<BoundConjunctionExpression>();
			found_conjunction = true;
			// AND expression, append the other children
			for (idx_t k = 1; k < conjunction.children.size(); k++) {
				expressions.push_back(std::move(conjunction.children[k]));
			}
			// Replace this expression with the first child
			expressions[i] = std::move(conjunction.children[0]);
			// Move back by one so the right child is checked again
			i--;
		}
	}
	return found_conjunction;
}

} // namespace duckdb
```

**Key Implementation Details:**

1. **ResolveTypes():** Filter doesn't change types, just passes them through
2. **SplitPredicates():** Splits `A AND B` into separate predicates for optimizer
   - Example: `WHERE age > 18 AND status = 'active'` becomes two separate predicates
   - Enables independent predicate pushdown to table scans

---

## Step 2: Update the Planner/Binder

The Binder is responsible for creating logical operators during query planning. You need to update the relevant bind function to create your logical operator.

### 2.1 Where Filter Operators are Created

For a FILTER operator, it's typically created when binding a SELECT statement with a WHERE clause.

**File:** `src/planner/binder/statement/bind_select.cpp`

```cpp
// Simplified example of how a filter is bound
BoundStatement Binder::Bind(SelectStatement &stmt) {
	// ... bind FROM clause ...
	auto from_result = BindFrom(stmt.from_table);

	// ... bind WHERE clause ...
	if (stmt.where_clause) {
		// Bind the WHERE expression
		auto where_expr = BindExpression(stmt.where_clause);

		// Create a LogicalFilter operator
		auto filter = make_uniq<LogicalFilter>(std::move(where_expr));
		filter->children.push_back(std::move(from_result));
		from_result = std::move(filter);
	}

	// ... continue with SELECT list, GROUP BY, etc. ...
	return CreatePlan(std::move(from_result));
}
```

**Note:** The exact binding logic varies based on where the filter appears (WHERE vs HAVING vs filter in subquery). The important concept is that the Binder:
1. Parses and validates the filter expression
2. Creates a `LogicalFilter` node
3. Inserts it into the logical operator tree

### 2.2 Register the Operator Type

Ensure your operator type is registered in the enum:

**File:** `src/include/duckdb/common/enums/logical_operator_type.hpp`

```cpp
enum class LogicalOperatorType : uint8_t {
	// ... other types ...
	LOGICAL_FILTER,
	// ... more types ...
};
```

---

## Step 3: Define the Physical Operator

Now we create the physical operator that actually executes the filtering logic.

### 3.1 Create the Physical Operator Header

**File:** `src/include/duckdb/execution/operator/filter/physical_filter.hpp`

```cpp
//===----------------------------------------------------------------------===//
//                         DuckDB
//
// duckdb/execution/operator/filter/physical_filter.hpp
//
//===----------------------------------------------------------------------===//

#pragma once

#include "duckdb/execution/physical_operator.hpp"
#include "duckdb/planner/expression.hpp"

namespace duckdb {

//! PhysicalFilter represents a filter operator. It removes non-matching tuples
//! from the result. Note that it does not physically change the data, it only
//! adds a selection vector to the chunk.
class PhysicalFilter : public CachingPhysicalOperator {
public:
	static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::FILTER;

public:
	PhysicalFilter(PhysicalPlan &physical_plan,
	               vector<LogicalType> types,
	               vector<unique_ptr<Expression>> select_list,
	               idx_t estimated_cardinality);

	//! The filter expression
	unique_ptr<Expression> expression;

public:
	unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) const override;

	bool ParallelOperator() const override {
		return true;
	}

	InsertionOrderPreservingMap<string> ParamsToString() const override;

protected:
	OperatorResultType ExecuteInternal(ExecutionContext &context,
	                                   DataChunk &input,
	                                   DataChunk &chunk,
	                                   GlobalOperatorState &gstate,
	                                   OperatorState &state) const override;
};

} // namespace duckdb
```

**Key Design Decisions:**

1. **Inherits from `CachingPhysicalOperator`:**
   - Automatically handles small chunk buffering for better performance
   - Implements `ExecuteInternal()` instead of `Execute()`

2. **`ParallelOperator() = true`:**
   - Multiple threads can execute this operator in parallel
   - Each thread has its own `OperatorState`

3. **Stream operator:**
   - Not a sink or source, just transforms data as it flows through
   - No blocking, no materialization

### 3.2 Register the Physical Operator Type

**File:** `src/include/duckdb/common/enums/physical_operator_type.hpp`

```cpp
enum class PhysicalOperatorType : uint8_t {
	INVALID,
	// ... other types ...
	FILTER,
	// ... more types ...
};
```

---

## Step 4: Implement the Execution Logic

Now we implement the actual filtering logic that executes at runtime.

### 4.1 Constructor

**File:** `src/execution/operator/filter/physical_filter.cpp`

```cpp
#include "duckdb/execution/operator/filter/physical_filter.hpp"
#include "duckdb/execution/expression_executor.hpp"
#include "duckdb/planner/expression/bound_conjunction_expression.hpp"
#include "duckdb/parallel/thread_context.hpp"

namespace duckdb {

PhysicalFilter::PhysicalFilter(PhysicalPlan &physical_plan,
                               vector<LogicalType> types,
                               vector<unique_ptr<Expression>> select_list,
                               idx_t estimated_cardinality)
    : CachingPhysicalOperator(physical_plan, PhysicalOperatorType::FILTER,
                             std::move(types), estimated_cardinality) {

	D_ASSERT(!select_list.empty());

	if (select_list.size() == 1) {
		// Single predicate
		expression = std::move(select_list[0]);
		return;
	}

	// Multiple predicates: create a conjunction from the select list
	auto conjunction = make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_AND);
	for (auto &expr : select_list) {
		conjunction->children.push_back(std::move(expr));
	}
	expression = std::move(conjunction);
}
```

**Constructor Logic:**
- Takes a list of filter expressions (predicates)
- If multiple predicates, combines them into a single AND conjunction
- Stores the combined expression for evaluation

### 4.2 Core Execution Logic

```cpp
OperatorResultType PhysicalFilter::ExecuteInternal(ExecutionContext &context,
                                                   DataChunk &input,
                                                   DataChunk &chunk,
                                                   GlobalOperatorState &gstate,
                                                   OperatorState &state_p) const {
	auto &state = state_p.Cast<FilterState>();

	// Evaluate the filter expression and get selection vector
	idx_t result_count = state.executor.SelectExpression(input, state.sel);

	if (result_count == input.size()) {
		// Nothing was filtered: reference input directly (no copy)
		chunk.Reference(input);
	} else {
		// Some rows filtered: create sliced chunk with selection vector
		chunk.Slice(input, state.sel, result_count);
	}

	return OperatorResultType::NEED_MORE_INPUT;
}
```

**Execution Flow:**

1. **Get state:** Cast generic state to our `FilterState`
2. **Evaluate expression:** `SelectExpression()` evaluates the predicate and returns:
   - A `SelectionVector` containing indices of rows that passed the filter
   - The count of rows that passed
3. **Create output:**
   - If all rows passed: Just reference the input (zero-copy)
   - If some filtered: Create a sliced view using the selection vector (still zero-copy!)
4. **Return NEED_MORE_INPUT:** This is a streaming operator, we need more data

**Performance Optimization:**
- **No data copying:** Uses selection vectors to track which rows are valid
- **Zero-copy path:** When no rows are filtered, just passes the chunk through
- **Vectorized evaluation:** Expression executor processes entire chunks at once

### 4.3 Debug and Introspection

```cpp
InsertionOrderPreservingMap<string> PhysicalFilter::ParamsToString() const {
	InsertionOrderPreservingMap<string> result;
	result["__expression__"] = expression->GetName();
	SetEstimatedCardinality(result, estimated_cardinality);
	return result;
}

} // namespace duckdb
```

**Purpose:** Used by `EXPLAIN` queries to show operator details.

---

## Step 5: Handle State Management

Physical operators need state to track execution progress. There are two types of state:

1. **OperatorState (local):** Per-thread state for parallel execution
2. **GlobalOperatorState (global):** Shared state across all threads

### 5.1 Define the Operator State

For a FILTER operator, we need:
- An `ExpressionExecutor` to evaluate the filter predicate
- A `SelectionVector` to store which rows passed the filter

```cpp
class FilterState : public CachingOperatorState {
public:
	explicit FilterState(ExecutionContext &context, Expression &expr)
	    : executor(context.client, expr), sel(STANDARD_VECTOR_SIZE) {
	}

	ExpressionExecutor executor;  // Evaluates the filter expression
	SelectionVector sel;           // Stores indices of passing rows

public:
	void Finalize(const PhysicalOperator &op, ExecutionContext &context) override {
		context.thread.profiler.Flush(op);
	}
};
```

**Key Components:**

1. **ExpressionExecutor:** Handles expression evaluation
   - Initialized with the filter expression
   - Reused across multiple Execute() calls

2. **SelectionVector:** Fixed-size array (2048 entries)
   - Stores row indices that pass the filter
   - Reused to avoid allocations

3. **Finalize():** Called when thread finishes
   - Flushes profiling information

### 5.2 Create the State Factory

```cpp
unique_ptr<OperatorState> PhysicalFilter::GetOperatorState(ExecutionContext &context) const {
	return make_uniq<FilterState>(context, *expression);
}
```

**Called by:** The execution engine when starting operator execution
**Called when:** Once per thread for parallel execution
**Returns:** A new state object for this thread

### 5.3 State Lifetime

```
Pipeline Execution:
┌─────────────────────────────────────────────────────┐
│ Thread 1                  Thread 2                  │
├─────────────────────────────────────────────────────┤
│ GetOperatorState()        GetOperatorState()        │
│ ↓                         ↓                         │
│ FilterState 1             FilterState 2             │
│ ↓                         ↓                         │
│ Execute() [chunk 1]       Execute() [chunk 3]       │
│ Execute() [chunk 2]       Execute() [chunk 4]       │
│ ...                       ...                       │
│ ↓                         ↓                         │
│ Finalize()                Finalize()                │
└─────────────────────────────────────────────────────┘
```

---

## Step 6: Connect Physical Plan Generator

The Physical Plan Generator converts logical operators to physical operators.

### 6.1 Create the Plan Generator

**File:** `src/execution/physical_plan/plan_filter.cpp`

```cpp
#include "duckdb/execution/operator/filter/physical_filter.hpp"
#include "duckdb/execution/operator/projection/physical_projection.hpp"
#include "duckdb/execution/physical_plan_generator.hpp"
#include "duckdb/planner/expression/bound_reference_expression.hpp"
#include "duckdb/planner/operator/logical_filter.hpp"

namespace duckdb {

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalFilter &op) {
	D_ASSERT(op.children.size() == 1);

	// First, create the physical plan for the child
	reference<PhysicalOperator> plan = CreatePlan(*op.children[0]);

	// Create the filter operator if there are expressions to filter
	if (!op.expressions.empty()) {
		D_ASSERT(!plan.get().GetTypes().empty());

		// Create PhysicalFilter with the filter expressions
		auto &filter = Make<PhysicalFilter>(plan.get().GetTypes(),
		                                    std::move(op.expressions),
		                                    op.estimated_cardinality);
		filter.children.push_back(plan);
		plan = filter;
	}

	// If there's a projection map, add a projection operator
	if (op.HasProjectionMap()) {
		vector<unique_ptr<Expression>> select_list;
		for (idx_t i = 0; i < op.projection_map.size(); i++) {
			select_list.push_back(
				make_uniq<BoundReferenceExpression>(op.types[i], op.projection_map[i])
			);
		}
		auto &proj = Make<PhysicalProjection>(op.types,
		                                      std::move(select_list),
		                                      op.estimated_cardinality);
		proj.children.push_back(plan);
		plan = proj;
	}

	return plan;
}

} // namespace duckdb
```

**Conversion Process:**

1. **Recursively create child plans:** The filter needs its input
2. **Create PhysicalFilter:** If expressions exist
3. **Add projection:** If columns need reordering
4. **Build operator tree:** Child → Filter → (optional) Projection

**Example Transformation:**

```
Logical Plan:              Physical Plan:
┌─────────────┐           ┌─────────────┐
│ PROJECTION  │           │ PROJECTION  │
└──────┬──────┘           └──────┬──────┘
       │                         │
┌──────┴──────┐           ┌──────┴──────┐
│   FILTER    │   ────>   │   FILTER    │
└──────┬──────┘           └──────┬──────┘
       │                         │
┌──────┴──────┐           ┌──────┴──────┐
│     GET     │           │  TABLE_SCAN │
└─────────────┘           └─────────────┘
```

### 6.2 Register the Plan Creator

The Physical Plan Generator needs to know how to handle `LogicalFilter` operators.

**File:** `src/execution/physical_plan_generator.cpp`

```cpp
PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalOperator &op) {
	switch (op.type) {
	// ... other cases ...
	case LogicalOperatorType::LOGICAL_FILTER:
		return CreatePlan(op.Cast<LogicalFilter>());
	// ... more cases ...
	default:
		throw InternalException("Unimplemented logical operator type");
	}
}
```

---

## Step 7: Add Optimizer Support

Optimizers can transform and improve your operator. Here are common optimizations for filters.

### 7.1 Filter Pushdown

Push filters as close to the data source as possible to reduce data movement.

**File:** `src/optimizer/pushdown/pushdown_filter.cpp`

```cpp
// Simplified example of filter pushdown logic
unique_ptr<LogicalOperator> FilterPushdown::PushdownFilter(unique_ptr<LogicalOperator> op) {
	switch (op->type) {
	case LogicalOperatorType::LOGICAL_FILTER: {
		auto &filter = op->Cast<LogicalFilter>();

		// Try to push filter predicates into child operator
		// For example, push into table scan for storage-level filtering
		if (filter.children[0]->type == LogicalOperatorType::LOGICAL_GET) {
			auto &get = filter.children[0]->Cast<LogicalGet>();

			// Add filter predicates to table scan
			for (auto &expr : filter.expressions) {
				if (CanPushdown(expr)) {
					get.table_filters.push_back(expr->Copy());
				}
			}

			// Remove pushed predicates from filter
			RemovePushedPredicates(filter.expressions);

			// If all predicates were pushed, remove filter entirely
			if (filter.expressions.empty()) {
				return std::move(filter.children[0]);
			}
		}
		return op;
	}
	default:
		return op;
	}
}
```

**Benefits of Filter Pushdown:**
- Reduces data read from disk (storage-level filtering)
- Reduces network transfer (for remote tables)
- Reduces memory usage (fewer rows in memory)
- Improves all downstream operators (less data to process)

### 7.2 Predicate Simplification

**File:** `src/optimizer/expression_rewriter.cpp`

```cpp
// Examples of filter simplifications:
// 1. Constant folding: "WHERE 1 + 1 = 2" → "WHERE TRUE" → remove filter
// 2. Remove redundant predicates: "WHERE a > 5 AND a > 3" → "WHERE a > 5"
// 3. Contradiction detection: "WHERE a > 5 AND a < 3" → empty result
```

### 7.3 Filter Statistics

Track selectivity for cost-based optimization:

```cpp
void LogicalFilter::UpdateStatistics(StatisticsBundle &stats) {
	// Estimate how many rows pass the filter
	// This helps the optimizer estimate cardinality
	estimated_cardinality = ApplySelectivity(
		children[0]->estimated_cardinality,
		EstimateFilterSelectivity(expressions)
	);
}
```

---

## Step 8: Build and Test

### 8.1 Add to Build System

**File:** `src/execution/operator/filter/CMakeLists.txt`

```cmake
add_library(duckdb_operator_filter OBJECT
    physical_filter.cpp)
```

**File:** `src/execution/operator/CMakeLists.txt`

```cmake
add_subdirectory(filter)
```

### 8.2 Build the Project

```bash
# Clean build
make clean

# Build debug version (faster compilation, includes sanitizers)
make debug

# Or build release version (optimized)
make release

# Using Ninja for faster parallel builds
GEN=ninja make debug

# Limit parallel jobs to avoid out-of-memory
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make debug
```

**Common Build Issues:**

1. **Missing includes:** Add `#include` statements
2. **Linker errors:** Ensure CMakeLists.txt is correct
3. **Unity build conflicts:** Disable with `DISABLE_UNITY=1 make` if needed

### 8.3 Run Basic Tests

```bash
# Run fast unit tests (1-2 minutes)
make unit

# Run specific test
build/debug/test/unittest "[filter]"

# Run all tests (takes ~1 hour)
make allunit
```

---

## Step 9: Write Comprehensive Tests

DuckDB strongly prefers **sqllogictest** (`.test` files) over C++ tests.

### 9.1 Create a SQL Logic Test

**File:** `test/sql/filter/test_basic_filter.test`

```sql
# name: test/sql/filter/test_basic_filter.test
# description: Test basic filter functionality
# group: [filter]

# Enable verification mode to catch bugs
statement ok
PRAGMA enable_verification

# Create test table
statement ok
CREATE TABLE test (id INTEGER, value VARCHAR, amount DECIMAL(10,2))

# Insert test data
statement ok
INSERT INTO test VALUES
    (1, 'apple', 10.50),
    (2, 'banana', 5.25),
    (3, 'apple', 7.00),
    (4, NULL, 3.00),
    (5, 'cherry', NULL)

# Test 1: Simple equality filter
query IVR
SELECT * FROM test WHERE value = 'apple' ORDER BY id
----
1	apple	10.50
3	apple	7.00

# Test 2: Numeric comparison
query IVR
SELECT * FROM test WHERE amount > 5.00 ORDER BY id
----
1	apple	10.50
2	banana	5.25
3	apple	7.00

# Test 3: AND conjunction
query IVR
SELECT * FROM test WHERE value = 'apple' AND amount > 8.00
----
1	apple	10.50

# Test 4: OR disjunction
query IVR
SELECT * FROM test WHERE value = 'apple' OR amount < 4.00 ORDER BY id
----
1	apple	10.50
3	apple	7.00
4	NULL	3.00

# Test 5: NULL handling
query IVR
SELECT * FROM test WHERE value IS NULL
----
4	NULL	3.00

query IVR
SELECT * FROM test WHERE amount IS NULL
----
5	cherry	NULL

# Test 6: Complex expression
query IVR
SELECT * FROM test WHERE (value = 'apple' OR value = 'banana') AND amount BETWEEN 5.00 AND 10.00 ORDER BY id
----
2	banana	5.25
3	apple	7.00

# Test 7: Filter on computed column
query II
SELECT id, amount * 2 as doubled FROM test WHERE amount * 2 > 15.00 ORDER BY id
----
1	21.00

# Test 8: Empty result
query IVR
SELECT * FROM test WHERE value = 'nonexistent'
----

# Test 9: Filter with type casting
query IVR
SELECT * FROM test WHERE CAST(amount AS INTEGER) = 5
----
2	banana	5.25

# Test 10: Case sensitivity
query IVR
SELECT * FROM test WHERE value = 'APPLE'
----
```

### 9.2 Test Coverage Areas

Ensure your tests cover:

1. **Different data types:**
   ```sql
   -- Integers
   WHERE int_col > 5

   -- Strings
   WHERE str_col = 'value'

   -- Decimals
   WHERE dec_col BETWEEN 1.0 AND 10.0

   -- Dates
   WHERE date_col > DATE '2024-01-01'

   -- Nested types
   WHERE list_col[1] = 5
   WHERE struct_col.field = 'value'
   ```

2. **NULL handling:**
   ```sql
   WHERE col IS NULL
   WHERE col IS NOT NULL
   WHERE col = NULL  -- Should return empty (NULL != NULL)
   ```

3. **Edge cases:**
   ```sql
   -- Empty table
   SELECT * FROM empty_table WHERE id > 5

   -- All rows filtered
   SELECT * FROM table WHERE FALSE

   -- No rows filtered
   SELECT * FROM table WHERE TRUE
   ```

4. **Complex expressions:**
   ```sql
   WHERE (a > 5 AND b < 10) OR (c = 'value' AND d IS NOT NULL)
   WHERE CASE WHEN x > 0 THEN y ELSE z END > 100
   WHERE col IN (SELECT id FROM other_table)
   ```

5. **Error cases:**
   ```sql
   # Type mismatch should error
   statement error
   SELECT * FROM test WHERE int_col = 'not_a_number'
   ----
   <REGEX>:.*Conversion Error.*
   ```

### 9.3 Test Execution

```bash
# Run your test file
build/debug/test/unittest test/sql/filter/test_basic_filter.test

# Run all filter tests
build/debug/test/unittest "test/sql/filter/*"

# Run tests one-by-one to isolate failures
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest test/sql/filter/
```

### 9.4 Writing C++ Tests (Less Preferred)

Only write C++ tests for:
- Internal data structures
- Performance-critical paths
- Component integration testing

**File:** `test/sql/filter/test_filter.cpp`

```cpp
#include "catch.hpp"
#include "test_helpers.hpp"

using namespace duckdb;
using namespace std;

TEST_CASE("Test filter operator", "[filter]") {
	DuckDB db(nullptr);
	Connection con(db);

	// Create table
	REQUIRE_NO_FAIL(con.Query("CREATE TABLE test (a INTEGER, b INTEGER)"));
	REQUIRE_NO_FAIL(con.Query("INSERT INTO test VALUES (1, 10), (2, 20), (3, 30)"));

	// Test basic filter
	auto result = con.Query("SELECT * FROM test WHERE a > 1");
	REQUIRE(CHECK_COLUMN(result, 0, {2, 3}));
	REQUIRE(CHECK_COLUMN(result, 1, {20, 30}));

	// Test empty result
	result = con.Query("SELECT * FROM test WHERE a > 100");
	REQUIRE(result->RowCount() == 0);
}

TEST_CASE("Test filter with NULL values", "[filter]") {
	DuckDB db(nullptr);
	Connection con(db);

	REQUIRE_NO_FAIL(con.Query("CREATE TABLE test (a INTEGER)"));
	REQUIRE_NO_FAIL(con.Query("INSERT INTO test VALUES (1), (NULL), (3)"));

	// NULL should not pass equality filter
	auto result = con.Query("SELECT * FROM test WHERE a = 1");
	REQUIRE(result->RowCount() == 1);

	// IS NULL filter
	result = con.Query("SELECT * FROM test WHERE a IS NULL");
	REQUIRE(result->RowCount() == 1);
}
```

---

## Complete Code Listings

### Logical Operator Files

#### logical_filter.hpp
```cpp
//===----------------------------------------------------------------------===//
//                         DuckDB
//
// duckdb/planner/operator/logical_filter.hpp
//
//===----------------------------------------------------------------------===//

#pragma once

#include "duckdb/planner/logical_operator.hpp"

namespace duckdb {

//! LogicalFilter represents a filter operation (e.g. WHERE or HAVING clause)
class LogicalFilter : public LogicalOperator {
public:
	static constexpr const LogicalOperatorType TYPE = LogicalOperatorType::LOGICAL_FILTER;

public:
	explicit LogicalFilter(unique_ptr<Expression> expression);
	LogicalFilter();

	vector<idx_t> projection_map;

public:
	vector<ColumnBinding> GetColumnBindings() override;

	bool HasProjectionMap() const override {
		return !projection_map.empty();
	}

	void Serialize(Serializer &serializer) const override;
	static unique_ptr<LogicalOperator> Deserialize(Deserializer &deserializer);

	bool SplitPredicates() {
		return SplitPredicates(expressions);
	}
	static bool SplitPredicates(vector<unique_ptr<Expression>> &expressions);

protected:
	void ResolveTypes() override;
};

} // namespace duckdb
```

#### logical_filter.cpp
```cpp
#include "duckdb/planner/operator/logical_filter.hpp"
#include "duckdb/planner/expression/bound_conjunction_expression.hpp"

namespace duckdb {

LogicalFilter::LogicalFilter(unique_ptr<Expression> expression)
    : LogicalOperator(LogicalOperatorType::LOGICAL_FILTER) {
	expressions.push_back(std::move(expression));
	SplitPredicates(expressions);
}

LogicalFilter::LogicalFilter()
    : LogicalOperator(LogicalOperatorType::LOGICAL_FILTER) {
}

void LogicalFilter::ResolveTypes() {
	types = MapTypes(children[0]->types, projection_map);
}

vector<ColumnBinding> LogicalFilter::GetColumnBindings() {
	return MapBindings(children[0]->GetColumnBindings(), projection_map);
}

bool LogicalFilter::SplitPredicates(vector<unique_ptr<Expression>> &expressions) {
	bool found_conjunction = false;
	for (idx_t i = 0; i < expressions.size(); i++) {
		if (expressions[i]->GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
			auto &conjunction = expressions[i]->Cast<BoundConjunctionExpression>();
			found_conjunction = true;
			for (idx_t k = 1; k < conjunction.children.size(); k++) {
				expressions.push_back(std::move(conjunction.children[k]));
			}
			expressions[i] = std::move(conjunction.children[0]);
			i--;
		}
	}
	return found_conjunction;
}

} // namespace duckdb
```

### Physical Operator Files

#### physical_filter.hpp
```cpp
//===----------------------------------------------------------------------===//
//                         DuckDB
//
// duckdb/execution/operator/filter/physical_filter.hpp
//
//===----------------------------------------------------------------------===//

#pragma once

#include "duckdb/execution/physical_operator.hpp"
#include "duckdb/planner/expression.hpp"

namespace duckdb {

//! PhysicalFilter represents a filter operator. It removes non-matching tuples
//! from the result. Note that it does not physically change the data, it only
//! adds a selection vector to the chunk.
class PhysicalFilter : public CachingPhysicalOperator {
public:
	static constexpr const PhysicalOperatorType TYPE = PhysicalOperatorType::FILTER;

public:
	PhysicalFilter(PhysicalPlan &physical_plan, vector<LogicalType> types,
	               vector<unique_ptr<Expression>> select_list,
	               idx_t estimated_cardinality);

	//! The filter expression
	unique_ptr<Expression> expression;

public:
	unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) const override;

	bool ParallelOperator() const override {
		return true;
	}

	InsertionOrderPreservingMap<string> ParamsToString() const override;

protected:
	OperatorResultType ExecuteInternal(ExecutionContext &context, DataChunk &input,
	                                   DataChunk &chunk, GlobalOperatorState &gstate,
	                                   OperatorState &state) const override;
};

} // namespace duckdb
```

#### physical_filter.cpp
```cpp
#include "duckdb/execution/operator/filter/physical_filter.hpp"
#include "duckdb/execution/expression_executor.hpp"
#include "duckdb/planner/expression/bound_conjunction_expression.hpp"
#include "duckdb/parallel/thread_context.hpp"

namespace duckdb {

PhysicalFilter::PhysicalFilter(PhysicalPlan &physical_plan, vector<LogicalType> types,
                               vector<unique_ptr<Expression>> select_list,
                               idx_t estimated_cardinality)
    : CachingPhysicalOperator(physical_plan, PhysicalOperatorType::FILTER,
                             std::move(types), estimated_cardinality) {

	D_ASSERT(!select_list.empty());
	if (select_list.size() == 1) {
		expression = std::move(select_list[0]);
		return;
	}

	// Create a conjunction from the select list
	auto conjunction = make_uniq<BoundConjunctionExpression>(ExpressionType::CONJUNCTION_AND);
	for (auto &expr : select_list) {
		conjunction->children.push_back(std::move(expr));
	}
	expression = std::move(conjunction);
}

class FilterState : public CachingOperatorState {
public:
	explicit FilterState(ExecutionContext &context, Expression &expr)
	    : executor(context.client, expr), sel(STANDARD_VECTOR_SIZE) {
	}

	ExpressionExecutor executor;
	SelectionVector sel;

public:
	void Finalize(const PhysicalOperator &op, ExecutionContext &context) override {
		context.thread.profiler.Flush(op);
	}
};

unique_ptr<OperatorState> PhysicalFilter::GetOperatorState(ExecutionContext &context) const {
	return make_uniq<FilterState>(context, *expression);
}

OperatorResultType PhysicalFilter::ExecuteInternal(ExecutionContext &context,
                                                   DataChunk &input,
                                                   DataChunk &chunk,
                                                   GlobalOperatorState &gstate,
                                                   OperatorState &state_p) const {
	auto &state = state_p.Cast<FilterState>();
	idx_t result_count = state.executor.SelectExpression(input, state.sel);
	if (result_count == input.size()) {
		// Nothing was filtered: skip adding any selection vectors
		chunk.Reference(input);
	} else {
		chunk.Slice(input, state.sel, result_count);
	}
	return OperatorResultType::NEED_MORE_INPUT;
}

InsertionOrderPreservingMap<string> PhysicalFilter::ParamsToString() const {
	InsertionOrderPreservingMap<string> result;
	result["__expression__"] = expression->GetName();
	SetEstimatedCardinality(result, estimated_cardinality);
	return result;
}

} // namespace duckdb
```

### Physical Plan Generator

#### plan_filter.cpp
```cpp
#include "duckdb/execution/operator/filter/physical_filter.hpp"
#include "duckdb/execution/operator/projection/physical_projection.hpp"
#include "duckdb/execution/physical_plan_generator.hpp"
#include "duckdb/planner/expression/bound_reference_expression.hpp"
#include "duckdb/planner/operator/logical_filter.hpp"

namespace duckdb {

PhysicalOperator &PhysicalPlanGenerator::CreatePlan(LogicalFilter &op) {
	D_ASSERT(op.children.size() == 1);
	reference<PhysicalOperator> plan = CreatePlan(*op.children[0]);

	if (!op.expressions.empty()) {
		D_ASSERT(!plan.get().GetTypes().empty());
		auto &filter = Make<PhysicalFilter>(plan.get().GetTypes(),
		                                    std::move(op.expressions),
		                                    op.estimated_cardinality);
		filter.children.push_back(plan);
		plan = filter;
	}

	if (op.HasProjectionMap()) {
		vector<unique_ptr<Expression>> select_list;
		for (idx_t i = 0; i < op.projection_map.size(); i++) {
			select_list.push_back(
				make_uniq<BoundReferenceExpression>(op.types[i], op.projection_map[i])
			);
		}
		auto &proj = Make<PhysicalProjection>(op.types, std::move(select_list),
		                                      op.estimated_cardinality);
		proj.children.push_back(plan);
		plan = proj;
	}

	return plan;
}

} // namespace duckdb
```

---

## Summary and Next Steps

You've now learned how to add a complete operator to DuckDB! Here's what we covered:

### Key Takeaways

1. **Two-Phase Design:** Logical operators (planning) → Physical operators (execution)
2. **Vectorized Execution:** Process data in chunks for performance
3. **State Management:** Per-thread state for parallel execution
4. **Zero-Copy Optimization:** Use selection vectors instead of copying data
5. **Test-Driven Development:** Comprehensive tests ensure correctness

### Operator Implementation Checklist

- [ ] Define logical operator (header + implementation)
- [ ] Register logical operator type in enum
- [ ] Update binder to create logical operator
- [ ] Define physical operator (header + implementation)
- [ ] Register physical operator type in enum
- [ ] Implement execution logic (ExecuteInternal)
- [ ] Implement state management (GetOperatorState)
- [ ] Create physical plan generator (CreatePlan)
- [ ] Add optimizer support (optional but recommended)
- [ ] Update build system (CMakeLists.txt)
- [ ] Write comprehensive tests (.test files)
- [ ] Run all tests (make allunit)
- [ ] Format code (make format-fix)

### More Complex Operators

For more complex operators, consider:

1. **Blocking Operators (e.g., ORDER BY, GROUP BY):**
   - Implement `IsSink() = true` and `IsSource() = true`
   - Implement `Sink()` to consume input
   - Implement `GetData()` to produce output
   - Use `GlobalSinkState` for accumulated data

2. **Join Operators:**
   - Build hash table in sink phase
   - Probe hash table in operator phase
   - Handle multiple join types (INNER, LEFT, RIGHT, FULL)

3. **Window Functions:**
   - Partition data
   - Sort within partitions
   - Compute window aggregates

### Further Reading

- **Architecture Deep Dive:** `learning/architecture-deep-dive.md`
- **Vectorized Execution:** `learning/vectorized-execution.md`
- **Testing Guide:** `learning/testing-guide.md`
- **Existing Operators:** Browse `src/execution/operator/` for examples

### Contributing

Before submitting a PR:
1. Run `make format-fix` to format code
2. Run `make allunit` to ensure all tests pass
3. Add comprehensive test coverage
4. Keep PRs focused and well-documented
5. Follow DuckDB coding guidelines (CLAUDE.md)

Good luck with your operator implementation!
