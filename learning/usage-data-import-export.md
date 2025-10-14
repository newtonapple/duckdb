# Data Import and Export Guide

A comprehensive guide for importing and exporting data in DuckDB from a user perspective.

## Table of Contents

1. [Overview](#overview)
2. [Reading CSV Files](#reading-csv-files)
3. [Reading Parquet Files](#reading-parquet-files)
4. [Reading JSON Files](#reading-json-files)
5. [Writing and Exporting Data](#writing-and-exporting-data)
6. [COPY Command](#copy-command)
7. [Working with Remote Files](#working-with-remote-files)
8. [Glob Patterns and Multiple Files](#glob-patterns-and-multiple-files)
9. [Schema Inference and Type Detection](#schema-inference-and-type-detection)
10. [Common Import Issues and Solutions](#common-import-issues-and-solutions)
11. [Performance Tips](#performance-tips)

## Overview

DuckDB provides powerful and flexible data import/export capabilities supporting multiple formats:
- **CSV** - Comma-separated values (most common text format)
- **Parquet** - Columnar binary format (efficient for analytics)
- **JSON/NDJSON** - JavaScript Object Notation and newline-delimited JSON
- And many more formats through extensions

DuckDB automatically handles compression formats like GZIP, ZSTD, and others without requiring special configuration.

## Reading CSV Files

### Basic CSV Reading

The simplest way to read a CSV file:

```sql
-- Direct file reference (auto-detects CSV format)
SELECT * FROM 'data/customers.csv';

-- Explicit read_csv function
SELECT * FROM read_csv('data/customers.csv');

-- With auto-detection enabled
SELECT * FROM read_csv_auto('data/customers.csv');
```

### Reading CSV with Inline Data

You can query CSV data directly without creating files:

```sql
-- Using FROM clause with VALUES
SELECT * FROM (VALUES
    ('Alice', 30),
    ('Bob', 25),
    ('Charlie', 35)
) AS people(name, age);
```

### Customizing CSV Parameters

Control delimiter, quotes, headers, and more:

```sql
-- Custom delimiter (pipe-separated)
SELECT * FROM read_csv('data/data.csv', delim='|');

-- No header row
SELECT * FROM read_csv('data/data.csv', header=false);

-- Custom quote and escape characters
SELECT * FROM read_csv('data/data.csv', quote='"', escape='\\');

-- Specify column names and types explicitly
SELECT * FROM read_csv('data/data.csv',
    columns={
        'id': 'INTEGER',
        'name': 'VARCHAR',
        'amount': 'DECIMAL(10,2)',
        'date': 'DATE'
    }
);

-- Skip rows at the beginning
SELECT * FROM read_csv('data/data.csv', skip=5);

-- Handle different newline formats
SELECT * FROM read_csv('data/data.csv', new_line='\r\n');
```

### CSV with NULL Values

```sql
-- Specify NULL representation
SELECT * FROM read_csv('data/data.csv', nullstr='N/A');

-- Multiple NULL representations
SELECT * FROM read_csv('data/data.csv', null_padding=true);
```

### CSV Date and Timestamp Formats

```sql
-- Custom date format
SELECT * FROM read_csv('data/data.csv', dateformat='%Y.%m.%d');

-- Custom timestamp format
SELECT * FROM read_csv('data/data.csv',
    dateformat='%Y-%m-%d',
    timestampformat='%Y-%m-%d %H:%M:%S'
);
```

### Reading Compressed CSV

DuckDB automatically detects and decompresses:

```sql
-- GZIP compressed
SELECT * FROM 'data/customers.csv.gz';

-- ZSTD compressed
SELECT * FROM 'data/customers.csv.zst';

-- No special syntax needed - compression is automatic
SELECT * FROM read_csv('data/large_file.csv.gz');
```

### Error Handling in CSV

```sql
-- Ignore errors and continue
SELECT * FROM read_csv('data/messy.csv', ignore_errors=true);

-- Strict mode (fail on any error)
SELECT * FROM read_csv('data/data.csv', strict_mode=true);

-- Unquoted escapes
SELECT * FROM read_csv('data/data.csv', quote='', escape='\\');
```

## Reading Parquet Files

Parquet is a columnar storage format ideal for analytical workloads.

### Basic Parquet Reading

```sql
-- Direct file reference
SELECT * FROM 'data/events.parquet';

-- Explicit parquet_scan function
SELECT * FROM parquet_scan('data/events.parquet');

-- Alternative: read_parquet
SELECT * FROM read_parquet('data/events.parquet');
```

### Parquet with Specific Columns

Parquet's columnar format allows reading only needed columns:

```sql
-- Only read specific columns (projection pushdown)
SELECT user_id, event_time
FROM 'data/events.parquet';

-- This is efficient - only those columns are read from disk
```

### Parquet Metadata

Query metadata without reading the full file:

```sql
-- Get file metadata
SELECT * FROM parquet_metadata('data/file.parquet');

-- Get schema information
SELECT * FROM parquet_schema('data/file.parquet');

-- Get file statistics
SELECT * FROM parquet_file_metadata('data/file.parquet');
```

### Reading Parquet Arrays

```sql
-- Read into a list/array
SELECT * FROM read_parquet([
    'data/file1.parquet',
    'data/file2.parquet'
]);
```

### Hive Partitioning

Parquet files organized in Hive-style partitions:

```sql
-- Hive partitioned structure: data/year=2023/month=01/data.parquet
SELECT * FROM read_parquet('data/**/*.parquet', hive_partitioning=true);

-- Partition columns are automatically added
SELECT year, month, *
FROM read_parquet('data/**/*.parquet', hive_partitioning=true);
```

## Reading JSON Files

DuckDB supports both regular JSON arrays and newline-delimited JSON (NDJSON).

### Basic JSON Reading

```sql
-- Auto-detect JSON format
SELECT * FROM 'data/users.json';

-- Explicit read_json_auto
SELECT * FROM read_json_auto('data/users.json');

-- Newline-delimited JSON
SELECT * FROM read_ndjson('data/events.ndjson');
```

### JSON with Schema

```sql
-- Specify column schema
SELECT * FROM read_json('data/users.json',
    columns={
        'id': 'INTEGER',
        'name': 'VARCHAR',
        'tags': 'VARCHAR[]'
    }
);

-- NDJSON with specific columns
SELECT * FROM read_ndjson('data/events.ndjson',
    columns={
        'event_id': 'INTEGER',
        'timestamp': 'TIMESTAMP',
        'user_data': 'JSON'
    }
);
```

### JSON Format Options

```sql
-- Array format (top-level JSON array)
SELECT * FROM read_json('data/array.json', format='array');

-- Newline-delimited format
SELECT * FROM read_json('data/logs.ndjson', format='newline_delimited');

-- Unstructured format (each line is a separate JSON object)
SELECT * FROM read_json('data/mixed.json', format='unstructured');

-- Auto-detect format
SELECT * FROM read_json('data/data.json', format='auto');
```

### JSON Detection Depth

Control how deeply DuckDB inspects JSON structure:

```sql
-- Maximum depth for type detection (default: unlimited)
SELECT * FROM read_json_auto('data/nested.json', maximum_depth=0);
-- Depth 0: Everything as JSON type

SELECT * FROM read_json_auto('data/nested.json', maximum_depth=1);
-- Depth 1: Detect first level

SELECT * FROM read_json_auto('data/nested.json', maximum_depth=3);
-- Depth 3: Detect up to 3 levels deep
```

### JSON with Nested Types

```sql
-- Arrays in JSON
SELECT id, unnest(tags) AS tag
FROM 'data/posts.json';

-- Structs/objects in JSON
SELECT id, user_data.name, user_data.email
FROM 'data/users.json';

-- Complex nested structures
SELECT * FROM read_json_auto('data/complex.json', maximum_depth=5);
```

### JSON Error Handling

```sql
-- Ignore malformed JSON lines
SELECT * FROM read_json('data/messy.ndjson',
    format='newline_delimited',
    ignore_errors=true
);
```

### JSON Records vs Values

```sql
-- Records (objects)
SELECT * FROM read_json('data.json', records=true);

-- Values (primitive types or arrays)
SELECT * FROM read_json('data.json', records=false);

-- Auto-detect
SELECT * FROM read_json('data.json', records='auto');
```

## Writing and Exporting Data

### Writing to CSV

```sql
-- Basic CSV export
COPY (SELECT * FROM customers) TO 'output/customers.csv';

-- With CSV options
COPY customers TO 'output/customers.csv' (
    HEADER true,
    DELIMITER ',',
    QUOTE '"'
);

-- Export query results
COPY (
    SELECT customer_id, SUM(amount) as total
    FROM orders
    GROUP BY customer_id
) TO 'output/totals.csv' (HEADER true);
```

### Writing to Parquet

```sql
-- Basic Parquet export
COPY customers TO 'output/customers.parquet' (FORMAT PARQUET);

-- With compression
COPY large_table TO 'output/compressed.parquet' (
    FORMAT PARQUET,
    COMPRESSION 'ZSTD'
);

-- Control row group size
COPY events TO 'output/events.parquet' (
    FORMAT PARQUET,
    ROW_GROUP_SIZE 100000
);

-- Multiple row groups per file
COPY big_data TO 'output/data.parquet' (
    FORMAT PARQUET,
    ROW_GROUP_SIZE 50000,
    ROW_GROUPS_PER_FILE 5
);
```

### Writing to JSON

```sql
-- Array format (single JSON array)
COPY users TO 'output/users.json' (FORMAT JSON, ARRAY true);

-- Newline-delimited JSON (one object per line)
COPY events TO 'output/events.ndjson' (FORMAT JSON, ARRAY false);
```

### Writing with Compression

```sql
-- GZIP compressed CSV
COPY data TO 'output/data.csv.gz' (FORMAT CSV);

-- ZSTD compressed Parquet
COPY data TO 'output/data.parquet.zst' (FORMAT PARQUET);

-- Explicit compression setting
COPY data TO 'output/data.csv' (
    FORMAT CSV,
    COMPRESSION 'GZIP'
);
```

### Partitioned Writes

```sql
-- Partition by column
COPY events TO 'output/events' (
    FORMAT PARQUET,
    PARTITION_BY date
);
-- Creates: output/events/date=2023-01-01/data.parquet
--          output/events/date=2023-01-02/data.parquet

-- Multiple partition columns
COPY sales TO 'output/sales' (
    FORMAT PARQUET,
    PARTITION_BY (year, month)
);
```

### Per-Thread Output

```sql
-- Write separate files per thread (faster for large datasets)
COPY big_table TO 'output/data' (
    FORMAT PARQUET,
    PER_THREAD_OUTPUT true
);
-- Creates: output/data_0.parquet, output/data_1.parquet, etc.
```

### File Size Control

```sql
-- Limit file size
COPY large_table TO 'output/chunks' (
    FORMAT CSV,
    FILE_SIZE_BYTES '100MB'
);
-- Automatically creates multiple files when size limit is reached

-- With units
COPY data TO 'output/data' (
    FORMAT PARQUET,
    FILE_SIZE_BYTES '500MB'
);
```

### Custom Filename Patterns

```sql
-- Use UUID for filenames
COPY data TO 'output/data' (
    FORMAT PARQUET,
    FILENAME_PATTERN '{uuid}'
);

-- With custom pattern
COPY data TO 'output/data_part_{i}' (FORMAT CSV);
```

## COPY Command

The COPY command is the primary way to bulk load and export data.

### COPY FROM (Import)

```sql
-- Import CSV into existing table
CREATE TABLE customers (id INTEGER, name VARCHAR, email VARCHAR);
COPY customers FROM 'data/customers.csv';

-- Import with format specification
COPY orders FROM 'data/orders.csv' (FORMAT CSV, DELIMITER '|');

-- Import Parquet
COPY events FROM 'data/events.parquet' (FORMAT PARQUET);

-- Import specific columns
COPY customers (id, name) FROM 'data/partial.csv';
```

### COPY TO (Export)

```sql
-- Export entire table
COPY customers TO 'output/customers.parquet';

-- Export query results
COPY (
    SELECT c.name, COUNT(o.id) as order_count
    FROM customers c
    LEFT JOIN orders o ON c.id = o.customer_id
    GROUP BY c.name
) TO 'output/customer_orders.csv';

-- Export with options
COPY products TO 'output/products.csv' (
    HEADER true,
    DELIMITER ',',
    FORCE_QUOTE *
);
```

### COPY with Statistics

```sql
-- Return statistics about the export
COPY large_table TO 'output/data.parquet' (
    FORMAT PARQUET,
    RETURN_STATS
);
-- Returns: min, max, null_count for each column

-- Return list of files created
COPY data TO 'output/data.parquet' (
    FORMAT PARQUET,
    PER_THREAD_OUTPUT true,
    RETURN_FILES
);
-- Returns: list of all files written
```

### COPY Options Summary

Common options across formats:

```sql
-- Format specification
FORMAT CSV | PARQUET | JSON

-- Compression
COMPRESSION 'GZIP' | 'ZSTD' | 'SNAPPY' | 'NONE'

-- Output control
PER_THREAD_OUTPUT true | false
FILE_SIZE_BYTES '100MB'
FILENAME_PATTERN 'custom_{i}'

-- Partitioning
PARTITION_BY column_name
PARTITION_BY (col1, col2)

-- Metadata
RETURN_STATS
RETURN_FILES

-- File handling
USE_TMP_FILE true | false
OVERWRITE_OR_IGNORE true | false
```

## Working with Remote Files

DuckDB can read files over HTTP/HTTPS and S3 with the `httpfs` extension.

### HTTP/HTTPS Files

```sql
-- Read CSV from URL
SELECT * FROM 'https://example.com/data/customers.csv';

-- Read Parquet from URL
SELECT * FROM read_parquet('https://example.com/data/events.parquet');

-- Read JSON from GitHub
SELECT * FROM read_json_auto('https://raw.githubusercontent.com/user/repo/main/data.json');
```

### Enable HTTPFS Extension

```sql
-- Install and load httpfs extension (for S3 and advanced HTTP features)
INSTALL httpfs;
LOAD httpfs;
```

### S3 Files

```sql
-- Configure S3 credentials
SET s3_region='us-east-1';
SET s3_access_key_id='your_access_key';
SET s3_secret_access_key='your_secret_key';

-- Read from S3
SELECT * FROM 's3://bucket-name/path/to/data.parquet';

-- Read with wildcards
SELECT * FROM 's3://bucket-name/data/*.parquet';
```

### Remote File Performance

```sql
-- Set HTTP timeout (milliseconds)
SET http_timeout=120000;

-- Set number of retries
SET http_retries=3;

-- Enable HTTP keep-alive
SET http_keep_alive=true;
```

## Glob Patterns and Multiple Files

Read multiple files at once using glob patterns.

### Basic Glob Patterns

```sql
-- Read all CSV files in a directory
SELECT * FROM 'data/*.csv';

-- Read all Parquet files recursively
SELECT * FROM 'data/**/*.parquet';

-- Wildcard in filename
SELECT * FROM 'data/sales_202?.csv';

-- Character ranges
SELECT * FROM 'data/file[0-9].parquet';
```

### Glob Pattern Examples

```sql
-- All files in specific subdirectories
SELECT * FROM 'data/glob*/t?.parquet';

-- Multiple wildcards
SELECT * FROM 'data/**/dir/*.parquet';

-- Forward slashes work on all platforms
SELECT * FROM 'data/2023/*/sales.csv';
```

### Union by Name

When files have different schemas:

```sql
-- Union files with different columns
SELECT * FROM read_csv('data/*.csv', union_by_name=true);

-- Missing columns are filled with NULL
-- Extra columns are included in the result
```

### Recursive File Reading

```sql
-- Read all files recursively from a directory
SELECT * FROM read_csv('data/**/*.csv');

-- Parquet files in nested directories
SELECT * FROM read_parquet('data/**/*.parquet');
```

### Filename Metadata

```sql
-- Include filename in results
SELECT filename, *
FROM read_csv('data/*.csv', filename=true);

-- Include file number
SELECT file_row_number, *
FROM read_parquet('data/*.parquet');
```

## Schema Inference and Type Detection

DuckDB automatically detects column types and CSV dialects.

### Automatic Type Detection

```sql
-- Full auto-detection (types, delimiter, headers)
SELECT * FROM read_csv_auto('data/unknown.csv');

-- Auto-detect for JSON
SELECT * FROM read_json_auto('data/data.json');

-- Auto-detect for Parquet (schema is embedded)
SELECT * FROM 'data/file.parquet';
```

### CSV Sniffer

Analyze CSV structure without loading data:

```sql
-- Sniff CSV to determine parameters
SELECT * FROM sniff_csv('data/mystery.csv');

-- Returns: delimiter, quote, escape, columns with types, etc.

-- With hints
SELECT * FROM sniff_csv('data/data.csv',
    delim='|',
    header=true
);
```

### Controlling Type Detection

```sql
-- Sample size for type detection (number of rows)
SELECT * FROM read_csv_auto('data/large.csv', sample_size=10000);

-- All rows as VARCHAR (disable type detection)
SELECT * FROM read_csv('data/data.csv', all_varchar=true);

-- Then cast as needed
SELECT
    id::INTEGER,
    amount::DECIMAL(10,2),
    date::DATE
FROM read_csv('data/data.csv', all_varchar=true);
```

### Date Format Detection

```sql
-- DuckDB auto-detects common date formats
SELECT * FROM read_csv_auto('data/dates.csv');

-- Manual date format
SELECT * FROM read_csv('data/dates.csv',
    dateformat='%d/%m/%Y',
    timestampformat='%d/%m/%Y %H:%M:%S'
);
```

### Common Date Formats

```
%Y - 4-digit year (2023)
%y - 2-digit year (23)
%m - Month (01-12)
%d - Day (01-31)
%H - Hour (00-23)
%M - Minute (00-59)
%S - Second (00-59)

Examples:
'%Y-%m-%d' - 2023-01-15
'%d/%m/%Y' - 15/01/2023
'%Y.%m.%d' - 2023.01.15
'%Y-%m-%d %H:%M:%S' - 2023-01-15 14:30:00
```

### Type Inference Options

```sql
-- Maximum sample size for inference
SELECT * FROM read_csv('data.csv',
    auto_detect=true,
    sample_size=100000
);

-- Detect types but use custom column names
SELECT * FROM read_csv('data.csv',
    header=false,
    names=['id', 'name', 'value']
);
```

## Common Import Issues and Solutions

### Issue 1: Schema Mismatch

```sql
-- Error: Column count doesn't match

-- Solution 1: Allow padding with NULLs
SELECT * FROM read_csv('data.csv', null_padding=true);

-- Solution 2: Specify exact columns
SELECT * FROM read_csv('data.csv',
    columns={'col1': 'INTEGER', 'col2': 'VARCHAR'}
);
```

### Issue 2: Encoding Problems

```sql
-- Error: Invalid UTF-8 characters

-- Solution: Specify encoding
SELECT * FROM read_csv('data.csv', encoding='latin1');

-- Common encodings: UTF-8, latin1, ISO-8859-1
```

### Issue 3: Quote/Escape Conflicts

```sql
-- Error: Unterminated quotes

-- Solution 1: Disable quotes
SELECT * FROM read_csv('data.csv', quote='');

-- Solution 2: Adjust escape character
SELECT * FROM read_csv('data.csv', quote='"', escape='\\');

-- Solution 3: Use ignore_errors
SELECT * FROM read_csv('data.csv', ignore_errors=true);
```

### Issue 4: Wrong Delimiter Detection

```sql
-- Auto-detection picked wrong delimiter

-- Solution: Specify explicitly
SELECT * FROM read_csv('data.csv', delim='|');
SELECT * FROM read_csv('data.csv', delim='\t'); -- Tab
SELECT * FROM read_csv('data.csv', delim=';'); -- Semicolon
```

### Issue 5: Type Conversion Errors

```sql
-- Error: Could not convert string to INTEGER

-- Solution 1: Read as VARCHAR first
CREATE TABLE temp AS
SELECT * FROM read_csv('data.csv', all_varchar=true);

-- Then clean and convert
SELECT
    TRY_CAST(id AS INTEGER) as id,
    TRY_CAST(amount AS DOUBLE) as amount
FROM temp
WHERE TRY_CAST(id AS INTEGER) IS NOT NULL;

-- Solution 2: Use ignore_errors
SELECT * FROM read_csv('data.csv', ignore_errors=true);
```

### Issue 6: Header vs No Header Confusion

```sql
-- Data has no header but auto-detect thinks it does

-- Solution: Disable header
SELECT * FROM read_csv('data.csv', header=false);

-- Data has header but auto-detect missed it
SELECT * FROM read_csv('data.csv', header=true);
```

### Issue 7: Line Endings

```sql
-- Mixed or unusual line endings

-- Solution: Specify line ending
SELECT * FROM read_csv('data.csv', new_line='\r\n'); -- Windows
SELECT * FROM read_csv('data.csv', new_line='\n');   -- Unix/Linux
SELECT * FROM read_csv('data.csv', new_line='\r');   -- Old Mac
```

### Issue 8: Large Field Sizes

```sql
-- Error: Maximum line size exceeded

-- Solution: Increase maximum field size
SELECT * FROM read_json('data.json',
    maximum_object_size=104857600  -- 100MB
);
```

### Issue 9: Files Not Found

```sql
-- Error: No files found that match the pattern

-- Solution 1: Use absolute paths
SELECT * FROM '/absolute/path/to/data.csv';

-- Solution 2: Check glob pattern
SELECT * FROM 'data/**/*.csv';  -- Recursive

-- Solution 3: Verify working directory
SELECT current_setting('file_search_path');
```

### Issue 10: Memory Issues with Large Files

```sql
-- Out of memory with large CSV

-- Solution 1: Stream with limited sample
CREATE TABLE data AS
SELECT * FROM read_csv('huge.csv') LIMIT 1000000;

-- Solution 2: Use Parquet instead (more efficient)
-- Convert first:
COPY (SELECT * FROM read_csv('huge.csv'))
TO 'huge.parquet' (FORMAT PARQUET);

-- Then read Parquet
SELECT * FROM 'huge.parquet';
```

## Performance Tips

### 1. Use Parquet for Large Datasets

```sql
-- CSV is slower for repeated queries
SELECT * FROM 'large_data.csv';  -- Slow

-- Convert to Parquet once
COPY (SELECT * FROM 'large_data.csv')
TO 'large_data.parquet';

-- Then use Parquet
SELECT * FROM 'large_data.parquet';  -- Much faster
```

### 2. Projection Pushdown

```sql
-- Only select columns you need (especially with Parquet)
SELECT user_id, event_time
FROM 'events.parquet';  -- Only reads those columns

-- Avoid SELECT *
SELECT * FROM 'events.parquet';  -- Reads all columns
```

### 3. Filter Pushdown

```sql
-- Filters are pushed down to file readers
SELECT *
FROM 'events.parquet'
WHERE date = '2023-01-01';  -- Filter applied during read

-- Use partition columns for best performance
SELECT *
FROM read_parquet('data/**/*.parquet', hive_partitioning=true)
WHERE year = 2023 AND month = 1;  -- Skips entire files
```

### 4. Parallel Reading

```sql
-- DuckDB automatically parallelizes file reading
-- For best performance:

-- 1. Use multiple smaller files instead of one huge file
SELECT * FROM 'data/*.parquet';  -- Reads files in parallel

-- 2. Increase thread count if needed
SET threads=8;
```

### 5. Compression Trade-offs

```sql
-- Reading compressed files
-- GZIP: Slower read, better compression
-- ZSTD: Faster read, good compression
-- SNAPPY: Fastest read, moderate compression

-- For write-once, read-many: Use ZSTD or SNAPPY
COPY data TO 'output.parquet' (
    FORMAT PARQUET,
    COMPRESSION 'ZSTD'
);
```

### 6. Row Group Size (Parquet)

```sql
-- Smaller row groups: Better filtering, more overhead
-- Larger row groups: Less overhead, less selective filtering

-- For analytical queries (default is usually good):
COPY data TO 'output.parquet' (
    FORMAT PARQUET,
    ROW_GROUP_SIZE 122880  -- ~122K rows
);

-- For very selective queries:
COPY data TO 'output.parquet' (
    FORMAT PARQUET,
    ROW_GROUP_SIZE 10000  -- Smaller for better filtering
);
```

### 7. Avoid Type Casting During Reads

```sql
-- Slow: Read as wrong type then cast
SELECT amount::DECIMAL(10,2)
FROM read_csv('data.csv');

-- Fast: Specify correct type upfront
SELECT amount
FROM read_csv('data.csv',
    columns={'amount': 'DECIMAL(10,2)'}
);
```

### 8. Use COPY for Bulk Operations

```sql
-- Slow: INSERT with SELECT
CREATE TABLE target (id INTEGER, name VARCHAR);
INSERT INTO target SELECT * FROM 'source.csv';

-- Faster: Use COPY
COPY target FROM 'source.csv';

-- Fastest: CREATE TABLE AS
CREATE TABLE target AS SELECT * FROM 'source.csv';
```

### 9. Partition Large Exports

```sql
-- Writing huge table to single file: Slow
COPY huge_table TO 'output.parquet';

-- Partitioned write: Much faster
COPY huge_table TO 'output' (
    FORMAT PARQUET,
    PARTITION_BY date,
    PER_THREAD_OUTPUT true
);
```

### 10. Disable Unnecessary Features

```sql
-- When schema is known, skip detection
SELECT * FROM read_csv('data.csv',
    auto_detect=false,
    columns={'id': 'INTEGER', 'name': 'VARCHAR'}
);

-- Skip sample size when not needed
SELECT * FROM read_csv('data.csv', sample_size=1000);
```

### 11. Use Direct File Paths

```sql
-- Glob patterns require scanning directories
SELECT * FROM 'data/*.csv';  -- Scans directory first

-- Direct paths are faster when you know the files
SELECT * FROM 'data/file1.csv'
UNION ALL
SELECT * FROM 'data/file2.csv'
UNION ALL
SELECT * FROM 'data/file3.csv';
```

### 12. Remote File Optimization

```sql
-- For remote files, enable HTTP keep-alive
SET http_keep_alive=true;
SET http_timeout=300000;  -- 5 minutes

-- Consider caching frequently-accessed remote files locally
COPY (SELECT * FROM 'https://example.com/data.parquet')
TO 'local_cache.parquet';

-- Then query local copy
SELECT * FROM 'local_cache.parquet';
```

## Summary

DuckDB provides flexible, high-performance data import/export capabilities:

- **CSV**: Universal text format with auto-detection
- **Parquet**: Columnar format ideal for analytics
- **JSON/NDJSON**: Flexible for semi-structured data
- **Remote files**: HTTP/HTTPS and S3 support
- **Glob patterns**: Read multiple files easily
- **Auto-detection**: Smart schema inference
- **Performance**: Columnar storage, parallel processing, compression

### Quick Reference Card

```sql
-- Reading
SELECT * FROM 'file.csv';                    -- Auto-detect CSV
SELECT * FROM read_csv('file.csv');          -- Explicit CSV
SELECT * FROM 'file.parquet';                -- Parquet
SELECT * FROM read_json_auto('file.json');   -- JSON
SELECT * FROM 'data/*.csv';                  -- Multiple files
SELECT * FROM 'https://url/data.csv';        -- Remote file

-- Writing
COPY table TO 'output.csv';                  -- Export CSV
COPY table TO 'output.parquet';              -- Export Parquet
COPY (SELECT ...) TO 'output.json' (ARRAY);  -- Export JSON

-- With options
COPY table TO 'out.csv' (DELIMITER '|', HEADER true);
COPY table TO 'out.parquet' (COMPRESSION 'ZSTD');

-- Import to table
COPY table FROM 'file.csv';
CREATE TABLE t AS SELECT * FROM 'file.csv';
```

---

*Last updated: 2025-10-14*
