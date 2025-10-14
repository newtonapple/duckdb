# DuckDB Usage Overview - A Beginner's Guide

## Table of Contents
1. [What is DuckDB?](#what-is-duckdb)
2. [When to Use DuckDB](#when-to-use-duckdb)
3. [Installation](#installation)
4. [Getting Started with the CLI](#getting-started-with-the-cli)
5. [Basic SQL Operations](#basic-sql-operations)
6. [Working with Data Files](#working-with-data-files)
7. [CLI Commands and Shortcuts](#cli-commands-and-shortcuts)
8. [Configuration and Settings](#configuration-and-settings)
9. [Output Formatting](#output-formatting)
10. [Performance Tips](#performance-tips)
11. [DuckDB vs Other Databases](#duckdb-vs-other-databases)

---

## What is DuckDB?

DuckDB is a high-performance analytical database system designed to be fast, reliable, portable, and easy to use. Think of it as "SQLite for analytics" - it's an embeddable database that excels at analytical queries (OLAP) rather than transactional workloads (OLTP).

**Key Features:**
- **Fast analytical queries**: Optimized for aggregations, joins, and complex queries on large datasets
- **Embeddable**: No server to manage - runs in-process with your application
- **Rich SQL dialect**: Supports window functions, CTEs, complex types (arrays, structs, maps), and nested queries
- **Zero dependencies**: Single binary with no external requirements
- **Direct file querying**: Query CSV, Parquet, and JSON files directly without loading them first
- **Extensible**: Support for various extensions (Parquet, JSON, Excel, etc.)

---

## When to Use DuckDB

**Use DuckDB when:**
- Analyzing large CSV, Parquet, or JSON files
- Running analytical queries on data (aggregations, reporting, data science)
- You need a lightweight analytical database without server setup
- Performing data transformations and ETL operations
- Working with pandas DataFrames in Python
- You need portable analytics (single file database)
- Doing exploratory data analysis

**Don't use DuckDB when:**
- You need high-concurrency writes (use PostgreSQL, MySQL)
- Building a traditional web application with many users (use PostgreSQL, MySQL)
- You need real-time transactional processing
- Multiple processes need to write simultaneously

---

## Installation

### CLI Installation

**macOS:**
```bash
# Using Homebrew
brew install duckdb

# Or download directly
wget https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-osx-universal.zip
unzip duckdb_cli-osx-universal.zip
```

**Linux:**
```bash
# Download the CLI
wget https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-amd64.zip
unzip duckdb_cli-linux-amd64.zip
chmod +x duckdb
```

**Windows:**
```bash
# Download from GitHub releases
# https://github.com/duckdb/duckdb/releases
# Extract duckdb.exe and run
```

### Python Installation

```bash
pip install duckdb
```

```python
import duckdb

# Create an in-memory database
con = duckdb.connect()

# Or connect to a file
con = duckdb.connect('my_database.db')

# Run queries
result = con.execute("SELECT 42 AS answer").fetchall()
print(result)
```

### Other Clients
- **R**: `install.packages("duckdb")`
- **Java**: Available via Maven Central
- **Node.js**: `npm install duckdb`
- **WebAssembly**: Available for browser usage

---

## Getting Started with the CLI

### Starting DuckDB

**In-memory database (temporary):**
```bash
duckdb
```

**Persistent database:**
```bash
# Create or open a database file
duckdb my_database.db
```

**Read-only mode:**
```bash
duckdb my_database.db -readonly
```

You'll see a welcome message:
```
v1.x.x xxxxxxxx
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D
```

### Exiting DuckDB

```sql
.quit
-- or
.exit
-- or press Ctrl+D (Linux/Mac) or Ctrl+Z (Windows)
```

---

## Basic SQL Operations

### Creating Tables

```sql
-- Create a simple table
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name VARCHAR,
    age INTEGER,
    email VARCHAR
);

-- Create table with constraints
CREATE TABLE products (
    product_id INTEGER PRIMARY KEY,
    name VARCHAR NOT NULL,
    price DECIMAL(10, 2) CHECK (price > 0),
    category VARCHAR DEFAULT 'general'
);

-- Create table from query
CREATE TABLE summary AS
SELECT category, COUNT(*) as count, AVG(price) as avg_price
FROM products
GROUP BY category;
```

### Inserting Data

```sql
-- Insert single row
INSERT INTO users VALUES (1, 'Alice', 30, 'alice@example.com');

-- Insert with column names
INSERT INTO users (id, name, age, email)
VALUES (2, 'Bob', 25, 'bob@example.com');

-- Insert multiple rows
INSERT INTO users VALUES
    (3, 'Charlie', 35, 'charlie@example.com'),
    (4, 'Diana', 28, 'diana@example.com'),
    (5, 'Eve', 32, 'eve@example.com');
```

### Querying Data

```sql
-- Simple SELECT
SELECT * FROM users;

-- Filter with WHERE
SELECT name, age FROM users WHERE age > 30;

-- Aggregations
SELECT COUNT(*) as total_users, AVG(age) as avg_age
FROM users;

-- GROUP BY
SELECT
    CASE WHEN age < 30 THEN 'Young'
         WHEN age < 40 THEN 'Middle'
         ELSE 'Senior' END as age_group,
    COUNT(*) as count
FROM users
GROUP BY age_group;

-- Ordering
SELECT name, age FROM users ORDER BY age DESC LIMIT 3;
```

### Updating and Deleting

```sql
-- Update records
UPDATE users SET age = 31 WHERE name = 'Alice';

-- Delete records
DELETE FROM users WHERE age < 25;

-- Delete all records
DELETE FROM users;

-- Drop table
DROP TABLE users;
```

### Joins

```sql
-- Create related tables
CREATE TABLE orders (
    order_id INTEGER,
    user_id INTEGER,
    product VARCHAR,
    amount DECIMAL
);

-- Inner join
SELECT u.name, o.product, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id;

-- Left join
SELECT u.name, o.product, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

---

## Working with Data Files

One of DuckDB's most powerful features is direct file querying.

### Querying CSV Files

```sql
-- Query CSV directly
SELECT * FROM 'data.csv';

-- With options
SELECT * FROM read_csv('data.csv',
    header=true,
    delimiter=',',
    auto_detect=true
);

-- Create table from CSV
CREATE TABLE my_table AS SELECT * FROM 'data.csv';

-- Import CSV into existing table
COPY my_table FROM 'data.csv' (HEADER);
```

### Querying Parquet Files

```sql
-- Query Parquet directly
SELECT * FROM 'data.parquet';

-- Multiple files with glob patterns
SELECT * FROM 'data/*.parquet';

-- Create table from Parquet
CREATE TABLE my_table AS SELECT * FROM 'data.parquet';

-- Export to Parquet
COPY (SELECT * FROM my_table) TO 'output.parquet' (FORMAT PARQUET);
```

### Querying JSON Files

```sql
-- Query JSON file
SELECT * FROM 'data.json';

-- Read JSON with specific format
SELECT * FROM read_json('data.json', format='array');

-- Export to JSON
COPY (SELECT * FROM my_table) TO 'output.json';
```

### Querying Multiple Files

```sql
-- All CSV files in a directory
SELECT * FROM 'data/*.csv';

-- Combine multiple Parquet files
SELECT * FROM 'data/year=2023/*.parquet'
UNION ALL
SELECT * FROM 'data/year=2024/*.parquet';

-- Query files from the web
SELECT * FROM 'https://example.com/data.csv';
```

---

## CLI Commands and Shortcuts

DuckDB's CLI provides many useful dot commands (similar to SQLite).

### Help and Information

```sql
-- Show all available commands
.help

-- Show help for specific command
.help .mode

-- Show current settings
.show

-- List all tables
.tables

-- Show table schema
.schema users

-- Show schema for all tables
.schema

-- Show database connections
.databases
```

### Database Operations

```sql
-- Open a different database
.open my_database.db

-- Open database with options
.open --readonly my_database.db
.open --new fresh_database.db

-- Close current and open in-memory
.open
```

### Importing and Exporting

```sql
-- Import CSV into table
.import data.csv my_table

-- Import CSV with options
.import --csv data.csv my_table
.import --csv --skip 1 data.csv my_table

-- Dump database as SQL
.dump

-- Dump specific table
.dump my_table

-- Read SQL from file
.read my_script.sql
```

### Output Control

```sql
-- Redirect output to file
.output results.txt
SELECT * FROM users;
.output  -- Reset to stdout

-- Run query and output once
.once results.txt
SELECT * FROM users;

-- Output as CSV to file
.mode csv
.output results.csv
SELECT * FROM users;
.output
```

### Display Settings

```sql
-- Show column headers
.headers on
.headers off

-- Set display mode
.mode duckbox    -- Default, nice tables
.mode csv        -- Comma-separated
.mode json       -- JSON array
.mode markdown   -- Markdown tables
.mode line       -- One column per line
.mode column     -- Columnar output
.mode insert     -- SQL INSERT statements
.mode list       -- Pipe-separated

-- Set NULL display value
.nullvalue NULL

-- Enable query timer
.timer on
.timer off

-- Echo commands
.echo on
.echo off
```

### Editor Integration

```sql
-- Open external editor for query
.edit

-- Or use the shortcut
\e
```

### Miscellaneous

```sql
-- Print text
.print Hello, DuckDB!

-- Shell command (if not in safe mode)
.shell ls -la

-- Exit
.quit
.exit
```

---

## Configuration and Settings

DuckDB can be configured using SET statements and PRAGMA commands.

### Memory Settings

```sql
-- Set memory limit (in human-readable format)
SET memory_limit='4GB';
SET memory_limit='512MB';

-- View current settings
SELECT * FROM duckdb_settings();

-- View specific setting
SELECT * FROM duckdb_settings() WHERE name = 'memory_limit';
```

### Thread Settings

```sql
-- Set number of threads
SET threads=4;

-- Use all available cores
SET threads TO -1;
```

### Other Common Settings

```sql
-- Set default null order in sorting
SET default_null_order='nulls_first';
SET default_null_order='nulls_last';

-- Set timezone
SET TimeZone='America/New_York';
SET TimeZone='UTC';

-- Enable/disable progress bar
SET enable_progress_bar=true;

-- Temporary directory
SET temp_directory='/tmp/duckdb_temp';

-- Enable/disable optimizer
SET enable_optimizer=true;
```

### PRAGMAs

```sql
-- Enable verification (for development)
PRAGMA enable_verification;

-- Show database size
PRAGMA database_size;

-- Show table info
PRAGMA table_info('users');

-- Show version
SELECT version();
```

---

## Output Formatting

### Table Display Modes

**Duckbox (default)** - Beautiful Unicode tables:
```sql
.mode duckbox
SELECT * FROM users LIMIT 3;
```
```
┌────────┬─────────┬───────┬──────────────────────┐
│   id   │  name   │  age  │        email         │
│ int32  │ varchar │ int32 │       varchar        │
├────────┼─────────┼───────┼──────────────────────┤
│      1 │ Alice   │    30 │ alice@example.com    │
│      2 │ Bob     │    25 │ bob@example.com      │
│      3 │ Charlie │    35 │ charlie@example.com  │
└────────┴─────────┴───────┴──────────────────────┘
```

**CSV** - For data export:
```sql
.mode csv
SELECT * FROM users LIMIT 2;
```
```
id,name,age,email
1,Alice,30,alice@example.com
2,Bob,25,bob@example.com
```

**JSON** - For API integration:
```sql
.mode json
SELECT * FROM users LIMIT 2;
```
```json
[
  {"id":1,"name":"Alice","age":30,"email":"alice@example.com"},
  {"id":2,"name":"Bob","age":25,"email":"bob@example.com"}
]
```

**Markdown** - For documentation:
```sql
.mode markdown
SELECT * FROM users LIMIT 2;
```
```
| id | name  | age | email             |
|----|-------|-----|-------------------|
| 1  | Alice | 30  | alice@example.com |
| 2  | Bob   | 25  | bob@example.com   |
```

### Formatting Options

```sql
-- Limit displayed rows
.maxrows 100

-- Set column widths
.width 10 20 15

-- Set separators for list mode
.separator '|'

-- Configure number rendering
.large_number_rendering all
.decimal_sep ','
.thousand_sep '.'
```

---

## Performance Tips

### 1. Use Appropriate Data Types

```sql
-- Use specific types instead of VARCHAR for better performance
CREATE TABLE events (
    event_id INTEGER,
    event_date DATE,          -- Not VARCHAR
    event_time TIMESTAMP,     -- Not VARCHAR
    price DECIMAL(10,2),      -- Not VARCHAR
    is_active BOOLEAN         -- Not VARCHAR
);
```

### 2. Create Indexes for Lookups

```sql
-- Create index on frequently queried columns
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_order_date ON orders(order_date);
```

### 3. Use Parquet for Large Datasets

Parquet is much faster than CSV for analytical queries:

```sql
-- Export to Parquet for better performance
COPY (SELECT * FROM large_table) TO 'data.parquet' (FORMAT PARQUET);

-- Query Parquet files directly
SELECT COUNT(*) FROM 'large_data.parquet';
```

### 4. Partition Large Datasets

```sql
-- Write partitioned data
COPY (SELECT * FROM orders)
TO 'orders_by_year'
(FORMAT PARQUET, PARTITION_BY (year));

-- Query specific partitions
SELECT * FROM 'orders_by_year/year=2024/*.parquet';
```

### 5. Use LIMIT for Exploration

```sql
-- When exploring large datasets, use LIMIT
SELECT * FROM huge_table LIMIT 100;

-- Use TABLESAMPLE for random sampling
SELECT * FROM huge_table USING SAMPLE 10%;
```

### 6. Optimize Memory Usage

```sql
-- For large datasets, increase memory limit
SET memory_limit='8GB';

-- Use external sorting for very large sorts
SET temp_directory='/path/to/fast/disk';
```

### 7. Leverage Parallelism

```sql
-- Use all CPU cores
SET threads TO -1;

-- Or set specific number
SET threads=8;
```

### 8. Use CTEs and Views

```sql
-- Create views for complex queries
CREATE VIEW user_summary AS
SELECT
    user_id,
    COUNT(*) as order_count,
    SUM(amount) as total_spent
FROM orders
GROUP BY user_id;

-- Query the view
SELECT * FROM user_summary WHERE total_spent > 1000;
```

### 9. Monitor Query Performance

```sql
-- Enable timing
.timer on

-- Analyze query plan
EXPLAIN SELECT * FROM large_table WHERE condition = 'value';

-- Analyze with execution stats
EXPLAIN ANALYZE SELECT * FROM large_table WHERE condition = 'value';
```

### 10. Batch Operations

```sql
-- Insert multiple rows at once
INSERT INTO users VALUES
    (1, 'Alice', 30),
    (2, 'Bob', 25),
    (3, 'Charlie', 35);

-- Better than individual inserts
-- INSERT INTO users VALUES (1, 'Alice', 30);
-- INSERT INTO users VALUES (2, 'Bob', 25);
-- INSERT INTO users VALUES (3, 'Charlie', 35);
```

---

## DuckDB vs Other Databases

### DuckDB vs SQLite

**DuckDB:**
- Optimized for analytical queries (OLAP)
- Columnar storage
- Vectorized execution engine
- Better for aggregations, joins on large datasets
- Rich analytical SQL features (window functions, complex types)

**SQLite:**
- Optimized for transactional queries (OLTP)
- Row-based storage
- Better for point lookups and updates
- Simpler SQL dialect
- More mature, widely deployed

**Use DuckDB when:** Analyzing data, running aggregations, data science
**Use SQLite when:** Building applications with transactional needs

### DuckDB vs PostgreSQL

**DuckDB:**
- Embedded, no server needed
- Single-user by default
- Optimized for analytics
- Direct file querying (CSV, Parquet)
- Zero administration

**PostgreSQL:**
- Client-server architecture
- Multi-user, high concurrency
- ACID transactions at scale
- Production web applications
- Requires server management

**Use DuckDB when:** Local analysis, data science, single-user analytics
**Use PostgreSQL when:** Multi-user applications, high-concurrency, production services

### DuckDB vs Pandas

**DuckDB:**
- SQL interface
- Handles larger-than-memory datasets
- More efficient for groupby/join operations
- Can query files without loading into memory
- Better multi-threading

**Pandas:**
- Python DataFrame API
- Rich ecosystem
- Better for data manipulation
- More flexible for complex transformations
- In-memory processing

**Best approach:** Use both together! DuckDB can query pandas DataFrames:

```python
import duckdb
import pandas as pd

df = pd.read_csv('data.csv')

# Query pandas DataFrame with SQL
result = duckdb.query("SELECT * FROM df WHERE age > 30").to_df()
```

### DuckDB vs Apache Spark

**DuckDB:**
- Single machine
- Embedded, lightweight
- Faster for small-to-medium datasets
- No cluster management
- Easy to use

**Apache Spark:**
- Distributed computing
- Scales to petabytes
- Requires cluster setup
- More complex operations
- Higher operational overhead

**Use DuckDB when:** Data fits on one machine, quick analysis needed
**Use Spark when:** Multi-node distributed processing required

---

## Quick Reference

### Essential Commands

```sql
-- Database operations
.open database.db          -- Open database
.tables                    -- List tables
.schema                    -- Show all schemas
.quit                      -- Exit

-- Display settings
.mode duckbox              -- Set output format
.headers on                -- Show column headers
.timer on                  -- Show query timing

-- Import/Export
.import file.csv table     -- Import CSV
.read script.sql           -- Execute SQL file
.output results.txt        -- Redirect output

-- Configuration
SET memory_limit='4GB';    -- Set memory limit
SET threads=4;             -- Set thread count
```

### Common SQL Patterns

```sql
-- Query files directly
SELECT * FROM 'data.csv';

-- Aggregate and group
SELECT category, COUNT(*), AVG(price)
FROM products
GROUP BY category;

-- Window functions
SELECT
    name,
    sales,
    RANK() OVER (ORDER BY sales DESC) as rank
FROM employees;

-- Export results
COPY (SELECT * FROM table) TO 'output.parquet' (FORMAT PARQUET);
```

---

## Getting Help

- **Official Documentation**: https://duckdb.org/docs/
- **Discord Community**: https://discord.duckdb.org/
- **GitHub Issues**: https://github.com/duckdb/duckdb/issues
- **Stack Overflow**: Tag your questions with `duckdb`
- **CLI Help**: Type `.help` in the CLI

## Conclusion

DuckDB is a powerful tool for data analysis that combines the simplicity of SQLite with the analytical power of modern data warehouses. Its ability to query files directly, handle complex SQL, and deliver fast performance makes it ideal for data scientists, analysts, and developers working with analytical workloads.

Start with the CLI for quick data exploration, then integrate DuckDB into your Python, R, or other applications as needed. The learning curve is gentle, and the performance benefits are significant.

Happy querying!
