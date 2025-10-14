# DuckDB Client APIs Usage Guide

This guide covers using DuckDB from various programming languages through their respective client APIs. DuckDB provides native support for Python, R, Java/JDBC, Node.js, C, C++, Julia, Swift, and more.

## Table of Contents

1. [Python API](#1-python-api)
2. [C++ API](#2-c-api-1)
3. [C API](#3-c-api)
4. [Java/JDBC API](#4-javajdbc-api)
5. [Node.js API](#5-nodejs-api)
6. [R API](#6-r-api)
7. [Connection Management](#7-connection-management)
8. [Parameterized Queries](#8-parameterized-queries)
9. [Fetching Results](#9-fetching-results)
10. [Working with Pandas and Arrow](#10-working-with-pandas-and-arrow)
11. [Best Practices](#11-best-practices)

---

## 1. Python API

The Python API is the most feature-rich client API for DuckDB, supporting both SQL execution and the Relation API for programmatic querying.

### Installation

```bash
pip install duckdb
```

### Basic Connection and Queries

```python
import duckdb

# Connect to an in-memory database
conn = duckdb.connect()

# Or connect to a persistent database file
# conn = duckdb.connect('my_database.db')

# Create a table
conn.execute("CREATE TABLE users (id INTEGER, name STRING, age INTEGER)")

# Insert data
conn.execute("INSERT INTO users VALUES (1, 'Alice', 30)")

# Query data
result = conn.execute("SELECT * FROM users").fetchall()
print(result)
```

### Using Parameterized Queries

```python
# Using positional parameters with ?
conn.execute("INSERT INTO users VALUES (?, ?, ?)", [2, 'Bob', 25])

# Using executemany for batch inserts
conn.executemany(
    "INSERT INTO users VALUES (?, ?, ?)",
    [[3, 'Charlie', 35], [4, 'Diana', 28]]
)

# Query with parameters
result = conn.execute(
    "SELECT * FROM users WHERE age > ?",
    [25]
).fetchall()
```

### Working with Pandas DataFrames

```python
import pandas as pd

# Create a pandas DataFrame
df = pd.DataFrame({
    'id': [5, 6, 7],
    'name': ['Eve', 'Frank', 'Grace'],
    'age': [32, 29, 31]
})

# Register DataFrame as a view (zero-copy)
conn.register('users_df', df)

# Query the DataFrame directly
result = conn.execute("SELECT * FROM users_df WHERE age > 30").fetchdf()
print(result)

# Or use the DataFrame directly in queries
result = conn.execute("SELECT * FROM df WHERE age > 30").fetchdf()
```

### Relation API (Programmatic Querying)

The Relation API provides a pandas-like interface for building queries programmatically:

```python
# Create a relation from a table
rel = conn.table('users')

# Chain operations
result = (rel
    .filter('age > 25')
    .project('id, name, age + 1 AS next_age')
    .order('name')
    .limit(5))

# Execute and fetch results
df = result.fetchdf()

# Or convert to pandas DataFrame
df = rel.to_df()

# Create relation from DataFrame
rel = conn.from_df(df)

# Create relation from CSV
rel = duckdb.from_csv_auto('data.csv')

# Aggregate operations
agg_result = rel.aggregate('age, COUNT(*) as count, AVG(age) as avg_age', 'age')
print(agg_result.fetchall())

# Join operations
users_rel = conn.table('users')
orders_rel = conn.table('orders')
joined = users_rel.join(orders_rel, 'users.id = orders.user_id')
```

### Working with Arrow

```python
import pyarrow as pa

# Export query result as Arrow table
arrow_table = conn.execute("SELECT * FROM users").arrow()

# Query Arrow tables directly
conn.register('arrow_view', arrow_table)
result = conn.execute("SELECT * FROM arrow_view").fetchall()

# Scan Arrow table
conn.execute("CREATE TABLE users_from_arrow AS SELECT * FROM arrow_view")
```

### Context Manager Pattern

```python
# Use context manager for automatic cleanup
with duckdb.connect('my_database.db') as conn:
    conn.execute("CREATE TABLE temp_data (x INTEGER)")
    conn.execute("INSERT INTO temp_data VALUES (1), (2), (3)")
    result = conn.execute("SELECT * FROM temp_data").fetchall()
# Connection automatically closed
```

### Cursor API (DB-API 2.0 Compatible)

```python
# Create a cursor (PEP 249 compliant)
cursor = conn.cursor()

cursor.execute("SELECT * FROM users WHERE age > ?", [25])
rows = cursor.fetchall()

# Iterate over results
for row in cursor:
    print(row)

cursor.close()
```

---

## 2. C++ API

The C++ API provides a high-level, object-oriented interface to DuckDB.

### Basic Usage

```cpp
#include "duckdb.hpp"

using namespace duckdb;

int main() {
    // Create an in-memory database
    DuckDB db(nullptr);

    // Or create/open a persistent database
    // DuckDB db("my_database.db");

    // Create a connection
    Connection con(db);

    // Execute DDL
    con.Query("CREATE TABLE users (id INTEGER, name VARCHAR, age INTEGER)");

    // Execute DML
    con.Query("INSERT INTO users VALUES (1, 'Alice', 30), (2, 'Bob', 25)");

    // Query and get results
    auto result = con.Query("SELECT * FROM users WHERE age > 25");

    // Print results
    result->Print();

    return 0;
}
```

### Working with Query Results

```cpp
auto result = con.Query("SELECT * FROM users");

// Check for errors
if (result->HasError()) {
    std::cerr << "Query error: " << result->GetError() << std::endl;
    return 1;
}

// Iterate through rows
for (size_t row_idx = 0; row_idx < result->RowCount(); row_idx++) {
    auto id = result->GetValue(0, row_idx);
    auto name = result->GetValue(1, row_idx);
    auto age = result->GetValue(2, row_idx);

    std::cout << "ID: " << id.ToString()
              << ", Name: " << name.ToString()
              << ", Age: " << age.ToString() << std::endl;
}

// Access column information
size_t column_count = result->ColumnCount();
for (size_t i = 0; i < column_count; i++) {
    std::cout << "Column " << i << ": " << result->ColumnName(i) << std::endl;
}
```

### Prepared Statements

```cpp
// Prepare a statement
auto prepared = con.Prepare("INSERT INTO users VALUES (?, ?, ?)");

if (prepared->HasError()) {
    std::cerr << "Prepare error: " << prepared->GetError() << std::endl;
    return 1;
}

// Execute with parameters
auto result = prepared->Execute(3, "Charlie", 35);

// Execute multiple times with different parameters
prepared->Execute(4, "Diana", 28);
prepared->Execute(5, "Eve", 32);
```

### Appender for Bulk Inserts

```cpp
// Create an appender for efficient bulk inserts
Appender appender(con, "users");

// Append rows
appender.BeginRow();
appender.Append(6);
appender.Append("Frank");
appender.Append(29);
appender.EndRow();

appender.BeginRow();
appender.Append(7);
appender.Append("Grace");
appender.Append(31);
appender.EndRow();

// Flush to commit the data
appender.Close();
```

---

## 3. C API

The C API provides low-level access to DuckDB functionality and is used by many language bindings.

### Basic Usage

```c
#include "duckdb.h"
#include <stdio.h>
#include <stdlib.h>

int main() {
    duckdb_database db = NULL;
    duckdb_connection con = NULL;
    duckdb_result result;

    // Open database (NULL for in-memory)
    if (duckdb_open(NULL, &db) == DuckDBError) {
        fprintf(stderr, "Failed to open database\n");
        return 1;
    }

    // Create connection
    if (duckdb_connect(db, &con) == DuckDBError) {
        fprintf(stderr, "Failed to connect\n");
        duckdb_close(&db);
        return 1;
    }

    // Execute query
    if (duckdb_query(con, "CREATE TABLE users (id INTEGER, name VARCHAR)", NULL)
        == DuckDBError) {
        fprintf(stderr, "Failed to create table\n");
        goto cleanup;
    }

    // Insert data
    if (duckdb_query(con, "INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob')", NULL)
        == DuckDBError) {
        fprintf(stderr, "Failed to insert data\n");
        goto cleanup;
    }

    // Query with results
    if (duckdb_query(con, "SELECT * FROM users", &result) == DuckDBError) {
        fprintf(stderr, "Query failed: %s\n", duckdb_result_error(&result));
        duckdb_destroy_result(&result);
        goto cleanup;
    }

    // Process results
    idx_t row_count = duckdb_row_count(&result);
    idx_t column_count = duckdb_column_count(&result);

    // Print column names
    for (idx_t col = 0; col < column_count; col++) {
        printf("%s\t", duckdb_column_name(&result, col));
    }
    printf("\n");

    // Print data
    for (idx_t row = 0; row < row_count; row++) {
        for (idx_t col = 0; col < column_count; col++) {
            char *val = duckdb_value_varchar(&result, col, row);
            printf("%s\t", val ? val : "NULL");
            duckdb_free(val);
        }
        printf("\n");
    }

    duckdb_destroy_result(&result);

cleanup:
    duckdb_disconnect(&con);
    duckdb_close(&db);
    return 0;
}
```

### Prepared Statements

```c
duckdb_prepared_statement stmt = NULL;
duckdb_result result;

// Prepare statement
if (duckdb_prepare(con, "INSERT INTO users VALUES (?, ?)", &stmt) == DuckDBError) {
    fprintf(stderr, "Prepare failed: %s\n", duckdb_prepare_error(stmt));
    duckdb_destroy_prepare(&stmt);
    return 1;
}

// Bind parameters
duckdb_bind_int32(stmt, 1, 3);
duckdb_bind_varchar(stmt, 2, "Charlie");

// Execute
if (duckdb_execute_prepared(stmt, &result) == DuckDBError) {
    fprintf(stderr, "Execute failed: %s\n", duckdb_result_error(&result));
    duckdb_destroy_result(&result);
    duckdb_destroy_prepare(&stmt);
    return 1;
}

duckdb_destroy_result(&result);
duckdb_destroy_prepare(&stmt);
```

### Type-Specific Value Access

```c
// Get values by type
for (idx_t row = 0; row < row_count; row++) {
    // Integer column
    int32_t id = duckdb_value_int32(&result, 0, row);

    // String column (must be freed)
    char *name = duckdb_value_varchar(&result, 1, row);

    // Check for NULL
    bool is_null = duckdb_value_is_null(&result, 1, row);

    printf("ID: %d, Name: %s\n", id, name ? name : "NULL");

    duckdb_free(name);
}
```

### Appender for Bulk Loading

```c
duckdb_appender appender;

// Create appender
if (duckdb_appender_create(con, NULL, "users", &appender) == DuckDBError) {
    fprintf(stderr, "Failed to create appender\n");
    return 1;
}

// Append rows
duckdb_appender_begin_row(appender);
duckdb_append_int32(appender, 4);
duckdb_append_varchar(appender, "Diana");
duckdb_appender_end_row(appender);

duckdb_appender_begin_row(appender);
duckdb_append_int32(appender, 5);
duckdb_append_varchar(appender, "Eve");
duckdb_appender_end_row(appender);

// Flush and close
duckdb_appender_flush(appender);
duckdb_appender_destroy(&appender);
```

---

## 4. Java/JDBC API

DuckDB provides a JDBC driver for Java applications.

### Installation

Add to your Maven `pom.xml`:

```xml
<dependency>
    <groupId>org.duckdb</groupId>
    <artifactId>duckdb_jdbc</artifactId>
    <version>0.9.2</version>
</dependency>
```

Or Gradle:

```gradle
implementation 'org.duckdb:duckdb_jdbc:0.9.2'
```

### Basic Usage

```java
import java.sql.*;

public class DuckDBExample {
    public static void main(String[] args) {
        try {
            // Load the driver (optional in newer Java versions)
            Class.forName("org.duckdb.DuckDBDriver");

            // Connect to in-memory database
            Connection conn = DriverManager.getConnection("jdbc:duckdb:");

            // Or connect to file database
            // Connection conn = DriverManager.getConnection("jdbc:duckdb:my_database.db");

            // Create table
            Statement stmt = conn.createStatement();
            stmt.execute("CREATE TABLE users (id INTEGER, name VARCHAR, age INTEGER)");

            // Insert data
            stmt.execute("INSERT INTO users VALUES (1, 'Alice', 30), (2, 'Bob', 25)");

            // Query data
            ResultSet rs = stmt.executeQuery("SELECT * FROM users");

            while (rs.next()) {
                int id = rs.getInt("id");
                String name = rs.getString("name");
                int age = rs.getInt("age");
                System.out.println("ID: " + id + ", Name: " + name + ", Age: " + age);
            }

            rs.close();
            stmt.close();
            conn.close();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### Prepared Statements

```java
// Prepare statement
PreparedStatement pstmt = conn.prepareStatement(
    "INSERT INTO users VALUES (?, ?, ?)"
);

// Bind parameters and execute
pstmt.setInt(1, 3);
pstmt.setString(2, "Charlie");
pstmt.setInt(3, 35);
pstmt.executeUpdate();

// Reuse with different parameters
pstmt.setInt(1, 4);
pstmt.setString(2, "Diana");
pstmt.setInt(3, 28);
pstmt.executeUpdate();

pstmt.close();
```

### Batch Operations

```java
PreparedStatement pstmt = conn.prepareStatement(
    "INSERT INTO users VALUES (?, ?, ?)"
);

// Add to batch
pstmt.setInt(1, 5);
pstmt.setString(2, "Eve");
pstmt.setInt(3, 32);
pstmt.addBatch();

pstmt.setInt(1, 6);
pstmt.setString(2, "Frank");
pstmt.setInt(3, 29);
pstmt.addBatch();

// Execute batch
int[] results = pstmt.executeBatch();
pstmt.close();
```

### Transactions

```java
try {
    // Disable auto-commit
    conn.setAutoCommit(false);

    Statement stmt = conn.createStatement();
    stmt.execute("INSERT INTO users VALUES (7, 'Grace', 31)");
    stmt.execute("INSERT INTO users VALUES (8, 'Henry', 27)");

    // Commit transaction
    conn.commit();

} catch (SQLException e) {
    // Rollback on error
    conn.rollback();
    e.printStackTrace();
} finally {
    conn.setAutoCommit(true);
}
```

### Working with Metadata

```java
// Get database metadata
DatabaseMetaData metaData = conn.getMetaData();
System.out.println("Database: " + metaData.getDatabaseProductName());
System.out.println("Version: " + metaData.getDatabaseProductVersion());

// Get table information
ResultSet tables = metaData.getTables(null, null, "users", null);
while (tables.next()) {
    System.out.println("Table: " + tables.getString("TABLE_NAME"));
}

// Get column information
ResultSet columns = metaData.getColumns(null, null, "users", null);
while (columns.next()) {
    String columnName = columns.getString("COLUMN_NAME");
    String columnType = columns.getString("TYPE_NAME");
    System.out.println("Column: " + columnName + " - Type: " + columnType);
}
```

---

## 5. Node.js API

DuckDB provides a Node.js module for JavaScript/TypeScript applications.

### Installation

```bash
npm install duckdb
```

### Basic Usage (Callback Style)

```javascript
const duckdb = require('duckdb');

// Create database (in-memory)
const db = new duckdb.Database(':memory:');

// Or create/open file database
// const db = new duckdb.Database('my_database.db');

// Get connection
db.all('SELECT * FROM range(10)', (err, res) => {
    if (err) {
        console.error(err);
        return;
    }
    console.log(res);
});

// Execute DDL
db.run('CREATE TABLE users (id INTEGER, name VARCHAR, age INTEGER)', (err) => {
    if (err) throw err;

    // Insert data
    db.run("INSERT INTO users VALUES (1, 'Alice', 30), (2, 'Bob', 25)", (err) => {
        if (err) throw err;

        // Query data
        db.all('SELECT * FROM users', (err, rows) => {
            if (err) throw err;
            console.log(rows);
        });
    });
});

// Close database
db.close();
```

### Promise-Based Usage

```javascript
const duckdb = require('duckdb');
const { promisify } = require('util');

const db = new duckdb.Database(':memory:');
const dbRun = promisify(db.run.bind(db));
const dbAll = promisify(db.all.bind(db));

async function main() {
    try {
        await dbRun('CREATE TABLE users (id INTEGER, name VARCHAR, age INTEGER)');
        await dbRun("INSERT INTO users VALUES (1, 'Alice', 30), (2, 'Bob', 25)");

        const rows = await dbAll('SELECT * FROM users WHERE age > 25');
        console.log(rows);

    } catch (err) {
        console.error(err);
    } finally {
        db.close();
    }
}

main();
```

### Prepared Statements

```javascript
db.run('CREATE TABLE users (id INTEGER, name VARCHAR, age INTEGER)', (err) => {
    if (err) throw err;

    // Prepare statement
    db.prepare('INSERT INTO users VALUES (?, ?, ?)', (err, stmt) => {
        if (err) throw err;

        // Execute with parameters
        stmt.run(3, 'Charlie', 35, (err) => {
            if (err) throw err;

            // Execute with different parameters
            stmt.run(4, 'Diana', 28, (err) => {
                if (err) throw err;

                stmt.finalize();
            });
        });
    });
});
```

### Streaming Results

```javascript
const connection = db.connect();

// Stream large result sets
const stream = connection.stream('SELECT * FROM large_table');

stream.on('data', (row) => {
    console.log(row);
});

stream.on('end', () => {
    console.log('Query complete');
});

stream.on('error', (err) => {
    console.error('Query error:', err);
});
```

### Arrow Integration

```javascript
// Query and return Arrow IPC format
db.arrowIPCAll('SELECT * FROM users', (err, buffer) => {
    if (err) throw err;

    // buffer contains Arrow IPC data
    // Can be used with Arrow libraries
    console.log('Arrow buffer size:', buffer.length);
});
```

---

## 6. R API

DuckDB provides an R package that integrates with the DBI interface.

### Installation

```r
install.packages("duckdb")
```

### Basic Usage

```r
library(DBI)
library(duckdb)

# Create an in-memory database
con <- dbConnect(duckdb::duckdb(), dbdir = ":memory:")

# Or connect to file database
# con <- dbConnect(duckdb::duckdb(), dbdir = "my_database.db")

# Create table
dbExecute(con, "CREATE TABLE users (id INTEGER, name VARCHAR, age INTEGER)")

# Insert data
dbExecute(con, "INSERT INTO users VALUES (1, 'Alice', 30), (2, 'Bob', 25)")

# Query data
result <- dbGetQuery(con, "SELECT * FROM users WHERE age > 25")
print(result)

# Disconnect
dbDisconnect(con, shutdown = TRUE)
```

### Working with Data Frames

```r
# Create a data frame
df <- data.frame(
    id = c(3, 4, 5),
    name = c('Charlie', 'Diana', 'Eve'),
    age = c(35, 28, 32)
)

# Write data frame to table
dbWriteTable(con, "users", df, append = TRUE)

# Read table into data frame
users_df <- dbReadTable(con, "users")
print(users_df)

# Register data frame as a view (zero-copy)
duckdb::duckdb_register(con, "users_view", df)

# Query the registered view
result <- dbGetQuery(con, "SELECT * FROM users_view WHERE age > 30")
```

### Prepared Statements

```r
# Create prepared statement
stmt <- dbSendStatement(con, "INSERT INTO users VALUES (?, ?, ?)")

# Bind and execute
dbBind(stmt, list(6, 'Frank', 29))
dbExecute(stmt)

# Execute with different parameters
dbBind(stmt, list(7, 'Grace', 31))
dbExecute(stmt)

# Clear statement
dbClearResult(stmt)
```

### Reading CSV Files

```r
# Read CSV directly with DuckDB
result <- dbGetQuery(con, "SELECT * FROM read_csv_auto('data.csv')")

# Or create a table from CSV
dbExecute(con, "CREATE TABLE data AS SELECT * FROM read_csv_auto('data.csv')")
```

### Integration with dplyr

```r
library(dplyr)

# Use dplyr with DuckDB connection
users_tbl <- tbl(con, "users")

# Chain dplyr operations
result <- users_tbl %>%
    filter(age > 25) %>%
    select(name, age) %>%
    arrange(desc(age)) %>%
    collect()

print(result)
```

---

## 7. Connection Management

### Connection Pooling (C++)

```cpp
// DuckDB automatically handles concurrent connections
DuckDB db("my_database.db");

// Create multiple connections
Connection con1(db);
Connection con2(db);

// Both connections can be used concurrently
con1.Query("SELECT * FROM table1");
con2.Query("SELECT * FROM table2");
```

### Configuration Options

#### Python

```python
# Configure memory limit and threads
conn = duckdb.connect(':memory:', config={
    'memory_limit': '2GB',
    'threads': 4,
    'default_order': 'DESC'
})

# Set configuration after connection
conn.execute("SET memory_limit='2GB'")
conn.execute("SET threads=4")
```

#### C

```c
duckdb_database db;
duckdb_config config;

// Create configuration
duckdb_create_config(&config);

// Set options
duckdb_set_config(config, "memory_limit", "2GB");
duckdb_set_config(config, "threads", "4");

// Open database with config
duckdb_open_ext(NULL, &db, config, NULL);

// Clean up config
duckdb_destroy_config(&config);
```

#### Java

```java
// Set configuration via connection properties
Properties props = new Properties();
props.setProperty("duckdb.memory_limit", "2GB");
props.setProperty("duckdb.threads", "4");

Connection conn = DriverManager.getConnection("jdbc:duckdb:", props);
```

### Thread Safety

- **Python**: Connections are not thread-safe. Use one connection per thread or implement locking.
- **C++**: Connections are not thread-safe, but Database instances can be shared across threads.
- **C**: Same as C++. Create separate connections for each thread.
- **Java**: JDBC connections are generally not thread-safe. Use connection pooling for multi-threaded applications.

---

## 8. Parameterized Queries

### Python

```python
# Positional parameters
conn.execute("SELECT * FROM users WHERE age > ? AND name LIKE ?", [25, 'A%'])

# Named parameters (via prepared statements)
stmt = conn.execute("PREPARE my_query AS SELECT * FROM users WHERE age > $1")
result = conn.execute("EXECUTE my_query(25)").fetchall()
```

### C

```c
duckdb_prepared_statement stmt;
duckdb_prepare(con, "SELECT * FROM users WHERE age > ? AND name LIKE ?", &stmt);

// Bind parameters (1-indexed)
duckdb_bind_int32(stmt, 1, 25);
duckdb_bind_varchar(stmt, 2, "A%");

duckdb_execute_prepared(stmt, &result);
```

### C++

```cpp
auto prepared = con.Prepare("SELECT * FROM users WHERE age > ? AND name LIKE ?");
auto result = prepared->Execute(25, "A%");
```

### Java

```java
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM users WHERE age > ? AND name LIKE ?"
);
pstmt.setInt(1, 25);
pstmt.setString(2, "A%");
ResultSet rs = pstmt.executeQuery();
```

### Node.js

```javascript
db.all('SELECT * FROM users WHERE age > ? AND name LIKE ?', [25, 'A%'],
    (err, rows) => {
        if (err) throw err;
        console.log(rows);
    }
);
```

---

## 9. Fetching Results

### Python Result Formats

```python
result = conn.execute("SELECT * FROM users")

# Fetch all rows as list of tuples
rows = result.fetchall()

# Fetch one row
row = result.fetchone()

# Fetch as pandas DataFrame
df = result.fetchdf()

# Fetch as dictionary
dict_result = result.fetchdf().to_dict('records')

# Fetch as NumPy arrays
numpy_result = result.fetchnumpy()

# Fetch as Arrow table
arrow_table = result.arrow()
```

### C Result Access

```c
duckdb_result result;
duckdb_query(con, "SELECT * FROM users", &result);

idx_t row_count = duckdb_row_count(&result);
idx_t col_count = duckdb_column_count(&result);

// Access by row and column
for (idx_t row = 0; row < row_count; row++) {
    for (idx_t col = 0; col < col_count; col++) {
        // Get as string (must free)
        char *val = duckdb_value_varchar(&result, col, row);
        printf("%s ", val);
        duckdb_free(val);
    }
    printf("\n");
}

duckdb_destroy_result(&result);
```

### Java ResultSet Processing

```java
ResultSet rs = stmt.executeQuery("SELECT * FROM users");

// Iterate through results
while (rs.next()) {
    int id = rs.getInt("id");
    String name = rs.getString("name");
    int age = rs.getInt("age");

    // Check for NULL
    if (rs.wasNull()) {
        System.out.println("Value was NULL");
    }
}

// Get metadata
ResultSetMetaData metaData = rs.getMetaData();
int columnCount = metaData.getColumnCount();
for (int i = 1; i <= columnCount; i++) {
    System.out.println("Column " + i + ": " + metaData.getColumnName(i));
}

rs.close();
```

---

## 10. Working with Pandas and Arrow

### Python: Zero-Copy Integration

```python
import pandas as pd
import pyarrow as pa

# Query directly to pandas (copy)
df = conn.execute("SELECT * FROM users").fetchdf()

# Register pandas DataFrame (zero-copy reference)
conn.register('my_df', df)
result = conn.execute("SELECT * FROM my_df WHERE age > 30").fetchall()

# Unregister when done
conn.unregister('my_df')
```

### Python: Arrow Integration

```python
import pyarrow as pa

# Get results as Arrow table
arrow_table = conn.execute("SELECT * FROM users").arrow()

# Register Arrow table
conn.register('arrow_users', arrow_table)

# Query Arrow data
result = conn.execute("SELECT * FROM arrow_users WHERE age > 30").fetchall()

# Read Parquet files (uses Arrow internally)
df = conn.execute("SELECT * FROM read_parquet('data.parquet')").fetchdf()

# Write to Parquet
conn.execute("COPY users TO 'output.parquet' (FORMAT PARQUET)")
```

### Python: Efficient Data Loading

```python
# Load large CSV files efficiently
conn.execute("CREATE TABLE data AS SELECT * FROM read_csv_auto('large_file.csv')")

# Load from pandas with copy=False for zero-copy when possible
df = pd.DataFrame({'a': [1, 2, 3], 'b': [4, 5, 6]})
conn.execute("CREATE TABLE data AS SELECT * FROM df")

# Bulk insert using Appender (fastest method)
import duckdb
conn = duckdb.connect()
conn.execute("CREATE TABLE fast_load (id INTEGER, value DOUBLE)")

appender = conn.appender("fast_load")
for i in range(1000000):
    appender.append_row([i, i * 1.5])
appender.close()
```

### R: Arrow Integration

```r
library(arrow)

# Read with Arrow
arrow_table <- read_parquet("data.parquet", as_data_frame = FALSE)

# Register with DuckDB
duckdb::duckdb_register_arrow(con, "arrow_data", arrow_table)

# Query
result <- dbGetQuery(con, "SELECT * FROM arrow_data WHERE value > 100")
```

---

## 11. Best Practices

### Connection Management

1. **Always close connections**: Ensure connections are properly closed to free resources.
   ```python
   # Use context managers in Python
   with duckdb.connect('db.duckdb') as conn:
       # Use connection
       pass
   # Automatically closed
   ```

2. **Reuse connections**: Don't create a new connection for every query.
   ```python
   # Good
   conn = duckdb.connect('db.duckdb')
   for query in queries:
       conn.execute(query)
   conn.close()

   # Bad
   for query in queries:
       conn = duckdb.connect('db.duckdb')
       conn.execute(query)
       conn.close()
   ```

3. **One connection per thread**: Connections are not thread-safe.
   ```python
   import threading

   def worker():
       # Create connection in thread
       conn = duckdb.connect('db.duckdb')
       conn.execute("SELECT * FROM data")
       conn.close()

   threads = [threading.Thread(target=worker) for _ in range(4)]
   ```

### Query Performance

1. **Use prepared statements**: For queries executed multiple times with different parameters.
   ```python
   # Good
   stmt = conn.execute("PREPARE insert_stmt AS INSERT INTO users VALUES ($1, $2, $3)")
   for row in data:
       conn.execute("EXECUTE insert_stmt(?, ?, ?)", row)

   # Better - use appender for bulk inserts
   appender = conn.appender("users")
   for row in data:
       appender.append_row(row)
   appender.close()
   ```

2. **Use bulk loading**: Appender API is much faster than individual inserts.
   ```python
   # Fastest method for bulk inserts
   appender = conn.appender("table_name")
   for row in large_dataset:
       appender.append_row(row)
   appender.close()
   ```

3. **Batch operations**: Group operations into transactions.
   ```python
   conn.execute("BEGIN TRANSACTION")
   for query in queries:
       conn.execute(query)
   conn.execute("COMMIT")
   ```

### Memory Management

1. **Configure memory limits**: Prevent out-of-memory errors.
   ```python
   conn = duckdb.connect(config={'memory_limit': '4GB'})
   ```

2. **Stream large results**: Don't materialize huge result sets.
   ```python
   # For very large results, use cursor
   cursor = conn.cursor()
   cursor.execute("SELECT * FROM huge_table")
   for batch in cursor.fetchmany(1000):
       process(batch)
   ```

3. **Clean up resources**: Destroy results and statements when done.
   ```c
   duckdb_destroy_result(&result);
   duckdb_destroy_prepare(&stmt);
   ```

### Error Handling

1. **Always check return values**: Especially in C/C++.
   ```c
   if (duckdb_query(con, query, &result) == DuckDBError) {
       fprintf(stderr, "Error: %s\n", duckdb_result_error(&result));
       duckdb_destroy_result(&result);
       return 1;
   }
   ```

2. **Use try-catch**: In languages that support exceptions.
   ```python
   try:
       conn.execute("SELECT * FROM non_existent_table")
   except duckdb.Error as e:
       print(f"Query failed: {e}")
   ```

3. **Transaction rollback**: Implement proper error recovery.
   ```java
   try {
       conn.setAutoCommit(false);
       // Execute queries
       conn.commit();
   } catch (SQLException e) {
       conn.rollback();
       throw e;
   }
   ```

### Data Type Handling

1. **Use appropriate types**: Match column types to data.
   ```python
   # Use appropriate DuckDB types
   conn.execute("""
       CREATE TABLE data (
           id INTEGER,
           amount DECIMAL(10,2),
           timestamp TIMESTAMP,
           metadata JSON
       )
   """)
   ```

2. **Handle NULL values**: Check for NULL before accessing.
   ```java
   int value = rs.getInt("column");
   if (!rs.wasNull()) {
       // Use value
   }
   ```

3. **Date/Time handling**: Use proper date/time types.
   ```python
   import datetime

   conn.execute(
       "INSERT INTO events VALUES (?, ?)",
       [datetime.datetime.now(), "event_name"]
   )
   ```

### Security

1. **Always use parameterized queries**: Never concatenate user input into SQL.
   ```python
   # Good - parameterized
   conn.execute("SELECT * FROM users WHERE name = ?", [user_input])

   # Bad - SQL injection risk
   conn.execute(f"SELECT * FROM users WHERE name = '{user_input}'")
   ```

2. **Validate input**: Check parameters before passing to queries.
   ```python
   def safe_query(conn, table_name):
       # Validate table name
       allowed_tables = ['users', 'orders', 'products']
       if table_name not in allowed_tables:
           raise ValueError("Invalid table name")

       return conn.execute(f"SELECT * FROM {table_name}").fetchall()
   ```

3. **Use read-only connections**: When appropriate.
   ```python
   conn = duckdb.connect('db.duckdb', read_only=True)
   ```

### Cross-Language Patterns

1. **Use standard formats**: Arrow for efficient data exchange between languages.
   ```python
   # Python: Export to Arrow
   arrow_table = conn.execute("SELECT * FROM data").arrow()

   # Save to IPC format
   import pyarrow as pa
   with pa.OSFile('data.arrow', 'wb') as f:
       writer = pa.ipc.new_file(f, arrow_table.schema)
       writer.write_table(arrow_table)
       writer.close()
   ```

2. **Export to standard formats**: CSV, Parquet for interoperability.
   ```python
   conn.execute("COPY data TO 'output.parquet' (FORMAT PARQUET)")
   ```

3. **Use consistent naming**: Keep table/column names compatible across platforms.

---

## Summary

DuckDB provides comprehensive client APIs across multiple programming languages:

- **Python**: Most feature-rich with SQL and Relation APIs, pandas/Arrow integration
- **C++**: High-performance object-oriented interface
- **C**: Low-level API used by many language bindings
- **Java/JDBC**: Standard JDBC interface for Java applications
- **Node.js**: Async/callback and promise-based APIs
- **R**: DBI-compatible interface with data frame integration

### Key Takeaways

1. Use **parameterized queries** to prevent SQL injection
2. Use **bulk loading** (Appender) for large data inserts
3. **Close connections** properly to free resources
4. **Reuse connections** within a thread
5. Configure **memory limits** for production environments
6. Use **Arrow integration** for efficient data exchange
7. Handle **errors properly** with try-catch or return value checks
8. Choose the **right API** for your use case (SQL vs Relation API in Python)

This guide covers the essential patterns for using DuckDB across different programming languages. For more details, refer to the official DuckDB documentation and the source code in `examples/` and `src/main/capi/`.
