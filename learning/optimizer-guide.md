# DuckDB Optimizer Guide for Developers

## Table of Contents
1. [Optimizer Architecture Overview](#optimizer-architecture-overview)
2. [Logical Plan Optimization](#logical-plan-optimization)
3. [Rule-Based Optimizations](#rule-based-optimizations)
4. [Cost-Based Optimizations](#cost-based-optimizations)
5. [Join Ordering](#join-ordering)
6. [Predicate Pushdown](#predicate-pushdown)
7. [Expression Rewriting](#expression-rewriting)
8. [Statistics and Cardinality Estimation](#statistics-and-cardinality-estimation)
9. [How to Add New Optimization Rules](#how-to-add-new-optimization-rules)
10. [Debugging Optimizer Decisions](#debugging-optimizer-decisions)
11. [Common Optimization Patterns](#common-optimization-patterns)
12. [Case Studies](#case-studies)

---

## Optimizer Architecture Overview

The DuckDB optimizer transforms logical query plans into more efficient equivalent plans. The optimizer is located in `src/optimizer/` and works on the logical plan produced by the planner.

### Key Components

**Main Optimizer Class** (`src/optimizer/optimizer.cpp`, `src/include/duckdb/optimizer/optimizer.hpp`)

The `Optimizer` class is the entry point for all optimization passes:

```cpp
class Optimizer {
public:
    Optimizer(Binder &binder, ClientContext &context);
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan);

private:
    ClientContext &context;
    Binder &binder;
    ExpressionRewriter rewriter;
    unique_ptr<LogicalOperator> plan;
};
```

### Optimization Pipeline

The optimizer runs optimizations in a specific sequence defined in `RunBuiltInOptimizers()` (`src/optimizer/optimizer.cpp:105-297`):

1. **Expression Rewriter** - Simplifies expressions without changing plan structure
2. **CTE Inlining** - Decides whether to inline or materialize CTEs
3. **Common Subplan Optimizer** - Converts repeated subplans into materialized CTEs
4. **Sum Rewriter** - Rewrites `SUM(x + C)` into `SUM(x) + C * COUNT(x)`
5. **Filter Pullup** - Pulls filters up through the plan tree
6. **Filter Pushdown** - Pushes filters down to reduce intermediate results
7. **CTE Filter Pusher** - Derives and pushes filters into materialized CTEs
8. **Regex Range Filter** - Optimizes regex patterns into range filters
9. **IN Clause Rewriter** - Optimizes IN clauses
10. **Deliminator** - Removes redundant DelimGets/DelimJoins
11. **Empty Result Pullup** - Propagates empty results upward
12. **Join Order Optimizer** - Reorders joins for optimal execution
13. **Unnest Rewriter** - Rewrites UNNESTs in DelimJoins
14. **Remove Unused Columns** - Eliminates columns not needed for query result
15. **Remove Duplicate Groups** - Removes duplicate grouping expressions
16. **Common Subexpressions** - Extracts repeated subexpressions
17. **Column Lifetime Analyzer** - Creates projection maps for early column elimination
18. **Build/Probe Side Optimizer** - Determines optimal build/probe sides for joins
19. **Limit Pushdown** - Pushes LIMIT below PROJECTION
20. **Sampling Pushdown** - Pushes SAMPLE operations down
21. **Top-N Optimizer** - Transforms ORDER BY + LIMIT to TopN
22. **Late Materialization** - Delays column materialization when beneficial
23. **Statistics Propagation** - Propagates statistics through the plan
24. **Top-N Window Elimination** - Rewrites row_number window + filter to aggregate
25. **Common Aggregate Optimizer** - Removes duplicate aggregates
26. **Expression Heuristics** - Reorders filter expressions using heuristics
27. **Join Filter Pushdown** - Pushes join filters after initial optimization

Each optimizer can be selectively disabled using the `disabled_optimizers` configuration setting.

---

## Logical Plan Optimization

### Logical Operators

The optimizer works on a tree of `LogicalOperator` nodes (`src/include/duckdb/planner/logical_operator.hpp`). Each operator represents a relational operation:

- `LOGICAL_GET` - Table scan
- `LOGICAL_FILTER` - Selection predicate
- `LOGICAL_PROJECTION` - Column projection
- `LOGICAL_AGGREGATE_AND_GROUP_BY` - Aggregation with grouping
- `LOGICAL_COMPARISON_JOIN` - Join with comparison predicates
- `LOGICAL_ORDER_BY` - Sort operation
- `LOGICAL_LIMIT` - Limit/offset operation
- And many more...

### Optimization Flow

```
Parsed SQL → Binder → Logical Plan → Optimizer → Optimized Logical Plan → Physical Plan → Execution
```

The planner (`src/planner/planner.cpp`) creates the initial logical plan through binding, then calls the optimizer:

```cpp
void Planner::CreatePlan(SQLStatement &statement) {
    // Bind the statement to create logical plan
    auto bound_statement = binder->Bind(statement);
    this->plan = std::move(bound_statement.plan);

    // Decorrelate independent subqueries
    this->plan = FlattenDependentJoins::DecorrelateIndependent(*this->binder, std::move(this->plan));

    // Verify the plan
    Planner::VerifyPlan(context, plan, bound_parameters.GetParametersPtr());
}
```

Later, the optimizer is invoked to transform this plan.

---

## Rule-Based Optimizations

Rule-based optimizations apply pattern-matching transformations that are always beneficial. They don't require cost estimation.

### Expression Rewriting Rules

The `ExpressionRewriter` (`src/optimizer/expression_rewriter.cpp`) applies a set of rules to simplify expressions. Rules are registered in the `Optimizer` constructor (`src/optimizer/optimizer.cpp:44-64`):

```cpp
Optimizer::Optimizer(Binder &binder, ClientContext &context) : context(context), binder(binder), rewriter(context) {
    rewriter.rules.push_back(make_uniq<ConstantFoldingRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DistributivityRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ArithmeticSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<CaseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ConjunctionSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DatePartSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<DateTruncSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<ComparisonSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<InClauseSimplificationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EqualOrNullSimplification>(rewriter));
    rewriter.rules.push_back(make_uniq<MoveConstantsRule>(rewriter));
    rewriter.rules.push_back(make_uniq<LikeOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<OrderedAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctAggregateOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<DistinctWindowedOptimizer>(rewriter));
    rewriter.rules.push_back(make_uniq<RegexOptimizationRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EmptyNeedleRemovalRule>(rewriter));
    rewriter.rules.push_back(make_uniq<EnumComparisonRule>(rewriter));
    rewriter.rules.push_back(make_uniq<JoinDependentFilterRule>(rewriter));
    rewriter.rules.push_back(make_uniq<TimeStampComparison>(context, rewriter));
}
```

### Major Expression Rules

#### 1. Constant Folding (`src/optimizer/rule/constant_folding.cpp`)

Evaluates constant expressions at compile time:

**Before:**
```sql
WHERE x > 5 + 3
```

**After:**
```sql
WHERE x > 8
```

Implementation:
```cpp
unique_ptr<Expression> ConstantFoldingRule::Apply(LogicalOperator &op, vector<reference<Expression>> &bindings,
                                                  bool &changes_made, bool is_root) {
    auto &root = bindings[0].get();
    D_ASSERT(root.IsFoldable() && root.GetExpressionType() != ExpressionType::VALUE_CONSTANT);

    // Use ExpressionExecutor to evaluate the expression
    Value result_value;
    if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), root, result_value)) {
        return nullptr;
    }
    // Return a constant expression with the computed value
    return make_uniq<BoundConstantExpression>(result_value);
}
```

#### 2. Distributivity Rule (`src/optimizer/rule/distributivity.cpp`)

Extracts common expressions from OR branches using the distributive law:

**Before:**
```sql
WHERE (x > 5 AND y = 10) OR (x > 5 AND z = 20)
```

**After:**
```sql
WHERE x > 5 AND (y = 10 OR z = 20)
```

This allows the common predicate `x > 5` to be evaluated once and potentially pushed down.

#### 3. Arithmetic Simplification

Simplifies arithmetic expressions:

**Before:**
```sql
WHERE x + 0 = 5
WHERE x * 1 = 5
WHERE x - 0 = 5
```

**After:**
```sql
WHERE x = 5
WHERE x = 5
WHERE x = 5
```

#### 4. Comparison Simplification

Simplifies comparison expressions:

**Before:**
```sql
WHERE NOT (x > 5)
```

**After:**
```sql
WHERE x <= 5
```

### Rule Implementation Pattern

All rules inherit from the `Rule` base class (`src/include/duckdb/optimizer/rule.hpp`):

```cpp
class Rule {
public:
    explicit Rule(ExpressionRewriter &rewriter) : rewriter(rewriter) {}
    virtual ~Rule() {}

    ExpressionRewriter &rewriter;
    unique_ptr<ExpressionMatcher> root;  // Pattern to match

    ClientContext &GetContext() const;
    virtual unique_ptr<Expression> Apply(LogicalOperator &op,
                                        vector<reference<Expression>> &bindings,
                                        bool &fixed_point,
                                        bool is_root) = 0;
};
```

Rules use **expression matchers** to identify applicable patterns, then transform matched expressions.

---

## Cost-Based Optimizations

Cost-based optimizations use statistics and cardinality estimates to choose between multiple valid plans.

### Join Order Optimization

The most significant cost-based optimization is join ordering, which determines the sequence and method for executing joins.

**Location:** `src/optimizer/join_order/`

**Key Files:**
- `join_order_optimizer.cpp` - Main join order optimization logic
- `cardinality_estimator.cpp` - Estimates result cardinality
- `cost_model.cpp` - Estimates execution cost
- `plan_enumerator.cpp` - Enumerates possible join orders
- `query_graph.cpp` - Represents joins as a query graph

### Cost Model

The cost model estimates the execution cost of different join orders. It considers:

1. **Cardinality** - Number of rows produced
2. **Join method** - Hash join vs. nested loop join
3. **Build/probe side** - Which table to use as build vs. probe
4. **Available statistics** - Column statistics, distinct counts, min/max values

---

## Join Ordering

Join ordering is one of the most critical optimizations for query performance.

### Query Graph Representation

DuckDB uses a **query graph** approach where:
- Nodes represent base relations (tables)
- Edges represent join conditions
- Filters are associated with nodes

### Join Order Algorithm

The join order optimizer (`src/optimizer/join_order/join_order_optimizer.cpp`) uses a **dynamic programming** approach:

```cpp
unique_ptr<LogicalOperator> JoinOrderOptimizer::Optimize(unique_ptr<LogicalOperator> plan,
                                                         optional_ptr<RelationStats> stats) {
    LogicalOperator *op = plan.get();

    // Extract relations and build query graph
    bool reorderable = query_graph_manager.Build(*this, *op);

    if (reorderable) {
        // Initialize cost model
        auto cost_model = CostModel(query_graph_manager);

        // Initialize plan enumerator
        auto plan_enumerator = PlanEnumerator(query_graph_manager, cost_model,
                                             query_graph_manager.GetQueryGraphEdges());

        // Initialize leaf/single node plans
        plan_enumerator.InitLeafPlans();

        // Solve join order using dynamic programming
        plan_enumerator.SolveJoinOrder();

        // Reconstruct logical plan from optimal join order
        query_graph_manager.plans = &plan_enumerator.GetPlans();
        new_logical_plan = query_graph_manager.Reconstruct(std::move(plan));
    }

    return new_logical_plan;
}
```

### Cardinality Estimation

Cardinality estimation is crucial for join ordering (`src/optimizer/join_order/cardinality_estimator.cpp`):

**Formula:**
```
Cardinality(R ⋈ S) = |R| × |S| / max(distinct(R.key), distinct(S.key))
```

The estimator tracks:
- **Total domain (TDom)** - Distinct values in join columns
- **Equivalence sets** - Columns with the same distinct count due to equi-joins
- **Filter selectivity** - How much filters reduce cardinality

**Example:**
```cpp
double CardinalityEstimator::EstimateCardinalityWithSet(JoinRelationSet &new_set) {
    if (relation_set_2_cardinality.find(new_set.ToString()) != relation_set_2_cardinality.end()) {
        return relation_set_2_cardinality[new_set.ToString()].cardinality_before_filters;
    }

    auto denom = GetDenominator(new_set);
    auto numerator = GetNumerator(denom.numerator_relations);

    double result = numerator / denom.denominator;
    return result;
}
```

### Join Order Example

**Query:**
```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
WHERE c.country = 'USA' AND p.price > 100;
```

**Unoptimized Plan:**
```
JOIN (orders, customers)  -- 1M rows
  JOIN products           -- 10K rows
```

**Optimized Plan:**
```
JOIN (customers WHERE country='USA', products WHERE price>100)  -- 100 rows
  JOIN orders                                                    -- 1K rows
```

The optimizer:
1. Pushes filters down to base tables
2. Estimates cardinality after filtering
3. Chooses to join smaller filtered tables first
4. Selects hash join for equi-joins

---

## Predicate Pushdown

Predicate pushdown moves filter operations closer to data sources to reduce the amount of data processed.

**Location:** `src/optimizer/filter_pushdown.cpp`

### Filter Pushdown Algorithm

The `FilterPushdown` optimizer traverses the logical plan top-down, pushing filters through operators:

```cpp
unique_ptr<LogicalOperator> FilterPushdown::Rewrite(unique_ptr<LogicalOperator> op) {
    D_ASSERT(!combiner.HasFilters());
    switch (op->type) {
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        return PushdownAggregate(std::move(op));
    case LogicalOperatorType::LOGICAL_FILTER:
        return PushdownFilter(std::move(op));
    case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
    case LogicalOperatorType::LOGICAL_ANY_JOIN:
    case LogicalOperatorType::LOGICAL_ASOF_JOIN:
    case LogicalOperatorType::LOGICAL_DELIM_JOIN:
        return PushdownJoin(std::move(op));
    case LogicalOperatorType::LOGICAL_PROJECTION:
        return PushdownProjection(std::move(op));
    case LogicalOperatorType::LOGICAL_GET:
        return PushdownGet(std::move(op));
    // ... other cases
    default:
        return FinishPushdown(std::move(op));
    }
}
```

### Filter Pushdown Rules

#### 1. Pushdown Through Projection

Filters can be pushed through projections by rewriting column references:

**Before:**
```
FILTER (projected_col > 5)
  PROJECTION (base_col AS projected_col)
    SCAN table
```

**After:**
```
PROJECTION (base_col AS projected_col)
  FILTER (base_col > 5)
    SCAN table
```

#### 2. Pushdown Into Joins

For inner joins, filters can be pushed to the appropriate side:

**Before:**
```
FILTER (t1.x > 5 AND t2.y < 10)
  JOIN (t1, t2) ON t1.id = t2.id
```

**After:**
```
JOIN ON t1.id = t2.id
  FILTER (t1.x > 5)
    SCAN t1
  FILTER (t2.y < 10)
    SCAN t2
```

For outer joins, pushdown is more restricted to preserve semantics.

#### 3. Pushdown Into Table Scans

Filters pushed all the way to table scans can be evaluated during scanning:

**Example:**
```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownGet(unique_ptr<LogicalOperator> op) {
    // Filters are pushed into the LogicalGet operator's table_filters
    // These can be evaluated during the scan, potentially using indexes
}
```

### Filter Combiner

The `FilterCombiner` (`src/include/duckdb/optimizer/filter_combiner.hpp`) combines and simplifies multiple filters:

- Eliminates redundant filters
- Detects contradictions (unsatisfiable filters)
- Combines overlapping range predicates
- Converts predicates to more efficient forms

**Example:**
```sql
-- Input filters
WHERE x > 5 AND x > 10 AND x < 20

-- Combined filter
WHERE x > 10 AND x < 20
```

---

## Expression Rewriting

Expression rewriting simplifies and optimizes expressions without changing the logical plan structure.

**Location:** `src/optimizer/expression_rewriter.cpp`

### Rewriting Process

The rewriter applies rules iteratively until a fixed point:

```cpp
void ExpressionRewriter::VisitExpression(unique_ptr<Expression> *expression) {
    bool changes_made;
    do {
        changes_made = false;
        *expression = ExpressionRewriter::ApplyRules(*op, to_apply_rules,
                                                     std::move(*expression),
                                                     changes_made, true);
    } while (changes_made);
}
```

### Common Subexpression Elimination (CSE)

CSE identifies and eliminates repeated subexpressions (`src/optimizer/cse_optimizer.cpp`):

**Before:**
```sql
SELECT (a + b) * 2, (a + b) * 3, (a + b) + 5
FROM table;
```

**After:**
```sql
SELECT cse_0 * 2, cse_0 * 3, cse_0 + 5
FROM (
  SELECT a + b AS cse_0, *
  FROM table
);
```

**Implementation:**
```cpp
void CommonSubExpressionOptimizer::ExtractCommonSubExpresions(LogicalOperator &op) {
    CSEReplacementState state;

    // Count occurrences of each expression
    LogicalOperatorVisitor::EnumerateExpressions(
        op, [&](unique_ptr<Expression> *child) { CountExpressions(**child, state); });

    // Check if any expressions occur more than once
    bool perform_replacement = false;
    for (auto &expr : state.expression_count) {
        if (expr.second.count > 1) {
            perform_replacement = true;
            break;
        }
    }

    if (perform_replacement) {
        // Replace repeated expressions with column references
        LogicalOperatorVisitor::EnumerateExpressions(
            op, [&](unique_ptr<Expression> *child) { PerformCSEReplacement(*child, state); });

        // Create projection node with extracted expressions
        auto projection = make_uniq<LogicalProjection>(state.projection_index, std::move(state.expressions));
        projection->children.push_back(std::move(op.children[0]));
        op.children[0] = std::move(projection);
    }
}
```

### Expression Heuristics

The `ExpressionHeuristics` optimizer (`src/include/duckdb/optimizer/expression_heuristics.hpp`) reorders filter expressions to evaluate cheaper/more selective filters first:

**Heuristic Order:**
1. Comparisons with constants (cheap, often selective)
2. Column comparisons
3. Function calls (potentially expensive)
4. Complex expressions

---

## Statistics and Cardinality Estimation

Statistics enable better optimization decisions.

**Location:** `src/optimizer/statistics_propagator.cpp`

### Statistics Propagation

The `StatisticsPropagator` walks the logical plan and propagates statistics:

```cpp
unique_ptr<NodeStatistics> StatisticsPropagator::PropagateStatistics(LogicalOperator &node,
                                                                     unique_ptr<LogicalOperator> &node_ptr) {
    unique_ptr<NodeStatistics> result;
    switch (node.type) {
    case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
        result = PropagateStatistics(node.Cast<LogicalAggregate>(), node_ptr);
        break;
    case LogicalOperatorType::LOGICAL_FILTER:
        result = PropagateStatistics(node.Cast<LogicalFilter>(), node_ptr);
        break;
    case LogicalOperatorType::LOGICAL_GET:
        result = PropagateStatistics(node.Cast<LogicalGet>(), node_ptr);
        break;
    // ... handle other operator types
    }
    return result;
}
```

### Types of Statistics

1. **Base Statistics**
   - Min/max values
   - Null count
   - Distinct count (exact or HyperLogLog estimate)
   - Data type information

2. **Operator Statistics**
   - Cardinality (row count)
   - Column-specific statistics

3. **Expression Statistics**
   - Value ranges after computation
   - Null propagation

### Using Statistics

Statistics are used for:

1. **Filter selectivity estimation**
   ```cpp
   // x > 5 on column with min=0, max=100, distinct=100
   // Selectivity = (100 - 5) / 100 = 0.95
   ```

2. **Join cardinality estimation**
   ```cpp
   // R.x = S.x, distinct(R.x) = 100, distinct(S.x) = 50
   // Cardinality = |R| * |S| / max(100, 50)
   ```

3. **Detecting empty results**
   - If filter range doesn't overlap with column min/max, result is empty

4. **Optimizing aggregates**
   - If GROUP BY columns have known cardinality, estimate aggregate result size

---

## How to Add New Optimization Rules

### Step 1: Create Rule Class

Create a new rule file in `src/optimizer/rule/`:

```cpp
// src/optimizer/rule/my_new_rule.cpp
#include "duckdb/optimizer/rule/my_new_rule.hpp"
#include "duckdb/optimizer/matcher/expression_matcher.hpp"
#include "duckdb/planner/expression/bound_function_expression.hpp"

namespace duckdb {

MyNewRule::MyNewRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    // Define the pattern to match
    // Example: Match any function call
    root = make_uniq<ExpressionMatcher>();
    root->expr_class = make_uniq<SpecificExpressionClassMatcher>(ExpressionClass::BOUND_FUNCTION);
}

unique_ptr<Expression> MyNewRule::Apply(LogicalOperator &op,
                                       vector<reference<Expression>> &bindings,
                                       bool &changes_made,
                                       bool is_root) {
    auto &func_expr = bindings[0].get().Cast<BoundFunctionExpression>();

    // Check if this is the specific function we want to optimize
    if (func_expr.function.name != "my_function") {
        return nullptr;
    }

    // Apply the transformation
    // Return the new expression or nullptr if no change
    return transformed_expr;
}

} // namespace duckdb
```

### Step 2: Create Header File

```cpp
// src/include/duckdb/optimizer/rule/my_new_rule.hpp
#pragma once

#include "duckdb/optimizer/rule.hpp"

namespace duckdb {

class MyNewRule : public Rule {
public:
    explicit MyNewRule(ExpressionRewriter &rewriter);

    unique_ptr<Expression> Apply(LogicalOperator &op,
                                 vector<reference<Expression>> &bindings,
                                 bool &changes_made,
                                 bool is_root) override;
};

} // namespace duckdb
```

### Step 3: Register the Rule

Add to the optimizer constructor in `src/optimizer/optimizer.cpp`:

```cpp
Optimizer::Optimizer(Binder &binder, ClientContext &context)
    : context(context), binder(binder), rewriter(context) {
    // ... existing rules ...
    rewriter.rules.push_back(make_uniq<MyNewRule>(rewriter));
}
```

### Step 4: Add Tests

Create tests in `test/sql/optimizer/` or add C++ unit tests:

```sql
# test/sql/optimizer/test_my_new_rule.test

name test_my_new_rule
description Test the new optimization rule

statement ok
CREATE TABLE test (x INTEGER, y INTEGER);

statement ok
INSERT INTO test VALUES (1, 2), (3, 4), (5, 6);

query I
SELECT my_function(x, y) FROM test;
----
(expected results)
```

### Step 5: Expression Matchers

Use expression matchers to define patterns:

```cpp
// Match specific expression type
root->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::COMPARE_EQUAL);

// Match specific function
root->function = make_uniq<SpecificFunctionMatcher>("substring");

// Match expression class
root->expr_class = make_uniq<SpecificExpressionClassMatcher>(ExpressionClass::BOUND_CAST);

// Match with children
auto child = make_uniq<ExpressionMatcher>();
child->expr_type = make_uniq<SpecificExpressionTypeMatcher>(ExpressionType::VALUE_CONSTANT);
root->matchers.push_back(std::move(child));
```

---

## Debugging Optimizer Decisions

### 1. EXPLAIN Statement

Use `EXPLAIN` to see the logical plan:

```sql
EXPLAIN SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t1.x > 5;
```

### 2. EXPLAIN ANALYZE

See both the plan and execution statistics:

```sql
EXPLAIN ANALYZE SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t1.x > 5;
```

### 3. Disable Specific Optimizers

Disable optimizers to understand their impact:

```sql
-- Disable join order optimization
SET disabled_optimizers = 'join_order';

-- Disable multiple optimizers
SET disabled_optimizers = 'join_order,filter_pushdown';
```

The `OptimizerType` enum defines all optimizer types that can be disabled.

### 4. Query Profiler

Enable query profiling for detailed metrics:

```sql
PRAGMA enable_profiling = 'json';
PRAGMA profiling_output = '/tmp/profile.json';
SELECT * FROM t1 JOIN t2 ON t1.id = t2.id;
```

The profile includes:
- Time spent in each optimizer phase
- Cardinality estimates vs. actual
- Operator execution times

### 5. Debug Logging

Enable verbose logging in debug builds:

```cpp
// In your optimizer code
#ifdef DEBUG
D_ASSERT(condition);  // Assertions for invariants
#endif
```

### 6. Verify Plan Integrity

The optimizer automatically verifies plan integrity:

```cpp
void Optimizer::Verify(LogicalOperator &op) {
    ColumnBindingResolver::Verify(op);
}
```

This catches:
- Invalid column bindings
- Mismatched types
- Broken operator trees

---

## Common Optimization Patterns

### Pattern 1: Operator Pushdown

Push operations closer to data sources:

```cpp
// Push LIMIT through PROJECTION
LIMIT(10)
  PROJECTION(a, b)
    SCAN(table)

// Becomes:
PROJECTION(a, b)
  LIMIT(10)
    SCAN(table)
```

This reduces the number of rows projected.

### Pattern 2: Operator Fusion

Combine multiple operators into one:

```cpp
// ORDER BY + LIMIT → TopN
ORDER BY x
  LIMIT 10
    SCAN(table)

// Becomes:
TOP_N(10, x)
  SCAN(table)
```

TopN is more efficient than full sort + limit.

### Pattern 3: Predicate Elimination

Remove redundant predicates:

```cpp
// x > 5 AND x > 10 → x > 10
// x > 10 AND x < 5 → FALSE (empty result)
```

### Pattern 4: Late Materialization

Delay reading columns until needed:

```cpp
// Before: Read all columns early
PROJECTION(a, b)
  FILTER(c > 5)
    SCAN(table) [read a, b, c]

// After: Read only filter columns, then project
PROJECTION(a, b)
  SCAN(table) [read a, b]
    WHERE id IN (
      SELECT id FROM SCAN(table) [read c, id] WHERE c > 5
    )
```

### Pattern 5: Index-Based Optimization

Use indexes when available:

```cpp
// Filter can use index if available
FILTER(x = 5)
  SCAN(table)

// Optimizer checks for index on column x
// If available, converts to index scan
```

---

## Case Studies

### Case Study 1: Top-N Optimization

**Query:**
```sql
SELECT * FROM large_table ORDER BY score DESC LIMIT 10;
```

**Unoptimized Plan:**
```
LIMIT(10)
  ORDER_BY(score DESC)
    SCAN(large_table)  -- 1M rows
```

**Problem:** Full sort of 1M rows when we only need top 10.

**Optimized Plan:**
```
TOP_N(10, score DESC)
  SCAN(large_table)
```

**Optimization:** The `TopN` optimizer (`src/optimizer/topn_optimizer.cpp`) detects ORDER BY + LIMIT patterns:

```cpp
bool TopN::CanOptimize(LogicalOperator &op, optional_ptr<ClientContext> context) {
    if (op.type == LogicalOperatorType::LOGICAL_LIMIT) {
        auto &limit = op.Cast<LogicalLimit>();

        // Need constant LIMIT value
        if (limit.limit_val.Type() != LimitNodeType::CONSTANT_VALUE) {
            return false;
        }

        // Check if child is ORDER BY
        auto child_op = op.children[0].get();
        while (child_op->type == LogicalOperatorType::LOGICAL_PROJECTION) {
            child_op = child_op->children[0].get();
        }

        return child_op->type == LogicalOperatorType::LOGICAL_ORDER_BY;
    }
    return false;
}
```

**Benefit:** Top-N uses a priority queue (size = 10) instead of sorting all 1M rows. Complexity: O(n log k) vs O(n log n).

**Further Optimization:** Top-N can push dynamic filters down to table scans to prune rows early.

### Case Study 2: Join Reordering with Filters

**Query:**
```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN line_items l ON o.id = l.order_id
WHERE c.country = 'USA';
```

**Initial Plan:**
```
JOIN (orders, customers)      -- 10M * 1M = 10T intermediate rows
  JOIN line_items              -- 10T * 50M = 500T rows
    FILTER (country = 'USA')   -- Final: 1M rows
```

**Optimizations Applied:**

1. **Filter Pushdown:** Push `country = 'USA'` to customers table
   ```
   FILTER (country = 'USA')
     SCAN customers  -- 1M → 10K rows
   ```

2. **Cardinality Estimation:** Estimate join result sizes
   - customers (filtered): 10K rows
   - orders: 10M rows
   - line_items: 50M rows

3. **Join Reordering:** Choose optimal join order
   ```
   JOIN (
     JOIN (
       SCAN customers WHERE country='USA',  -- 10K rows
       SCAN orders                          -- 10M rows
     ),                                     -- Result: 100K rows
     SCAN line_items                        -- 50M rows
   )                                        -- Result: 500K rows
   ```

**Optimized Plan:**
```
JOIN line_items ON o.id = l.order_id
  JOIN customers ON o.customer_id = c.id
    SCAN customers WHERE country = 'USA'  -- 10K rows
    SCAN orders                           -- 10M rows
```

**Benefit:** Reduces intermediate result size from 10T rows to 100K rows.

### Case Study 3: Common Subexpression Elimination

**Query:**
```sql
SELECT
  (price * 1.1) AS price_with_tax,
  (price * 1.1) * quantity AS total_with_tax,
  CASE WHEN (price * 1.1) > 100 THEN 'expensive' ELSE 'cheap' END
FROM products;
```

**Unoptimized Plan:**
- `price * 1.1` is computed 3 times per row

**Optimized Plan:**
```sql
SELECT
  cse_0 AS price_with_tax,
  cse_0 * quantity AS total_with_tax,
  CASE WHEN cse_0 > 100 THEN 'expensive' ELSE 'cheap' END
FROM (
  SELECT price * 1.1 AS cse_0, quantity, *
  FROM products
);
```

**Benefit:** `price * 1.1` is computed once and reused.

**Implementation:** The CSE optimizer identifies expressions that occur multiple times and extracts them into a projection.

### Case Study 4: Filter Combination and Simplification

**Query:**
```sql
SELECT * FROM table
WHERE x > 5 AND x > 10 AND x < 100 AND x IS NOT NULL;
```

**Optimization Steps:**

1. **Redundancy Elimination:** `x > 5` is redundant given `x > 10`
2. **NULL Handling:** `x > 10` implies `x IS NOT NULL`
3. **Range Combination:** Combine into single range check

**Optimized Filter:**
```sql
WHERE x > 10 AND x < 100
```

**Further:** If table statistics show `min(x) = 50, max(x) = 90`:
- Remove `x > 10` (always true)
- Keep `x < 100` (always true, but keep for safety)
- Effective filter: scan all rows

If statistics show `min(x) = 200`:
- Filter is unsatisfiable
- Replace entire plan with `LOGICAL_EMPTY_RESULT`

### Case Study 5: Distributivity Rule Application

**Query:**
```sql
SELECT * FROM table
WHERE (status = 'active' AND score > 80)
   OR (status = 'active' AND priority = 'high');
```

**Before Optimization:**
```
OR(
  AND(status = 'active', score > 80),
  AND(status = 'active', priority = 'high')
)
```

**After Distributivity Rule:**
```
AND(
  status = 'active',
  OR(score > 80, priority = 'high')
)
```

**Benefits:**
1. `status = 'active'` evaluated once instead of twice
2. `status = 'active'` can be pushed down to table scan
3. Remaining OR filter is simpler

**Implementation:** The `DistributivityRule` extracts common expressions from OR branches using set intersection.

---

## Performance Tips

### 1. Enable Statistics Collection

```sql
ANALYZE table_name;
```

Better statistics lead to better optimization decisions.

### 2. Use Appropriate Data Types

- Use `INTEGER` instead of `VARCHAR` for numeric data
- Use `DATE` instead of `VARCHAR` for dates
- Smaller types reduce memory and improve cache efficiency

### 3. Provide Hints Through Query Structure

DuckDB doesn't support query hints, but query structure matters:

```sql
-- Prefer this (explicit predicates)
WHERE x > 5 AND y < 10

-- Over this (function that might prevent optimization)
WHERE my_complex_function(x, y) = true
```

### 4. Monitor Query Plans

Regularly check query plans to understand optimizer decisions:

```sql
EXPLAIN SELECT ...;
```

### 5. Leverage Indexes (When Available)

DuckDB automatically uses indexes when beneficial. Ensure indexes exist on:
- Join columns
- Filter columns
- Sort columns (for ORDER BY)

---

## Advanced Topics

### Extension Points

DuckDB allows custom optimizers through extensions:

```cpp
struct OptimizerExtension {
    optimizer_function_t pre_optimize_function;
    optimizer_function_t optimize_function;
    unique_ptr<OptimizerExtensionInfo> optimizer_info;
};
```

Extensions can inject custom optimizations before or after built-in optimizers.

### Verification and Testing

The optimizer includes built-in verification:

```cpp
void Planner::VerifyPlan(ClientContext &context, unique_ptr<LogicalOperator> &op,
                         optional_ptr<bound_parameter_map_t> map) {
    // Verify column bindings
    ColumnBindingResolver::Verify(*op);

    // Test serialization/deserialization
    if (ClientConfig::GetConfig(context).verify_serializer) {
        // Serialize and deserialize the plan to ensure it's valid
    }
}
```

### Compressed Materialization

The optimizer can choose compressed representations for materialized data based on statistics:

- If a column has few distinct values, use dictionary encoding
- If a column has a small range, use bit-packing
- Statistics inform these decisions

---

## Summary

The DuckDB optimizer is a sophisticated system that combines:

1. **Rule-based optimizations** for always-beneficial transformations
2. **Cost-based optimizations** for choosing between alternatives
3. **Statistics propagation** for informed decisions
4. **Extensibility** for custom optimizations

Key takeaways:
- The optimizer runs ~27 different optimization passes in a specific order
- Filter pushdown and join ordering are the most impactful optimizations
- Statistics and cardinality estimation are critical for cost-based decisions
- The optimizer is designed to be extensible and testable
- Understanding the optimizer helps write more efficient queries

For more information:
- Read the source code in `src/optimizer/`
- Check test files in `test/sql/optimizer/`
- Review the CLAUDE.md guide for development best practices
