# Advanced SQL Features in DuckDB

This guide covers advanced SQL features for experienced users. All examples are drawn from DuckDB's test suite and represent production-ready patterns.

## Table of Contents

1. [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
2. [Recursive CTEs](#recursive-ctes)
3. [Window Functions](#window-functions)
4. [PIVOT and UNPIVOT](#pivot-and-unpivot)
5. [Nested Types](#nested-types)
6. [List Functions and Operations](#list-functions-and-operations)
7. [Struct Operations](#struct-operations)
8. [Map Operations](#map-operations)
9. [ASOF Joins](#asof-joins)
10. [Sampling and Approximate Queries](#sampling-and-approximate-queries)
11. [Advanced Subqueries](#advanced-subqueries)

---

## Common Table Expressions (CTEs)

CTEs provide a way to name temporary result sets that exist only during query execution. They improve readability and allow for better query organization.

### Basic CTE Syntax

```sql
WITH cte_name AS (
    SELECT column1, column2
    FROM table_name
    WHERE condition
)
SELECT * FROM cte_name;
```

### Multiple CTEs

```sql
WITH
    a(x) AS (
        SELECT * FROM generate_series(1, 10)
    ),
    b(x) AS (
        SELECT * FROM a WHERE x < 8
    )
SELECT * FROM b WHERE x % 3 = 1 ORDER BY x;
-- Returns: 1, 4, 7
```

### Materialized CTEs

Use `MATERIALIZED` to force DuckDB to compute the CTE once and reuse the result:

```sql
WITH a(x) AS MATERIALIZED (
    SELECT * FROM generate_series(1, 10)
)
SELECT * FROM a WHERE x < 5
UNION ALL
SELECT * FROM a WHERE x > 5;
```

This is useful when:
- The CTE is referenced multiple times
- The CTE computation is expensive
- You want to prevent optimizer transformations

---

## Recursive CTEs

Recursive CTEs allow queries to reference themselves, enabling hierarchical and iterative computations.

### Basic Recursive Pattern

```sql
WITH RECURSIVE t AS (
    -- Base case (anchor member)
    SELECT 1 AS x
    UNION ALL
    -- Recursive case
    SELECT x + 1 FROM t WHERE x < 5
)
SELECT * FROM t;
-- Returns: 1, 2, 3, 4, 5
```

### Recursive CTE with Cross Products

```sql
WITH RECURSIVE t AS MATERIALIZED (
    SELECT 1 AS x
    UNION
    SELECT t1.x + t2.x + t3.x AS x
    FROM t t1, t t2, t t3
    WHERE t1.x < 100
)
SELECT * FROM t ORDER BY 1;
-- Returns: 1, 3, 9, 27, 81, 243
```

### Recursive CTE with Aggregates

```sql
CREATE TABLE a AS SELECT * FROM range(100) t1(i);

WITH RECURSIVE t AS MATERIALIZED (
    SELECT 1 AS x
    UNION
    SELECT SUM(x) AS x
    FROM t, a
    WHERE x < 1000000
)
SELECT * FROM t ORDER BY 1 NULLS LAST;
-- Returns: 1, 100, 10000, 1000000, NULL
```

### Recursive CTE with Correlated Subqueries

```sql
WITH RECURSIVE t AS MATERIALIZED (
    SELECT 1 AS x
    UNION
    SELECT (SELECT t.x + t2.x FROM t t2 LIMIT 1) AS x
    FROM t
    WHERE x < 10
)
SELECT * FROM t ORDER BY 1;
-- Returns: 1, 2, 4, 8, 16
```

---

## Window Functions

Window functions perform calculations across rows related to the current row, without collapsing the result set like aggregates.

### Basic Window Function Syntax

```sql
function_name(...) OVER (
    [PARTITION BY partition_expression]
    [ORDER BY sort_expression]
    [frame_clause]
)
```

### Common Window Functions

#### ROW_NUMBER, RANK, DENSE_RANK

```sql
CREATE TABLE empsalary (
    depname VARCHAR,
    empno BIGINT,
    salary INT
);

-- ROW_NUMBER: sequential numbering within partition
SELECT
    depname,
    empno,
    row_number() OVER (PARTITION BY depname ORDER BY salary) AS rn
FROM empsalary;

-- RANK: gaps after ties
SELECT
    depname,
    salary,
    rank() OVER (PARTITION BY depname ORDER BY salary) AS rnk
FROM empsalary;
-- Example output: 1, 2, 3, 3, 5 (note the gap)

-- DENSE_RANK: no gaps after ties
SELECT
    depname,
    salary,
    dense_rank() OVER (PARTITION BY depname ORDER BY salary) AS drnk
FROM empsalary;
-- Example output: 1, 2, 3, 3, 4 (no gap)
```

#### LEAD and LAG

Access values from subsequent or previous rows:

```sql
-- LAG with offset and default value
SELECT
    id,
    v,
    t,
    lag(v, 2, NULL) OVER (PARTITION BY id ORDER BY t ASC) AS prev_2
FROM win
ORDER BY id, t;

-- LEAD with offset
SELECT
    date,
    "group",
    status,
    LEAD(date, 2) OVER (
        PARTITION BY "group"
        ORDER BY date ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS end_date
FROM issue14398
ORDER BY 2, 1;
```

#### NTH_VALUE

Access the Nth value in a window frame:

```sql
-- Get the 2nd value in each partition
SELECT
    depname,
    empno,
    nth_value(empno, 2) OVER (
        PARTITION BY depname
        ORDER BY empno ASC
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
    ) AS second_val
FROM empsalary
ORDER BY 1, 2;
```

#### NTILE

Divide rows into N roughly equal groups:

```sql
SELECT
    TeamName,
    Player,
    Score,
    NTILE(4) OVER (PARTITION BY TeamName ORDER BY Score ASC) AS quartile
FROM Scoreboard
ORDER BY TeamName, Score;
```

#### FIRST_VALUE and LAST_VALUE

```sql
-- First value in partition
SELECT
    empno,
    first_value(empno) OVER (
        PARTITION BY depname
        ORDER BY empno
    ) AS first_emp
FROM empsalary;

-- Last value (requires proper frame)
SELECT
    depname,
    empno,
    last_value(empno) OVER (
        PARTITION BY depname
        ORDER BY empno ASC
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
    ) AS last_emp
FROM empsalary
ORDER BY 1, 2;
```

### Window Frames

Window frames define which rows are included in the calculation.

#### ROWS-based Frames

```sql
-- Running sum using ROWS
SELECT
    depname,
    empno,
    salary,
    sum(salary) OVER (
        PARTITION BY depname
        ORDER BY empno
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM empsalary;
```

#### RANGE-based Frames

RANGE frames include all rows with values within a specified distance:

```sql
-- Sum values within +/- 5 of current row
SELECT
    a,
    sum(b) OVER (
        ORDER BY a
        RANGE BETWEEN 5 PRECEDING AND 5 FOLLOWING
    ) AS windowed_sum
FROM t1;

-- Works with dates and intervals
SELECT
    date_col,
    sum(value) OVER (
        ORDER BY date_col
        RANGE BETWEEN INTERVAL 5 DAYS PRECEDING
                  AND INTERVAL 5 DAYS FOLLOWING
    ) AS week_window
FROM time_series;
```

### Aggregate Window Functions

Most aggregate functions work as window functions:

```sql
SELECT
    depname,
    min(salary) OVER (PARTITION BY depname ORDER BY salary, empno) AS min_sal,
    max(salary) OVER (PARTITION BY depname ORDER BY salary, empno) AS max_sal,
    avg(salary) OVER (PARTITION BY depname ORDER BY salary, empno) AS avg_sal,
    stddev_pop(salary) OVER (PARTITION BY depname ORDER BY salary, empno) AS std_sal
FROM empsalary
ORDER BY depname, empno;
```

### Statistical Window Functions

```sql
-- Covariance
SELECT
    depname,
    covar_pop(salary, empno) OVER (
        PARTITION BY depname
        ORDER BY salary, empno
    ) AS covar
FROM empsalary;
```

---

## PIVOT and UNPIVOT

PIVOT transforms rows into columns, while UNPIVOT does the reverse.

### PIVOT Syntax

```sql
SELECT * FROM table_name
PIVOT (
    aggregate_function(value_column)
    FOR pivot_column
    IN (value1, value2, ...)
)
```

### Basic PIVOT Example

```sql
CREATE TABLE Produce (
    product VARCHAR,
    sales INT,
    quarter VARCHAR,
    year INT
);

-- Pivot quarters into columns
SELECT * FROM Produce
PIVOT (
    SUM(sales)
    FOR quarter
    IN ('Q1', 'Q2', 'Q3', 'Q4')
)
ORDER BY product, year;
```

### PIVOT with Multiple Aggregates

```sql
SELECT * FROM
    (SELECT product, sales, quarter FROM Produce)
PIVOT (
    SUM(sales) AS total_sales,
    COUNT(*) AS num_records
    FOR quarter
    IN ('Q1', 'Q2')
)
ORDER BY product;
```

### PIVOT with Expressions

```sql
-- Concatenate values in pivot column
PIVOT Cities
ON Country || '_' || Name
USING SUM(Population)
GROUP BY Year;

-- Use CASE expressions
PIVOT Cities
ON (CASE WHEN Country='NL' THEN NULL ELSE Country END)
USING SUM(Population)
GROUP BY Year;
```

### UNPIVOT Syntax

```sql
CREATE TABLE Produce_wide (
    product VARCHAR,
    Q1 INT,
    Q2 INT,
    Q3 INT,
    Q4 INT
);

-- Unpivot quarters back into rows
SELECT * FROM Produce_wide
UNPIVOT (
    sales
    FOR quarter
    IN (Q1, Q2, Q3, Q4)
)
ORDER BY product, quarter;
```

### UNPIVOT with Multiple Value Columns

```sql
SELECT
    product,
    first_half_sales,
    second_half_sales,
    semesters
FROM Produce
UNPIVOT (
    (first_half_sales, second_half_sales)
    FOR semesters
    IN ((Q1, Q2) AS 'semester_1', (Q3, Q4) AS 'semester_2')
);
```

---

## Nested Types

DuckDB supports three nested types: ARRAY (fixed-length), LIST (variable-length), and STRUCT (named fields).

### Creating Nested Types

```sql
-- Lists
SELECT [1, 2, 3] AS my_list;
SELECT list_value(1, 2, 3) AS my_list;

-- Structs
SELECT {'a': 1, 'b': 2} AS my_struct;
SELECT {'name': 'Alice', 'age': 30, 'city': 'NYC'} AS person;

-- Arrays (fixed length)
SELECT [1, 2, 3]::INT[3] AS my_array;

-- Nested combinations
SELECT [
    {'a': 3, 'b': NULL},
    NULL,
    {'a': NULL, 'b': 'hello'}
] AS list_of_structs;
```

### Accessing Nested Data

```sql
-- List indexing (1-based)
SELECT [10, 20, 30][1];  -- Returns: 10
SELECT [10, 20, 30][-1]; -- Returns: 30 (last element)

-- Struct field access
SELECT {'a': 1, 'b': 2}['a'];   -- Returns: 1
SELECT {'a': 1, 'b': 2}.a;      -- Returns: 1

-- Nested access
SELECT [
    {'a': 3, 'b': {'x': 3, 'y': [1, 2, 3]}},
    NULL,
    {'a': NULL, 'b': {'x': NULL, 'y': [4, 5]}}
][1]['b']['y'][2];  -- Returns: 2
```

---

## List Functions and Operations

DuckDB provides extensive functions for working with lists.

### List Construction

```sql
-- list_value: create from values
SELECT list_value(1, 2, 3);

-- list_concat / array_cat: concatenate lists
SELECT list_concat([1, 2], [3, 4]);           -- [1, 2, 3, 4]
SELECT list_concat([1, 2], [3, 4], [5, 6]);  -- [1, 2, 3, 4, 5, 6]

-- list_concat handles NULLs
SELECT list_concat(NULL, [3, 4]);  -- [3, 4]
SELECT list_concat([1, 2], NULL);  -- [1, 2]

-- range: generate numeric sequences
SELECT range(5);        -- [0, 1, 2, 3, 4]
SELECT range(2, 8);     -- [2, 3, 4, 5, 6, 7]
SELECT range(0, 10, 2); -- [0, 2, 4, 6, 8]
```

### List Manipulation

```sql
-- flatten: reduce nested lists by one level
SELECT flatten([[1, 2, 3], [4, 5]]);        -- [1, 2, 3, 4, 5]
SELECT flatten([[1, 2], [], [3, 4]]);       -- [1, 2, 3, 4]

-- list_distinct: remove duplicates
SELECT list_distinct([1, 2, 2, 3, 3, 3]);   -- [1, 2, 3]

-- list_reverse: reverse order
SELECT list_reverse([1, 2, 3, 4]);          -- [4, 3, 2, 1]

-- list_slice: extract sublist
SELECT [1, 2, 3, 4, 5][2:4];                -- [2, 3, 4]
SELECT [1, 2, 3, 4, 5][:3];                 -- [1, 2, 3]
SELECT [1, 2, 3, 4, 5][3:];                 -- [3, 4, 5]
SELECT [1, 2, 3, 4, 5][-2:];                -- [4, 5]
```

### List Search and Testing

```sql
-- list_contains: check for element
SELECT list_contains([1, 2, 3], 2);         -- true
SELECT list_contains([1, 2, 3], 5);         -- false

-- list_position: find first occurrence (1-based)
SELECT list_position([1, 2, 3, 2], 2);      -- 2

-- list_has_any: check if any elements match
SELECT list_has_any([1, 2, 3], [2, 4]);     -- true

-- list_has_all: check if all elements present
SELECT list_has_all([1, 2, 3, 4], [2, 3]);  -- true
```

### List Aggregation

```sql
-- list_aggr: aggregate list elements
SELECT list_aggr([1, 2, 3, 4], 'sum');      -- 10
SELECT list_aggr([1, 2, 3, 4], 'avg');      -- 2.5

-- array_agg: collect values into list
SELECT array_agg(x) FROM range(5) t(x);     -- [0, 1, 2, 3, 4]
SELECT array_agg(x ORDER BY x DESC) FROM range(5) t(x);  -- [4, 3, 2, 1, 0]
```

### List Transformation

```sql
-- list_sort: sort list elements
SELECT list_sort([3, 1, 4, 1, 5]);          -- [1, 1, 3, 4, 5]

-- list_zip: combine multiple lists
SELECT list_zip([1, 2], ['a', 'b']);
-- [{'0': 1, '1': a}, {'0': 2, '1': b}]

-- list_intersect: find common elements
SELECT list_intersect([1, 2, 3], [2, 3, 4]);  -- [2, 3]
```

### UNNEST

UNNEST transforms list elements into rows:

```sql
-- Basic unnest
SELECT UNNEST([1, 2, 3]);
-- Returns three rows: 1, 2, 3

-- Multiple unnests (creates cross product, then zips)
SELECT id, UNNEST(i), UNNEST(j)
FROM (VALUES
    (1, [1, 2], [10]),
    (2, NULL, [11, 12]),
    (3, [3, NULL, 4], [NULL])
) tbl(id, i, j);
-- Returns:
-- 1  1     10
-- 1  2     NULL
-- 2  NULL  11
-- 2  NULL  12
-- 3  3     NULL
-- 3  NULL  NULL
-- 3  4     NULL

-- Unnest list of structs
SELECT UNNEST([
    {'a': 10, 'b': 1},
    {'a': 11, 'b': 2}
]);
-- Returns two rows with struct values
```

---

## Struct Operations

Structs are ordered collections of named fields with potentially different types.

### Creating Structs

```sql
-- Literal syntax
SELECT {'name': 'Alice', 'age': 30} AS person;

-- struct_pack function
SELECT struct_pack(name := 'Alice', age := 30);

-- row constructor
SELECT ROW(1, 2, 3);  -- Creates unnamed struct
```

### Accessing Struct Fields

```sql
-- Dot notation
SELECT person.name FROM people;

-- Bracket notation
SELECT person['name'] FROM people;

-- Extract all fields
SELECT (person).* FROM people;
```

### Struct Functions

```sql
-- struct_extract: get field value
SELECT struct_extract({'a': 1, 'b': 2}, 'a');  -- 1

-- struct_insert: add/update field
SELECT struct_insert({'a': 1}, b := 2);  -- {'a': 1, 'b': 2}
```

### Nested Struct Access

```sql
CREATE TABLE nested AS SELECT [
    {'a': 3, 'b': {'x': 3, 'y': [1, 2, 3]}},
    NULL,
    {'a': NULL, 'b': {'x': NULL, 'y': [4, 5]}}
] AS l;

-- Chain access operators
SELECT l[1]['b']['y'][2] FROM nested;  -- Returns: 2
```

### Structs in Tables

```sql
CREATE TABLE a (
    id INTEGER,
    b ROW(i INTEGER, j INTEGER)
);

INSERT INTO a VALUES (1, {i: 1, j: 2});

-- Query struct fields
SELECT id, b.i, b.j FROM a;
SELECT id, (b).* FROM a;
```

---

## Map Operations

Maps store key-value pairs with unique keys.

### Creating Maps

```sql
-- MAP function
SELECT MAP(
    list_value(1, 2, 3),
    list_value(10, 9, 8)
);  -- {1=10, 2=9, 3=8}

-- Empty map
SELECT MAP();  -- {}
SELECT MAP(list_value(), list_value());  -- {}

-- From columns
CREATE TABLE tbl (a INTEGER[], b TEXT[]);
INSERT INTO tbl VALUES
    (ARRAY[5, 7], ARRAY['test', 'string']),
    (ARRAY[6, 3], ARRAY['foo', 'bar']);

SELECT MAP(a, b) FROM tbl;
-- {5=test, 7=string}
-- {6=foo, 3=bar}

-- map_from_entries: from list of structs
SELECT map_from_entries([
    {'key': 1, 'value': 'a'},
    {'key': 2, 'value': 'b'}
]);  -- {1=a, 2=b}
```

### Accessing Map Values

```sql
-- Subscript operator
SELECT MAP([1, 2], ['a', 'b'])[1];  -- 'a'

-- element_at function
SELECT element_at(MAP([1, 2], ['a', 'b']), 2);  -- 'b'
```

### Map Functions

```sql
-- map_keys: get all keys
SELECT map_keys(MAP(['a', 'b', 'c'], [1, 2, 3]));  -- [a, b, c]

-- map_values: get all values
SELECT map_values(MAP(['a', 'b', 'c'], [1, 2, 3]));  -- [1, 2, 3]

-- map_entries: convert to list of structs
SELECT map_entries(MAP(['a', 'b'], [1, 2]));
-- [{'key': a, 'value': 1}, {'key': b, 'value': 2}]

-- cardinality: count entries
SELECT cardinality(MAP(['a', 'b', 'c'], [1, 2, 3]));  -- 3
```

### Map with Complex Types

```sql
-- Struct keys
SELECT MAP(
    list_value({'i': 1, 'j': 2}, {'i': 3, 'j': 4}),
    list_value({'i': 1, 'j': 2}, {'i': 3, 'j': 4})
);

-- List values
SELECT MAP(
    list_value(1, 2, 3, 4),
    list_value([1], [2], [3], [4])
);  -- {1=[1], 2=[2], 3=[3], 4=[4]}
```

---

## ASOF Joins

ASOF (as-of) joins match rows based on the closest value less than or equal to the join key, useful for time-series data.

### Basic ASOF Join

```sql
CREATE TABLE prices (
    "when" TIMESTAMP,
    symbol INT,
    price INT
);
INSERT INTO prices VALUES ('2020-01-01 00:00:00', 1, 42);

CREATE TABLE trades (
    "when" TIMESTAMP,
    symbol INT
);
INSERT INTO trades VALUES ('2020-01-01 00:00:03', 1);

-- Find the most recent price for each trade
SELECT
    t.*,
    p.price
FROM trades t
ASOF JOIN prices p
    ON t.symbol = p.symbol
    AND t.when >= p.when;
-- Returns: 2020-01-01 00:00:03, 1, 42
```

### ASOF Join Requirements

1. Must have exactly one inequality (>=, >, <=, or <)
2. Can have additional equality conditions
3. The inequality typically uses the time/sequence column

```sql
-- Valid: equality + inequality
SELECT p.ts, e.value
FROM range(0, 10) p(ts)
ASOF JOIN events0 e
    ON 1 = 1 AND p.ts >= e.begin
ORDER BY p.ts ASC;

-- Error: missing inequality
SELECT p.ts, e.value
FROM range(0, 10) p(ts)
ASOF JOIN events0 e
    ON p.ts = e.begin;  -- ERROR: Missing ASOF JOIN inequality

-- Error: multiple inequalities
SELECT p.ts, e.value
FROM range(0, 10) p(ts)
ASOF JOIN events0 e
    ON p.ts >= e.begin
    AND p.ts >= e.value;  -- ERROR: Multiple ASOF JOIN inequalities
```

### ASOF Join with Multiple Conditions

```sql
WITH samples AS (
    SELECT col0 AS starts, col1 AS ends
    FROM (VALUES (5, 9), (10, 13), (14, 20), (21, 23))
)
SELECT
    s1.starts AS s1_starts,
    s2.starts AS s2_starts
FROM samples AS s1
ASOF JOIN samples AS s2
    ON s2.ends >= (s1.ends - 5)
WHERE s1_starts <> s2_starts
ORDER BY ALL;
-- Returns:
-- 10  5
-- 21  14
```

---

## Sampling and Approximate Queries

DuckDB supports statistical sampling and approximate algorithms for performance.

### Table Sampling

```sql
-- SYSTEM sampling: sample entire blocks
SELECT COUNT(*), MIN(x)
FROM test
TABLESAMPLE SYSTEM (25 PERCENT)
REPEATABLE (42);

-- BERNOULLI sampling: sample individual rows
SELECT COUNT(*), MIN(x)
FROM test
TABLESAMPLE BERNOULLI (25 PERCENT)
REPEATABLE (42);
```

Key differences:
- **SYSTEM**: Faster, samples data blocks, less accurate for small tables
- **BERNOULLI**: Slower, samples individual rows, more accurate
- **REPEATABLE**: Ensures same sample with same seed

### Approximate Quantiles

Use approximate quantiles for large datasets when exact values aren't required:

```sql
-- approx_quantile / reservoir_quantile
SELECT approx_quantile(r, 0.5) AS median
FROM quantile;

-- Compare with exact quantile
SELECT
    approx_quantile(r, 0.5) AS approx_median,
    quantile(r, 0.5) AS exact_median
FROM large_table;
```

### Approximate Count Distinct

```sql
-- approx_count_distinct: HyperLogLog-based estimation
SELECT approx_count_distinct(user_id)
FROM large_table;

-- Much faster than exact COUNT(DISTINCT)
```

---

## Advanced Subqueries

DuckDB supports various advanced subquery patterns for complex data retrieval.

### Scalar Subqueries

Scalar subqueries return a single value:

```sql
-- Correlated scalar subquery
SELECT
    i,
    (SELECT 42 + i1.i) AS j
FROM integers i1
ORDER BY i;

-- In ORDER BY clause
SELECT i
FROM integers i1
ORDER BY (SELECT 100 - i1.i);
```

### Correlated Subqueries

Subqueries that reference outer query columns:

```sql
CREATE TABLE integers (i INTEGER);
INSERT INTO integers VALUES (1), (2), (3), (NULL);

-- Correlated filter
SELECT
    i,
    (SELECT 42 + i1.i FROM integers LIMIT 1) AS j
FROM integers i1
ORDER BY i;

-- EXISTS with correlation
SELECT
    i,
    EXISTS(SELECT i FROM integers WHERE i1.i = i) AS exists_match
FROM integers i1
ORDER BY i;

-- ANY with correlation
SELECT
    i,
    i = ANY(SELECT i FROM integers WHERE i1.i = i) AS any_match
FROM integers i1
ORDER BY i;
```

### Subquery Error Handling

```sql
-- By default, multiple rows cause an error
-- This errors:
SELECT i, (SELECT 42 + i1.i FROM integers) AS j
FROM integers i1;

-- Allow multiple rows (returns first)
SET scalar_subquery_error_on_multiple_rows = false;
SELECT i, (SELECT 42 + i1.i FROM integers) AS j
FROM integers i1;
```

### Subqueries in Different Contexts

```sql
-- In SELECT list
SELECT
    i,
    (SELECT MAX(j) FROM other_table WHERE other_table.id = i) AS max_j
FROM main_table;

-- In WHERE clause
SELECT *
FROM customers
WHERE customer_id IN (
    SELECT customer_id
    FROM orders
    WHERE order_date > '2024-01-01'
);

-- In HAVING clause
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING AVG(salary) > (
    SELECT AVG(salary) * 1.1
    FROM employees
);
```

### Nested Subqueries with Windows

```sql
-- Subquery with window function
SELECT
    id,
    (
        SELECT row_number() OVER (ORDER BY value)
        FROM other_table
        WHERE other_table.id = main.id
        LIMIT 1
    ) AS ranked
FROM main;
```

### Array Subqueries

```sql
-- Collect results into array
SELECT
    department,
    ARRAY(
        SELECT employee_name
        FROM employees e
        WHERE e.dept = d.department
        ORDER BY salary DESC
        LIMIT 3
    ) AS top_earners
FROM departments d;
```

---

## Best Practices

### When to Use CTEs vs Subqueries

- **CTEs**: When query is referenced multiple times, or for readability
- **Subqueries**: For one-off transformations within a query
- **MATERIALIZED CTEs**: When CTE is expensive and used multiple times

### Window Function Performance

- Use appropriate frame specifications (ROWS vs RANGE)
- Partition data to reduce window size
- Avoid unnecessary ordering when frame is not needed

### Nested Types Performance

- Access nested fields directly rather than expanding entire structures
- Use projection pushdown by selecting specific fields
- Consider flattening for joins on nested fields

### ASOF Join Tips

- Ensure time/sequence columns are indexed
- Use appropriate inequality direction (>= for "most recent before")
- Combine with equality conditions for better performance

### Sampling Guidelines

- Use SYSTEM sampling for large tables (>1GB)
- Use BERNOULLI for more accurate statistics on smaller tables
- Always use REPEATABLE for reproducible results
- Consider approximate aggregates for exploratory analysis

---

## Additional Resources

- [DuckDB Documentation](https://duckdb.org/docs/)
- [SQL Test Files](https://github.com/duckdb/duckdb/tree/main/test/sql) - Comprehensive examples
- [DuckDB Extensions](https://duckdb.org/docs/extensions/overview) - Additional functionality

## Testing Your Queries

All examples in this guide are based on DuckDB's test suite. You can verify any example by:

```sql
-- Enable verification for correctness checks
PRAGMA enable_verification;

-- Your query here
SELECT ...;
```

For performance testing with sampling:

```sql
-- Ensure reproducibility
require vector_size 2048;

-- Your sampling query with REPEATABLE
SELECT * FROM large_table TABLESAMPLE SYSTEM(10 PERCENT) REPEATABLE(42);
```
