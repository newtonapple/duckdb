# DuckDB SQL Features Guide

This guide covers DuckDB's SQL dialect and unique features from a user perspective, with practical examples from the codebase.

## Table of Contents
1. [DuckDB SQL Dialect Overview](#duckdb-sql-dialect-overview)
2. [Unique and Friendly SQL Features](#unique-and-friendly-sql-features)
3. [Data Types](#data-types)
4. [String Functions and Operators](#string-functions-and-operators)
5. [Date/Time Functions](#datetime-functions)
6. [Aggregation Functions](#aggregation-functions)
7. [Window Functions](#window-functions)
8. [JSON Support](#json-support)
9. [Regular Expressions](#regular-expressions)
10. [Type Casting and Conversion](#type-casting-and-conversion)

---

## DuckDB SQL Dialect Overview

DuckDB uses a SQL dialect based on PostgreSQL's syntax but includes many unique and user-friendly features. Key characteristics:

- **OLAP-focused**: Optimized for analytical queries
- **PostgreSQL-compatible**: Supports most PostgreSQL syntax
- **Extended syntax**: Includes modern SQL features and shortcuts
- **Rich type system**: Support for nested types (arrays, structs, maps)
- **User-friendly**: Convenience features like `SELECT * EXCLUDE`, `GROUP BY ALL`, etc.

---

## Unique and Friendly SQL Features

### SELECT * EXCLUDE

Exclude specific columns from a `SELECT *` query:

```sql
-- Create table
CREATE TABLE integers(i INTEGER, j INTEGER, k INTEGER);
INSERT INTO integers VALUES (1, 2, 3);

-- Exclude single column
SELECT * EXCLUDE i FROM integers;
-- Result: j=2, k=3

-- Exclude multiple columns
SELECT * EXCLUDE (i, j) FROM integers;
-- Result: k=3

-- Works with qualified table names
SELECT integers.* EXCLUDE (i) FROM integers;
-- Result: j=2, k=3
```

### SELECT * REPLACE

Replace column values or expressions while keeping other columns:

```sql
-- Replace a column with a modified version
SELECT * REPLACE (i + 10 AS i) FROM integers;

-- Replace multiple columns
SELECT * REPLACE (i * 2 AS i, j * 3 AS j) FROM integers;
```

### SELECT * RENAME

Rename columns in your SELECT statement:

```sql
-- Rename single column
SELECT * RENAME i AS renamed_col FROM integers;

-- Rename multiple columns
SELECT * RENAME (i AS r1, j AS r2) FROM integers;

-- Can combine with EXCLUDE
SELECT * EXCLUDE (i) RENAME (j AS i) FROM integers;
```

### SELECT * LIKE / ILIKE / SIMILAR TO

Filter columns based on pattern matching:

```sql
CREATE TABLE integers(col1 INTEGER, col2 INTEGER, k INTEGER);
INSERT INTO integers VALUES (1, 2, 3);

-- Select columns matching pattern
SELECT * LIKE 'col%' FROM integers;
-- Result: col1=1, col2=2

-- Case-insensitive matching
SELECT * ILIKE 'COL%' FROM integers;
-- Result: col1=1, col2=2

-- Regex matching
SELECT * SIMILAR TO '.*col.*' FROM integers;
-- Result: col1=1, col2=2

-- NOT LIKE
SELECT * NOT LIKE 'col%' FROM integers;
-- Result: k=3
```

### COLUMNS() Function

Dynamically select columns using expressions:

```sql
-- Select columns matching a pattern
SELECT COLUMNS(lambda x: x LIKE 'col%') FROM integers;

-- Equivalent to SELECT * LIKE
SELECT COLUMNS('col%') FROM integers;
```

### GROUP BY ALL

Automatically group by all non-aggregate columns:

```sql
CREATE TABLE sales(region VARCHAR, product VARCHAR, amount INTEGER);
INSERT INTO sales VALUES ('North', 'A', 100), ('North', 'A', 150), ('South', 'B', 200);

-- Traditional grouping
SELECT region, product, SUM(amount) FROM sales GROUP BY region, product;

-- With GROUP BY ALL (automatically groups by region and product)
SELECT region, product, SUM(amount) FROM sales GROUP BY ALL;

-- Also works with star syntax
SELECT region, product, SUM(amount) FROM sales GROUP BY *;
```

### ORDER BY ALL

Order by all columns in the result:

```sql
SELECT region, product, amount FROM sales ORDER BY ALL;

-- Can also use with star
SELECT region, product, amount FROM sales ORDER BY *;
```

### PIVOT and UNPIVOT

Transform data between wide and long formats:

```sql
-- PIVOT example
CREATE TABLE monthly_sales(empid INT, amount INT, month TEXT);
INSERT INTO monthly_sales VALUES
    (1, 10000, 'JAN'),
    (1, 400, 'JAN'),
    (2, 4500, 'JAN'),
    (1, 5000, 'FEB');

SELECT *
FROM monthly_sales
PIVOT(SUM(amount) FOR month IN ('JAN', 'FEB', 'MAR'))
ORDER BY empid;
-- Result: empid | JAN   | FEB  | MAR
--         1     | 10400 | 5000 | NULL
--         2     | 4500  | NULL | NULL

-- UNPIVOT converts wide format back to long format
```

### FROM-First Syntax

DuckDB supports placing FROM before SELECT:

```sql
FROM integers SELECT i, j WHERE i > 0;
-- Equivalent to: SELECT i, j FROM integers WHERE i > 0;
```

---

## Data Types

### Numeric Types

```sql
-- Integer types
TINYINT   -- 1 byte (-128 to 127)
SMALLINT  -- 2 bytes (-32768 to 32767)
INTEGER   -- 4 bytes (-2147483648 to 2147483647)
BIGINT    -- 8 bytes
HUGEINT   -- 16 bytes (128-bit integer)
UHUGEINT  -- 16 bytes (unsigned 128-bit)

-- Unsigned variants
UTINYINT, USMALLINT, UINTEGER, UBIGINT

-- Floating point
FLOAT, REAL     -- 4 bytes
DOUBLE          -- 8 bytes

-- Fixed-point decimal
DECIMAL(precision, scale)  -- e.g., DECIMAL(10,2)
```

### String Types

```sql
VARCHAR      -- Variable-length string
TEXT         -- Alias for VARCHAR

-- String literals support Unicode
SELECT '🦆' AS duck_emoji;
SELECT 'Hello' || ' ' || 'World' AS greeting;
```

### Date and Time Types

```sql
DATE         -- Calendar date (year, month, day)
TIME         -- Time of day (no timezone)
TIMESTAMP    -- Date and time
TIMESTAMPTZ  -- Timestamp with timezone
INTERVAL     -- Time interval

-- Examples
SELECT DATE '1993-08-14';
SELECT TIME '12:01:00';
SELECT TIMESTAMP '1992-01-01 12:01:00';
SELECT INTERVAL '1 year 2 months';
```

### Boolean Type

```sql
BOOLEAN  -- TRUE, FALSE, or NULL

SELECT TRUE, FALSE, NULL::BOOLEAN;
```

### Binary Types

```sql
BLOB  -- Binary large object
UUID  -- Universally unique identifier

SELECT BLOB '\x00\x01\x02';
SELECT UUID '550e8400-e29b-41d4-a716-446655440000';
```

### Nested Types

#### Arrays/Lists

```sql
-- Create lists
SELECT [1, 2, 3] AS simple_list;
SELECT LIST_VALUE(1, 2, 3) AS another_list;

-- Lists can contain any type
SELECT ['a', 'b', 'c'] AS string_list;
SELECT [[1, 2], [3, 4]] AS nested_list;

-- Access elements (1-indexed)
SELECT [1, 2, 3][1] AS first_element;  -- Result: 1

-- List functions
SELECT list_concat([1, 2], [3, 4]);     -- [1, 2, 3, 4]
SELECT list_contains([1, 2, 3], 2);     -- TRUE
SELECT len([1, 2, 3, 4]);               -- 4
```

#### Structs

Structs are named tuples with typed fields:

```sql
-- Create structs
SELECT {'name': 'Alice', 'age': 30} AS person;
SELECT struct_pack(name := 'Alice', age := 30) AS person;

-- Access struct fields
SELECT person.name FROM (SELECT {'name': 'Alice', 'age': 30} AS person);

-- Arrow operator
SELECT person->name FROM (SELECT {'name': 'Alice'} AS person);

-- Nested structs
SELECT {'i': 1, 'j': [2, 3]} AS nested_struct;
SELECT [{'i': 1, 'j': [2, 3]}, NULL] AS list_of_structs;
```

#### Maps

Maps store key-value pairs:

```sql
-- Create maps
SELECT MAP([1, 2], [3, 4]) AS my_map;
SELECT MAP(LIST_VALUE(1, 2), LIST_VALUE(3, 4));
-- Result: {1=3, 2=4}

-- Access map values
SELECT my_map[1] FROM (SELECT MAP([1, 2], [3, 4]) AS my_map);
-- Result: 3

-- Map with struct keys/values
SELECT MAP(
    LIST_VALUE({'i': 1}, {'i': 2}),
    LIST_VALUE({'j': 10}, {'j': 20})
) AS complex_map;
```

### Enum Types

Define a type with a fixed set of values:

```sql
CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral');

CREATE TABLE person(name VARCHAR, current_mood mood);
INSERT INTO person VALUES ('Alice', 'happy');
```

---

## String Functions and Operators

### Basic String Operations

```sql
-- Concatenation
SELECT 'Hello' || ' ' || 'World';           -- Hello World
SELECT CONCAT('Hello', ' ', 'World');       -- Hello World
SELECT CONCAT_WS(', ', 'a', 'b', 'c');     -- a, b, c (with separator)

-- Length
SELECT LENGTH('Hello');                     -- 5
SELECT LENGTH('🦆');                        -- 1 (character count)

-- Case conversion
SELECT UPPER('hello');                      -- HELLO
SELECT LOWER('HELLO');                      -- hello

-- Substring
SELECT SUBSTRING('Hello World', 1, 5);      -- Hello
SELECT SUBSTRING('🦆🍞🦆', 2, 1);           -- 🍞 (Unicode-aware)

-- Trimming
SELECT TRIM('  hello  ');                   -- 'hello'
SELECT LTRIM('  hello');                    -- 'hello'
SELECT RTRIM('hello  ');                    -- 'hello'
```

### Pattern Matching

```sql
-- LIKE operator
SELECT 'hello' LIKE 'h%';                   -- TRUE
SELECT 'hello' LIKE '%llo';                 -- TRUE

-- ILIKE (case-insensitive)
SELECT 'Hello' ILIKE 'hello';               -- TRUE

-- NOT LIKE
SELECT 'hello' NOT LIKE 'w%';               -- TRUE

-- Escape character
SELECT 'test_value' LIKE 'test\_%' ESCAPE '\';  -- TRUE
```

### String Search

```sql
-- Position/Index
SELECT INSTR('hello world', 'world');       -- 7
SELECT POSITION('world' IN 'hello world');  -- 7

-- Contains
SELECT CONTAINS('hello world', 'world');    -- TRUE

-- Starts with / Ends with
SELECT PREFIX('hello', 'he');               -- TRUE
SELECT SUFFIX('hello', 'lo');               -- TRUE
```

### String Formatting

```sql
-- Format function
SELECT FORMAT('{}', 'hello');                           -- hello
SELECT FORMAT('{}: {}', 'Name', 'Alice');              -- Name: Alice
SELECT FORMAT('{} + {} = {}', 3, 5, 3 + 5);           -- 3 + 5 = 8

-- Format with dates and types
SELECT FORMAT('{}', DATE '1992-01-01');                -- 1992-01-01
SELECT FORMAT('{}', TRUE);                             -- true
```

### String Transformation

```sql
-- Reverse
SELECT REVERSE('hello');                    -- olleh

-- Repeat
SELECT REPEAT('ab', 3);                     -- ababab

-- Replace
SELECT REPLACE('hello world', 'world', 'there');  -- hello there

-- Padding
SELECT LPAD('42', 5, '0');                  -- 00042
SELECT RPAD('42', 5, '0');                  -- 42000
```

### Advanced String Functions

```sql
-- MD5, SHA256 hashing
SELECT MD5('hello');
SELECT SHA256('hello');

-- Base64 encoding
SELECT BASE64('hello');

-- ASCII values
SELECT ASCII('A');                          -- 65
SELECT CHR(65);                            -- A

-- Strip accents
SELECT STRIP_ACCENTS('café');              -- cafe
```

---

## Date/Time Functions

### Current Date/Time

```sql
SELECT CURRENT_DATE;                        -- Today's date
SELECT CURRENT_TIME;                        -- Current time
SELECT CURRENT_TIMESTAMP;                   -- Current date and time
SELECT NOW();                              -- Alias for CURRENT_TIMESTAMP
```

### Date/Time Extraction

```sql
-- EXTRACT function
SELECT EXTRACT(YEAR FROM DATE '1992-08-14');       -- 1992
SELECT EXTRACT(MONTH FROM DATE '1992-08-14');      -- 8
SELECT EXTRACT(DAY FROM DATE '1992-08-14');        -- 14

-- DATE_PART function (alternative syntax)
SELECT DATE_PART('year', DATE '1992-08-14');       -- 1992
SELECT DATE_PART('month', DATE '1992-08-14');      -- 8
SELECT DATE_PART('day', DATE '1992-08-14');        -- 14

-- Available parts: year, quarter, month, week, day, hour, minute, second
-- Also: millennium, century, decade, isoyear, weekday, isodow, dayofyear
SELECT DATE_PART('quarter', DATE '1992-08-14');    -- 3
SELECT DATE_PART('weekday', DATE '1992-08-14');    -- 5 (Friday)
```

### Date/Time Arithmetic

```sql
-- Add/subtract days
SELECT DATE '1993-08-14' + 5;              -- 1993-08-19
SELECT DATE '1993-08-14' - 5;              -- 1993-08-09

-- Difference between dates (returns number of days)
SELECT DATE '1993-08-19' - DATE '1993-08-14';  -- 5

-- Interval arithmetic
SELECT DATE '1992-01-01' + INTERVAL '1 year';      -- 1993-01-01
SELECT DATE '1992-01-01' + INTERVAL '2 months';    -- 1992-03-01
SELECT TIMESTAMP '1992-01-01 10:00:00' + INTERVAL '30 minutes';
```

### Date/Time Truncation

```sql
-- Truncate to specific granularity
SELECT DATE_TRUNC('year', DATE '1992-08-14');      -- 1992-01-01
SELECT DATE_TRUNC('month', DATE '1992-08-14');     -- 1992-08-01
SELECT DATE_TRUNC('day', TIMESTAMP '1992-08-14 13:45:00');  -- 1992-08-14 00:00:00
```

### Date Construction

```sql
-- Make date
SELECT MAKE_DATE(1992, 8, 14);             -- 1992-08-14

-- Make timestamp
SELECT MAKE_TIMESTAMP(1992, 8, 14, 12, 30, 0);
```

### Date/Time Formatting

```sql
-- Convert to string with format
SELECT STRFTIME(DATE '1992-08-14', '%Y-%m-%d');    -- 1992-08-14
SELECT STRFTIME(DATE '1992-08-14', '%B %d, %Y');   -- August 14, 1992

-- Parse string to date
SELECT STRPTIME('1992-08-14', '%Y-%m-%d');
```

---

## Aggregation Functions

### Basic Aggregates

```sql
-- COUNT
SELECT COUNT(*) FROM table;                -- Count all rows
SELECT COUNT(column) FROM table;           -- Count non-NULL values
SELECT COUNT(DISTINCT column) FROM table;  -- Count distinct values

-- SUM
SELECT SUM(amount) FROM sales;

-- AVG
SELECT AVG(amount) FROM sales;

-- MIN/MAX
SELECT MIN(amount), MAX(amount) FROM sales;
```

### Statistical Aggregates

```sql
-- Standard deviation
SELECT STDDEV(amount) FROM sales;          -- Sample standard deviation
SELECT STDDEV_POP(amount) FROM sales;      -- Population standard deviation

-- Variance
SELECT VARIANCE(amount) FROM sales;        -- Sample variance
SELECT VAR_POP(amount) FROM sales;         -- Population variance

-- Median
SELECT MEDIAN(amount) FROM sales;

-- Mode (most frequent value)
SELECT MODE(amount) FROM sales;

-- Quantiles
SELECT QUANTILE_CONT(amount, 0.5) FROM sales;  -- Continuous quantile (median)
SELECT QUANTILE_DISC(amount, 0.5) FROM sales;  -- Discrete quantile

-- Approximate quantile (faster for large datasets)
SELECT APPROX_QUANTILE(amount, 0.5) FROM sales;
```

### Statistical Measures

```sql
-- Correlation
SELECT CORR(x, y) FROM data;

-- Covariance
SELECT COVAR_POP(x, y) FROM data;          -- Population covariance
SELECT COVAR_SAMP(x, y) FROM data;         -- Sample covariance

-- Skewness
SELECT SKEWNESS(amount) FROM sales;

-- Kurtosis
SELECT KURTOSIS(amount) FROM sales;

-- Entropy
SELECT ENTROPY(category) FROM data;
```

### String Aggregates

```sql
-- String concatenation
SELECT STRING_AGG(name, ', ') FROM employees;
SELECT STRING_AGG(name, ', ' ORDER BY name) FROM employees;

-- List aggregation
SELECT LIST(name) FROM employees;          -- Create list of all values
SELECT LIST(name ORDER BY name) FROM employees;
```

### Advanced Aggregates

```sql
-- ARG_MIN/ARG_MAX: Return the value of one column when another is min/max
SELECT ARG_MIN(name, salary) FROM employees;  -- Name of lowest paid employee
SELECT ARG_MAX(name, salary) FROM employees;  -- Name of highest paid employee

-- FIRST/LAST
SELECT FIRST(amount ORDER BY date) FROM sales;
SELECT LAST(amount ORDER BY date) FROM sales;

-- ANY_VALUE: Return arbitrary value (useful with GROUP BY)
SELECT region, ANY_VALUE(manager) FROM stores GROUP BY region;

-- HISTOGRAM: Count occurrences
SELECT HISTOGRAM(category) FROM products;

-- BOOL_AND/BOOL_OR
SELECT BOOL_AND(is_active) FROM users;     -- TRUE if all are true
SELECT BOOL_OR(is_active) FROM users;      -- TRUE if any is true

-- BIT operations
SELECT BIT_AND(flags) FROM data;
SELECT BIT_OR(flags) FROM data;
SELECT BIT_XOR(flags) FROM data;

-- PRODUCT: Multiply all values
SELECT PRODUCT(quantity) FROM inventory;
```

### Aggregate Filters

```sql
-- Filter aggregates with FILTER clause
SELECT
    COUNT(*) FILTER (WHERE amount > 100) AS high_value_count,
    SUM(amount) FILTER (WHERE region = 'North') AS north_total
FROM sales;
```

---

## Window Functions

### Basic Window Functions

Window functions perform calculations across a set of rows related to the current row:

```sql
CREATE TABLE employees(name VARCHAR, department VARCHAR, salary INTEGER);
INSERT INTO employees VALUES
    ('Alice', 'Sales', 50000),
    ('Bob', 'Sales', 60000),
    ('Carol', 'IT', 70000),
    ('Dave', 'IT', 80000);

-- ROW_NUMBER: Assign sequential numbers
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary) AS row_num
FROM employees;

-- RANK: Rank with gaps for ties
SELECT
    name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank
FROM employees;

-- DENSE_RANK: Rank without gaps
SELECT
    name,
    salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;
```

### Window Partitioning

```sql
-- Partition by department
SELECT
    name,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;

-- Multiple window functions
SELECT
    name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg,
    salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
```

### Ranking Functions

```sql
-- NTILE: Divide into N buckets
SELECT
    name,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;

-- PERCENT_RANK: Relative rank (0 to 1)
SELECT
    name,
    salary,
    PERCENT_RANK() OVER (ORDER BY salary) AS pct_rank
FROM employees;

-- CUME_DIST: Cumulative distribution
SELECT
    name,
    salary,
    CUME_DIST() OVER (ORDER BY salary) AS cumulative_dist
FROM employees;
```

### Value Functions

```sql
-- LAG: Access previous row
SELECT
    name,
    salary,
    LAG(salary, 1) OVER (ORDER BY salary) AS prev_salary
FROM employees;

-- LEAD: Access next row
SELECT
    name,
    salary,
    LEAD(salary, 1) OVER (ORDER BY salary) AS next_salary
FROM employees;

-- FIRST_VALUE/LAST_VALUE
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary) AS min_dept_salary,
    LAST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS max_dept_salary
FROM employees;
```

### Window Frames

Control which rows are included in window calculations:

```sql
-- ROWS frame
SELECT
    name,
    salary,
    AVG(salary) OVER (
        ORDER BY salary
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS moving_avg
FROM employees;

-- RANGE frame
SELECT
    name,
    salary,
    SUM(salary) OVER (
        ORDER BY salary
        RANGE BETWEEN 10000 PRECEDING AND 10000 FOLLOWING
    ) AS salary_range_sum
FROM employees;
```

---

## JSON Support

DuckDB has native JSON support (requires `json` extension):

### Reading JSON

```sql
-- Read JSON file
SELECT * FROM 'data.json';

-- Read with specific format
SELECT * FROM read_json('data.json');

-- Read JSON array
COPY table FROM 'data.json' (ARRAY);
```

### Writing JSON

```sql
-- Export to JSON
COPY table TO 'output.json' (ARRAY);

-- Export database as JSON
EXPORT DATABASE 'export_dir' (FORMAT JSON);
```

### JSON Functions

```sql
-- Parse JSON string (if json extension loaded)
SELECT '{"name": "Alice", "age": 30}'::JSON;

-- Extract JSON field
SELECT json_extract('{"name": "Alice"}', '$.name');

-- Check JSON type
SELECT json_type('{"name": "Alice"}');
```

---

## Regular Expressions

### Basic Regex Matching

```sql
-- REGEXP_MATCHES: Check if pattern matches
SELECT REGEXP_MATCHES('foobarbaz', 'bar');     -- TRUE
SELECT REGEXP_MATCHES('foobarbaz', 'BAR');     -- FALSE

-- Case-insensitive with options
SELECT REGEXP_MATCHES('foobarbaz', 'BAR', 'i'); -- TRUE

-- Alternative operators
SELECT 'foobarbaz' ~ 'bar';                     -- TRUE (PostgreSQL style)
SELECT 'foobarbaz' !~ 'xyz';                    -- TRUE (not match)
```

### Regex Extract

```sql
-- Extract first match
SELECT REGEXP_EXTRACT('foobarbaz', 'b..');          -- 'bar'

-- Extract specific group
SELECT REGEXP_EXTRACT('foobarbaz', '(b..)(b..)', 1); -- 'bar'
SELECT REGEXP_EXTRACT('foobarbaz', '(b..)(b..)', 2); -- 'baz'

-- Extract all matches
SELECT REGEXP_EXTRACT_ALL('foobarbaz', 'b..');      -- ['bar', 'baz']
```

### Regex Replace

```sql
-- Replace first match
SELECT REGEXP_REPLACE('hello world', 'world', 'there');  -- 'hello there'

-- Replace all matches with 'g' flag
SELECT REGEXP_REPLACE('aaa', 'a', 'b', 'g');             -- 'bbb'

-- Case-insensitive replace
SELECT REGEXP_REPLACE('Hello World', 'hello', 'hi', 'i'); -- 'hi World'
```

### Regex Split

```sql
-- Split string by regex pattern
SELECT REGEXP_SPLIT_TO_TABLE('a,b,c', ',');
-- Result (3 rows):
-- a
-- b
-- c

SELECT REGEXP_SPLIT_TO_ARRAY('a,b,c', ',');
-- Result: ['a', 'b', 'c']
```

### Common Patterns

```sql
-- Email validation
SELECT REGEXP_MATCHES(email, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');

-- Phone number extraction
SELECT REGEXP_EXTRACT(text, '\d{3}-\d{3}-\d{4}');

-- URL parsing
SELECT REGEXP_EXTRACT(url, 'https?://([^/]+)');
```

---

## Type Casting and Conversion

### Basic Casting

```sql
-- CAST function
SELECT CAST('42' AS INTEGER);
SELECT CAST(42 AS VARCHAR);
SELECT CAST('1992-08-14' AS DATE);

-- :: operator (PostgreSQL style)
SELECT '42'::INTEGER;
SELECT 42::VARCHAR;
SELECT '1992-08-14'::DATE;
```

### Safe Casting (TRY_CAST)

Returns NULL instead of error on conversion failure:

```sql
-- TRY_CAST returns NULL on failure
SELECT TRY_CAST('42' AS INTEGER);          -- 42
SELECT TRY_CAST('invalid' AS INTEGER);     -- NULL
SELECT TRY_CAST('2024-13-40' AS DATE);     -- NULL (invalid date)
```

### Numeric Conversions

```sql
-- Integer to Float
SELECT CAST(42 AS DOUBLE);                 -- 42.0

-- Float to Integer (truncates)
SELECT CAST(3.7 AS INTEGER);               -- 3
SELECT CAST(-3.7 AS INTEGER);              -- -3

-- String to Numeric
SELECT CAST('123.45' AS DECIMAL(10,2));    -- 123.45
SELECT '123.45'::DECIMAL(10,2);            -- 123.45
```

### String Conversions

```sql
-- Any type to string
SELECT CAST(42 AS VARCHAR);                -- '42'
SELECT CAST(DATE '1992-08-14' AS VARCHAR); -- '1992-08-14'
SELECT CAST(TRUE AS VARCHAR);              -- 'true'

-- String to Date/Time
SELECT CAST('1992-08-14' AS DATE);
SELECT '12:30:00'::TIME;
SELECT '1992-08-14 12:30:00'::TIMESTAMP;
```

### Implicit Casting

DuckDB performs automatic type conversion in many contexts:

```sql
-- Numeric types are automatically promoted
SELECT 1 + 2.5;                            -- 3.5 (INTEGER + DOUBLE = DOUBLE)

-- Strings auto-cast to numbers in arithmetic
SELECT '10' + 5;                           -- 15

-- Date arithmetic
SELECT DATE '1992-08-14' + 7;              -- 1992-08-21 (days)
```

### Nested Type Casting

```sql
-- Cast entire list
SELECT CAST([1, 2, 3] AS INTEGER[]);
SELECT ['1', '2', '3']::INTEGER[];         -- [1, 2, 3]

-- Cast struct fields
SELECT CAST({'a': '42'} AS STRUCT(a INTEGER));
```

### UNION Type Casting

DuckDB automatically handles type casting in UNION operations:

```sql
-- Numeric types are promoted to common type
SELECT 1 AS x UNION SELECT 1.5 AS x;       -- Result: DOUBLE

-- Strings and numbers may need explicit casting
SELECT '42' AS x UNION SELECT 43 AS x;     -- Error: type mismatch
SELECT CAST('42' AS INTEGER) AS x UNION SELECT 43 AS x;  -- OK
```

---

## Advanced Features

### Lambda Functions

DuckDB supports lambda expressions for list operations:

```sql
-- Transform list elements
SELECT LIST_TRANSFORM([1, 2, 3], lambda x: x * 2);  -- [2, 4, 6]

-- Filter list elements
SELECT LIST_FILTER([1, 2, 3, 4], lambda x: x > 2);  -- [3, 4]

-- Reduce list to single value
SELECT LIST_REDUCE([1, 2, 3, 4], lambda x, y: x + y);  -- 10

-- Use with COLUMNS
SELECT COLUMNS(lambda x: x LIKE 'col%') FROM table;
```

### WITH (Common Table Expressions)

```sql
-- Basic CTE
WITH cte1 AS (
    SELECT i AS j FROM table_a
)
SELECT * FROM cte1;

-- Multiple CTEs
WITH
    cte1 AS (SELECT i AS j FROM table_a),
    cte2 AS (SELECT j * 2 AS k FROM cte1)
SELECT * FROM cte2;

-- Recursive CTE
WITH RECURSIVE count_to_10 AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM count_to_10 WHERE n < 10
)
SELECT * FROM count_to_10;
```

### CASE Expressions

```sql
-- Simple CASE
SELECT
    CASE status
        WHEN 'active' THEN 'Active User'
        WHEN 'inactive' THEN 'Inactive User'
        ELSE 'Unknown'
    END AS status_label
FROM users;

-- Searched CASE
SELECT
    CASE
        WHEN age < 18 THEN 'Minor'
        WHEN age < 65 THEN 'Adult'
        ELSE 'Senior'
    END AS age_group
FROM people;
```

### COALESCE and NULL Handling

```sql
-- COALESCE: Return first non-NULL value
SELECT COALESCE(NULL, NULL, 'default', 'other');  -- 'default'
SELECT COALESCE(column_a, column_b, 'N/A') FROM table;

-- NULLIF: Return NULL if values are equal
SELECT NULLIF(column, 0) FROM table;  -- Returns NULL when column = 0

-- IS NULL / IS NOT NULL
SELECT * FROM table WHERE column IS NULL;
SELECT * FROM table WHERE column IS NOT NULL;
```

### QUALIFY Clause

Filter results based on window function results:

```sql
-- Get top 3 salaries per department
SELECT name, department, salary
FROM employees
QUALIFY ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) <= 3;
```

---

## Best Practices

1. **Use appropriate data types**: Choose the smallest type that fits your data
2. **Leverage nested types**: Use lists, structs, and maps for complex data
3. **Use friendly syntax**: Take advantage of `GROUP BY ALL`, `SELECT * EXCLUDE`, etc.
4. **Prefer TRY_CAST**: Use `TRY_CAST` for safer type conversions
5. **Use CTEs**: Make complex queries readable with Common Table Expressions
6. **Window functions**: Prefer window functions over self-joins for analytical queries
7. **Index on filter columns**: Create indexes on frequently filtered columns
8. **QUALIFY over subqueries**: Use QUALIFY clause instead of nested queries with window functions

---

## Additional Resources

- DuckDB Documentation: https://duckdb.org/docs/
- Test files: `/test/sql/` directory in the DuckDB repository
- Function reference: https://duckdb.org/docs/sql/functions/overview

---

*This guide is based on DuckDB test files and represents features available in the codebase. Features may vary by version.*
