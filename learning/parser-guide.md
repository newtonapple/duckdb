# DuckDB Parser Developer Guide

## Table of Contents
1. [Parser Architecture Overview](#1-parser-architecture-overview)
2. [libpg_query Integration](#2-libpg_query-integration)
3. [Parse Tree Structure (SQLStatement Hierarchy)](#3-parse-tree-structure-sqlstatement-hierarchy)
4. [Expression Types and Classes](#4-expression-types-and-classes)
5. [TableRef Types](#5-tableref-types)
6. [How to Add New SQL Syntax](#6-how-to-add-new-sql-syntax)
7. [Transformer Classes](#7-transformer-classes)
8. [Parser Testing](#8-parser-testing)
9. [Error Handling in Parser](#9-error-handling-in-parser)
10. [Common Parser Patterns](#10-common-parser-patterns)
11. [Extending the Grammar](#11-extending-the-grammar)
12. [Case Studies: How Existing Features Were Added](#12-case-studies-how-existing-features-were-added)

---

## 1. Parser Architecture Overview

The DuckDB parser is the entry point for SQL queries. It transforms SQL text into an Abstract Syntax Tree (AST) represented by `SQLStatement` objects.

### Key Components

The parser subsystem is located in `src/parser/` and consists of:

- **Parser** (`src/parser/parser.cpp:20-78`): Main entry point that coordinates parsing
- **Transformer** (`src/parser/transformer.cpp:12-18`): Converts libpg_query parse tree to DuckDB AST
- **PostgresParser** (`third_party/libpg_query/postgres_parser.cpp`): Underlying lexer/parser from Postgres
- **Expression classes** (`src/parser/expression/`): Various expression node types
- **Statement classes** (`src/parser/statement/`): Different SQL statement types
- **TableRef classes** (`src/parser/tableref/`): Table reference types

### Parser Flow

```
SQL String
    |
    v
Parser::ParseQuery() [src/parser/parser.cpp:193]
    |
    v
PostgresParser::Parse() [src/parser/parser.cpp:240]
    |
    v
libpg_query grammar (Bison/Flex)
    |
    v
PGNode tree (Postgres format)
    |
    v
Transformer::TransformParseTree() [src/parser/transformer.cpp:28]
    |
    v
SQLStatement objects (DuckDB format)
```

### Directory Structure

```
src/parser/
├── parser.cpp                    # Main parser entry point
├── transformer.cpp               # Transform coordinator
├── expression/                   # Expression implementations
│   ├── columnref_expression.cpp
│   ├── function_expression.cpp
│   ├── constant_expression.cpp
│   └── ...
├── statement/                    # Statement implementations
│   ├── select_statement.cpp
│   ├── insert_statement.cpp
│   └── ...
├── tableref/                     # Table reference implementations
│   ├── basetableref.cpp
│   ├── joinref.cpp
│   └── ...
├── transform/                    # Transformation logic
│   ├── expression/               # Expression transformers
│   ├── statement/                # Statement transformers
│   └── tableref/                 # TableRef transformers
└── parsed_data/                  # Auxiliary parsed data structures
```

---

## 2. libpg_query Integration

DuckDB uses a modified version of PostgreSQL's parser (`libpg_query`) located in `third_party/libpg_query/`.

### Key Modifications

From `third_party/libpg_query/README.md:5-9`:
- Output format changed to C++
- Parser wrapped in `duckdb_libpgquery` namespace
- Split into multiple files (not one big file)
- Reduced duplication and simplified code

### Grammar Structure

Grammar files are in `third_party/libpg_query/grammar/`:

```
grammar/
├── grammar.y                     # Main grammar file
├── statements/                   # Individual statement grammars
│   ├── select.y                  # SELECT statement (bulk of grammar)
│   ├── insert.y
│   ├── create_table.y
│   └── ...
├── statements.list               # List of statement files to include
├── keywords/                     # Keyword definitions
│   ├── unreserved_keywords.list
│   ├── reserved_keywords.list
│   ├── column_name_keywords.list
│   ├── func_name_keywords.list
│   └── type_name_keywords.list
└── types/                        # Return type definitions
    ├── select.yh
    └── ...
```

### Compiling the Grammar

From `third_party/libpg_query/README.md:19-30`:

```bash
# Compile grammar (requires bison)
python3 scripts/generate_grammar.py

# Compile lexer (requires flex)
python3 scripts/generate_flex.py
```

### Bison Grammar Basics

From `third_party/libpg_query/README.md:44-76`:

Example rule:
```yacc
from_list:
    table_ref                   { $$ = list_make1($1); }
    | from_list ',' table_ref   { $$ = lappend($1, $3); }
;
```

- `$$` = return value of the rule
- `$1, $2, $3` = sub-rule values
- Rules can be recursive (typically left-recursive for lists)
- Return types defined in corresponding `.yh` files

---

## 3. Parse Tree Structure (SQLStatement Hierarchy)

### SQLStatement Base Class

File: `src/include/duckdb/parser/sql_statement.hpp:19-66`

```cpp
class SQLStatement {
public:
    static constexpr const StatementType TYPE = StatementType::INVALID_STATEMENT;

    explicit SQLStatement(StatementType type) : type(type) {}
    virtual ~SQLStatement() {}

    StatementType type;              // Statement type enum
    idx_t stmt_location = 0;         // Location in query string
    idx_t stmt_length = 0;           // Statement length
    case_insensitive_map_t<idx_t> named_param_map;  // Named parameters
    string query;                    // Original query text

    virtual string ToString() const = 0;
    virtual unique_ptr<SQLStatement> Copy() const = 0;

    template <class TARGET>
    TARGET &Cast() { /* ... */ }
};
```

### Statement Types

File: `src/include/duckdb/common/enums/statement_type.hpp:20-52`

```cpp
enum class StatementType : uint8_t {
    INVALID_STATEMENT,
    SELECT_STATEMENT,       // SELECT queries
    INSERT_STATEMENT,       // INSERT
    UPDATE_STATEMENT,       // UPDATE
    CREATE_STATEMENT,       // CREATE TABLE/VIEW/INDEX/etc
    DELETE_STATEMENT,       // DELETE
    PREPARE_STATEMENT,      // PREPARE
    EXECUTE_STATEMENT,      // EXECUTE
    ALTER_STATEMENT,        // ALTER
    TRANSACTION_STATEMENT,  // BEGIN/COMMIT/ROLLBACK
    COPY_STATEMENT,         // COPY
    ANALYZE_STATEMENT,      // ANALYZE
    VARIABLE_SET_STATEMENT, // SET variable
    CREATE_FUNC_STATEMENT,  // CREATE FUNCTION
    EXPLAIN_STATEMENT,      // EXPLAIN
    DROP_STATEMENT,         // DROP
    EXPORT_STATEMENT,       // EXPORT
    PRAGMA_STATEMENT,       // PRAGMA
    VACUUM_STATEMENT,       // VACUUM
    CALL_STATEMENT,         // CALL
    SET_STATEMENT,          // SET
    LOAD_STATEMENT,         // LOAD
    RELATION_STATEMENT,
    EXTENSION_STATEMENT,
    LOGICAL_PLAN_STATEMENT,
    ATTACH_STATEMENT,       // ATTACH
    DETACH_STATEMENT,       // DETACH
    MULTI_STATEMENT,
    COPY_DATABASE_STATEMENT,
    UPDATE_EXTENSIONS_STATEMENT,
    MERGE_INTO_STATEMENT    // MERGE INTO
};
```

### Statement Implementations

All statement classes are in `src/parser/statement/`:

```
statement/
├── select_statement.cpp          # SELECT
├── insert_statement.cpp          # INSERT
├── update_statement.cpp          # UPDATE
├── delete_statement.cpp          # DELETE
├── create_statement.cpp          # CREATE
├── drop_statement.cpp            # DROP
├── alter_statement.cpp           # ALTER
├── transaction_statement.cpp     # BEGIN/COMMIT/ROLLBACK
├── copy_statement.cpp            # COPY
├── pragma_statement.cpp          # PRAGMA
├── explain_statement.cpp         # EXPLAIN
├── prepare_statement.cpp         # PREPARE
├── execute_statement.cpp         # EXECUTE
├── call_statement.cpp            # CALL
├── load_statement.cpp            # LOAD
├── attach_statement.cpp          # ATTACH
├── detach_statement.cpp          # DETACH
└── ...
```

### Example: SelectStatement

File: `src/include/duckdb/parser/statement/select_statement.hpp`

```cpp
class SelectStatement : public SQLStatement {
public:
    static constexpr const StatementType TYPE = StatementType::SELECT_STATEMENT;

    SelectStatement() : SQLStatement(StatementType::SELECT_STATEMENT) {}

    // The main query node (can be SelectNode, SetOperationNode, etc)
    unique_ptr<QueryNode> node;

    string ToString() const override;
    unique_ptr<SQLStatement> Copy() const override;
};
```

---

## 4. Expression Types and Classes

### Expression Architecture

Expressions inherit from `ParsedExpression` which inherits from `BaseExpression`.

File: `src/include/duckdb/parser/base_expression.hpp:20-152`

```cpp
class BaseExpression {
public:
    BaseExpression(ExpressionType type, ExpressionClass expression_class);
    virtual ~BaseExpression() {}

    ExpressionType type;              // What operation (COMPARE_EQUAL, etc)
    ExpressionClass expression_class; // What kind of node (FUNCTION, COLUMN_REF, etc)
    string alias;                     // Expression alias
    optional_idx query_location;      // Location in query

    virtual bool IsAggregate() const = 0;
    virtual bool IsWindow() const = 0;
    virtual bool HasSubquery() const = 0;
    virtual bool IsScalar() const = 0;
    virtual bool HasParameter() const = 0;
    virtual string ToString() const = 0;
    virtual hash_t Hash() const = 0;
};
```

### Expression Classes

All expression classes have `static constexpr const ExpressionClass TYPE`:

| Expression Class | Type | Description | File |
|-----------------|------|-------------|------|
| COLUMN_REF | Column reference | `a.b.c` or `column_name` | `columnref_expression.hpp:21` |
| CONSTANT | Constant value | `42`, `'string'`, `TRUE` | `constant_expression.hpp:19` |
| FUNCTION | Function call | `sum(x)`, `func(a, b)` | `function_expression.hpp:19` |
| OPERATOR | Operator | `a + b`, `NOT x` | `operator_expression.hpp:21` |
| COMPARISON | Comparison | `a = b`, `x < y` | `comparison_expression.hpp:18` |
| CONJUNCTION | AND/OR | `a AND b`, `x OR y` | `conjunction_expression.hpp:19` |
| CAST | Type cast | `CAST(x AS INT)` | `cast_expression.hpp:19` |
| CASE | CASE expression | `CASE WHEN ... END` | `case_expression.hpp:27` |
| BETWEEN | BETWEEN | `x BETWEEN a AND b` | `between_expression.hpp:17` |
| SUBQUERY | Subquery | `(SELECT ...)` | `subquery_expression.hpp:20` |
| WINDOW | Window function | `row_number() OVER (...)` | `window_expression.hpp:40` |
| STAR | Star expression | `*` or `table.*` | `star_expression.hpp:22` |
| PARAMETER | Parameter | `$1`, `?` | `parameter_expression.hpp:32` |
| LAMBDA | Lambda function | `x -> x + 1` | `lambda_expression.hpp:25` |
| LAMBDA_REF | Lambda parameter | Reference to lambda param | `lambdaref_expression.hpp:21` |
| COLLATE | Collation | `x COLLATE "en_US"` | `collate_expression.hpp:18` |
| DEFAULT | DEFAULT | `DEFAULT` keyword | `default_expression.hpp:17` |
| POSITIONAL_REFERENCE | Positional ref | `#1` | `positional_reference_expression.hpp:16` |
| BOUND_EXPRESSION | Bound expression | Used in binding phase | `bound_expression.hpp:22` |

### Expression Types

File: `src/include/duckdb/common/enums/expression_type.hpp:18-100`

```cpp
enum class ExpressionType : uint8_t {
    INVALID = 0,

    // Operators
    OPERATOR_CAST = 12,
    OPERATOR_NOT = 13,
    OPERATOR_IS_NULL = 14,
    OPERATOR_IS_NOT_NULL = 15,

    // Comparisons
    COMPARE_EQUAL = 25,
    COMPARE_NOTEQUAL = 26,
    COMPARE_LESSTHAN = 27,
    COMPARE_GREATERTHAN = 28,
    COMPARE_LESSTHANOREQUALTO = 29,
    COMPARE_GREATERTHANOREQUALTO = 30,
    COMPARE_IN = 35,
    COMPARE_NOT_IN = 36,
    COMPARE_DISTINCT_FROM = 37,
    COMPARE_BETWEEN = 38,
    COMPARE_NOT_BETWEEN = 39,
    COMPARE_NOT_DISTINCT_FROM = 40,

    // Conjunctions
    CONJUNCTION_AND = 50,
    CONJUNCTION_OR = 51,

    // Values
    VALUE_CONSTANT = 75,
    VALUE_PARAMETER = 76,
    VALUE_TUPLE = 77,
    VALUE_NULL = 79,
    VALUE_DEFAULT = 82,

    // Aggregates
    AGGREGATE = 100,
    BOUND_AGGREGATE = 101,
    GROUPING_FUNCTION = 102,

    // Window Functions
    WINDOW_AGGREGATE = 110,
    WINDOW_RANK = 120,
    WINDOW_RANK_DENSE = 121,
    WINDOW_ROW_NUMBER = 125,
    WINDOW_FIRST_VALUE = 130,
    // ... (more window functions)
};
```

### Example: ColumnRefExpression

File: `src/include/duckdb/parser/expression/columnref_expression.hpp:19-58`

```cpp
class ColumnRefExpression : public ParsedExpression {
public:
    static constexpr const ExpressionClass TYPE = ExpressionClass::COLUMN_REF;

    // Constructors
    ColumnRefExpression(string column_name, string table_name);
    explicit ColumnRefExpression(string column_name);
    explicit ColumnRefExpression(vector<string> column_names);

    // The stack of names: column_names[0].column_names[1].column_names[2]...
    vector<string> column_names;

    bool IsQualified() const;
    const string &GetColumnName() const;
    const string &GetTableName() const;

    string ToString() const override;
    unique_ptr<ParsedExpression> Copy() const override;
    hash_t Hash() const override;
};
```

### Example: FunctionExpression

File: `src/include/duckdb/parser/expression/function_expression.hpp:17-134`

```cpp
class FunctionExpression : public ParsedExpression {
public:
    static constexpr const ExpressionClass TYPE = ExpressionClass::FUNCTION;

    FunctionExpression(string catalog_name, string schema_name,
                      const string &function_name,
                      vector<unique_ptr<ParsedExpression>> children,
                      unique_ptr<ParsedExpression> filter = nullptr,
                      unique_ptr<OrderModifier> order_bys = nullptr,
                      bool distinct = false,
                      bool is_operator = false,
                      bool export_state = false);

    string catalog;                              // Catalog name
    string schema;                               // Schema name
    string function_name;                        // Function name
    bool is_operator;                            // Is this an operator?
    vector<unique_ptr<ParsedExpression>> children;  // Arguments
    bool distinct;                               // DISTINCT modifier
    unique_ptr<ParsedExpression> filter;         // FILTER clause
    unique_ptr<OrderModifier> order_bys;         // ORDER BY clause
    bool export_state;                           // EXPORT_STATE modifier

    string ToString() const override;
    unique_ptr<ParsedExpression> Copy() const override;
};
```

---

## 5. TableRef Types

### TableRef Base Class

File: `src/include/duckdb/parser/tableref.hpp:20-79`

```cpp
class TableRef {
public:
    static constexpr const TableReferenceType TYPE = TableReferenceType::INVALID;

    explicit TableRef(TableReferenceType type) : type(type) {}
    virtual ~TableRef() {}

    TableReferenceType type;
    string alias;                                // Table alias
    unique_ptr<SampleOptions> sample;            // TABLESAMPLE options
    optional_idx query_location;                 // Location in query
    shared_ptr<ExternalDependency> external_dependency;
    vector<string> column_name_alias;            // Column aliases

    virtual string ToString() const = 0;
    virtual unique_ptr<TableRef> Copy() = 0;

    template <class TARGET>
    TARGET &Cast() { /* ... */ }
};
```

### TableRef Types

File: `src/include/duckdb/common/enums/tableref_type.hpp:18-32`

```cpp
enum class TableReferenceType : uint8_t {
    INVALID = 0,
    BASE_TABLE = 1,      // Regular table: FROM table_name
    SUBQUERY = 2,        // Subquery: FROM (SELECT ...)
    JOIN = 3,            // Join: FROM a JOIN b
    TABLE_FUNCTION = 5,  // Table function: FROM read_csv(...)
    EXPRESSION_LIST = 6, // VALUES: FROM (VALUES (1,2), (3,4))
    CTE = 7,             // CTE reference: FROM cte_name
    EMPTY_FROM = 8,      // Empty FROM clause
    PIVOT = 9,           // PIVOT: FROM x PIVOT ...
    SHOW_REF = 10,       // SHOW statement result
    COLUMN_DATA = 11,    // Column data collection
    DELIM_GET = 12,      // Delimited get ref
    BOUND_TABLE_REF = 13 // Bound table ref (binding phase)
};
```

### TableRef Implementations

All in `src/parser/tableref/`:

| Type | File | Description |
|------|------|-------------|
| BASE_TABLE | `basetableref.cpp` | Regular table reference |
| SUBQUERY | `subqueryref.cpp` | `FROM (SELECT ...)` |
| JOIN | `joinref.cpp` | `FROM a JOIN b ON ...` |
| TABLE_FUNCTION | `table_function.cpp` | `FROM func(args)` |
| EXPRESSION_LIST | `expressionlistref.cpp` | `VALUES (...)` |
| EMPTY_FROM | `emptytableref.cpp` | Queries with no FROM |
| PIVOT | `pivotref.cpp` | `FROM x PIVOT (...)` |
| SHOW_REF | `showref.cpp` | SHOW command results |

### Example: JoinRef

File: `src/include/duckdb/parser/tableref/joinref.hpp`

```cpp
class JoinRef : public TableRef {
public:
    static constexpr const TableReferenceType TYPE = TableReferenceType::JOIN;

    JoinRef() : TableRef(TableReferenceType::JOIN) {}

    unique_ptr<TableRef> left;           // Left table
    unique_ptr<TableRef> right;          // Right table
    unique_ptr<ParsedExpression> condition;  // ON condition
    JoinType type;                       // INNER, LEFT, RIGHT, FULL, etc
    JoinRefType ref_type;                // REGULAR, NATURAL, CROSS, etc
    vector<string> using_columns;        // USING columns

    string ToString() const override;
    unique_ptr<TableRef> Copy() override;
};
```

---

## 6. How to Add New SQL Syntax

### Step-by-Step Process

#### 1. Add Keywords (if needed)

File: `third_party/libpg_query/grammar/keywords/`

From `third_party/libpg_query/README.md:101-126`:

**Prefer unreserved keywords** - add to `unreserved_keywords.list`:
```
MY_NEW_KEYWORD
```

Reserved keywords (use sparingly) - add to `reserved_keywords.list`:
```
RESERVED_WORD
```

Partial reservations:
- `column_name_keywords.list` - can be used as column name
- `func_name_keywords.list` - can be used as function name
- `type_name_keywords.list` - can be used as type name

#### 2. Define Grammar Rules

Add grammar rules to appropriate file in `third_party/libpg_query/grammar/statements/`.

For SELECT-related syntax, edit `third_party/libpg_query/grammar/statements/select.y`.

Example adding a new clause:
```yacc
my_new_clause:
    MY_NEW_KEYWORD expr_list
    {
        PGMyNewClause *n = makeNode(PGMyNewClause);
        n->expressions = $2;
        $$ = (PGNode *) n;
    }
;
```

#### 3. Define Return Types

Add return type to corresponding `.yh` file:

File: `third_party/libpg_query/grammar/types/select.yh`
```yacc
%type <node> my_new_clause
```

#### 4. Define Parse Node Structure

File: `third_party/libpg_query/include/nodes/parsenodes.hpp`

```cpp
typedef struct PGMyNewClause {
    PGNodeTag type;
    PGList *expressions;
    int location;
} PGMyNewClause;
```

#### 5. Create DuckDB AST Node

Create header file (if needed):
```cpp
// src/include/duckdb/parser/parsed_data/my_new_info.hpp
namespace duckdb {

struct MyNewInfo {
    vector<unique_ptr<ParsedExpression>> expressions;

    unique_ptr<MyNewInfo> Copy() const;
    void Serialize(Serializer &serializer) const;
    static unique_ptr<MyNewInfo> Deserialize(Deserializer &deserializer);
};

} // namespace duckdb
```

#### 6. Add Transformer Method

File: `src/include/duckdb/parser/transformer.hpp`

Add declaration:
```cpp
class Transformer {
    // ...
    unique_ptr<MyNewInfo> TransformMyNewClause(duckdb_libpgquery::PGMyNewClause &clause);
};
```

File: `src/parser/transform/helpers/transform_my_new.cpp`

Implement transformation:
```cpp
#include "duckdb/parser/transformer.hpp"

namespace duckdb {

unique_ptr<MyNewInfo> Transformer::TransformMyNewClause(
    duckdb_libpgquery::PGMyNewClause &clause) {

    auto result = make_uniq<MyNewInfo>();

    // Transform the expression list
    if (clause.expressions) {
        TransformExpressionList(*clause.expressions, result->expressions);
    }

    return result;
}

} // namespace duckdb
```

#### 7. Integrate into Statement Transformer

File: `src/parser/transform/statement/transform_select.cpp` (or appropriate statement file)

```cpp
// In the appropriate transform function, add:
if (select.my_new_clause) {
    result->my_new_info = TransformMyNewClause(
        PGCast<duckdb_libpgquery::PGMyNewClause>(*select.my_new_clause)
    );
}
```

#### 8. Recompile Grammar

```bash
cd third_party/libpg_query
python3 scripts/generate_grammar.py
python3 scripts/generate_flex.py  # if lexer changed
```

#### 9. Add Tests

Create test file: `test/sql/parser/my_new_feature.test`

```
# name: test/sql/parser/my_new_feature.test
# description: Test MY_NEW_KEYWORD clause
# group: [parser]

statement ok
SELECT * FROM table MY_NEW_KEYWORD (expr1, expr2)

statement error
SELECT * FROM table MY_NEW_KEYWORD
----
Syntax error
```

---

## 7. Transformer Classes

### Transformer Architecture

File: `src/parser/transformer.cpp:12-18`

```cpp
Transformer::Transformer(ParserOptions &options)
    : parent(nullptr), options(options), stack_depth(DConstants::INVALID_INDEX) {
}

Transformer::Transformer(Transformer &parent)
    : parent(&parent), options(parent.options), stack_depth(DConstants::INVALID_INDEX) {
}
```

The `Transformer` class converts libpg_query parse nodes into DuckDB AST nodes.

### Main Transform Entry Point

File: `src/parser/transformer.cpp:28-41`

```cpp
bool Transformer::TransformParseTree(duckdb_libpgquery::PGList *tree,
                                     vector<unique_ptr<SQLStatement>> &statements) {
    InitializeStackCheck();
    for (auto entry = tree->head; entry != nullptr; entry = entry->next) {
        Clear();
        auto n = PGPointerCast<duckdb_libpgquery::PGNode>(entry->data.ptr_value);
        auto stmt = TransformStatement(*n);
        D_ASSERT(stmt);
        if (HasPivotEntries()) {
            stmt = CreatePivotStatement(std::move(stmt));
        }
        statements.push_back(std::move(stmt));
    }
    return true;
}
```

### Statement Transformation Dispatch

File: `src/parser/transformer.cpp:135-233`

```cpp
unique_ptr<SQLStatement> Transformer::TransformStatementInternal(
    duckdb_libpgquery::PGNode &stmt) {

    switch (stmt.type) {
    case duckdb_libpgquery::T_PGSelectStmt:
        return TransformSelectStmt(PGCast<duckdb_libpgquery::PGSelectStmt>(stmt));
    case duckdb_libpgquery::T_PGCreateStmt:
        return TransformCreateTable(PGCast<duckdb_libpgquery::PGCreateStmt>(stmt));
    case duckdb_libpgquery::T_PGInsertStmt:
        return TransformInsert(PGCast<duckdb_libpgquery::PGInsertStmt>(stmt));
    case duckdb_libpgquery::T_PGDeleteStmt:
        return TransformDelete(PGCast<duckdb_libpgquery::PGDeleteStmt>(stmt));
    case duckdb_libpgquery::T_PGUpdateStmt:
        return TransformUpdate(PGCast<duckdb_libpgquery::PGUpdateStmt>(stmt));
    // ... more cases
    default:
        throw NotImplementedException(NodetypeToString(stmt.type));
    }
}
```

### Expression Transformation Dispatch

File: `src/parser/transform/expression/transform_expression.cpp:27-85`

```cpp
unique_ptr<ParsedExpression> Transformer::TransformExpression(
    duckdb_libpgquery::PGNode &node) {

    auto stack_checker = StackCheck();

    switch (node.type) {
    case duckdb_libpgquery::T_PGColumnRef:
        return TransformColumnRef(PGCast<duckdb_libpgquery::PGColumnRef>(node));
    case duckdb_libpgquery::T_PGAConst:
        return TransformConstant(PGCast<duckdb_libpgquery::PGAConst>(node));
    case duckdb_libpgquery::T_PGAExpr:
        return TransformAExpr(PGCast<duckdb_libpgquery::PGAExpr>(node));
    case duckdb_libpgquery::T_PGFuncCall:
        return TransformFuncCall(PGCast<duckdb_libpgquery::PGFuncCall>(node));
    case duckdb_libpgquery::T_PGBoolExpr:
        return TransformBoolExpr(PGCast<duckdb_libpgquery::PGBoolExpr>(node));
    case duckdb_libpgquery::T_PGTypeCast:
        return TransformTypeCast(PGCast<duckdb_libpgquery::PGTypeCast>(node));
    case duckdb_libpgquery::T_PGCaseExpr:
        return TransformCase(PGCast<duckdb_libpgquery::PGCaseExpr>(node));
    case duckdb_libpgquery::T_PGSubLink:
        return TransformSubquery(PGCast<duckdb_libpgquery::PGSubLink>(node));
    // ... more cases
    default:
        throw NotImplementedException("Expression type %s (%d)",
                                     NodetypeToString(node.type), (int)node.type);
    }
}
```

### Transform Helpers

File: `src/include/duckdb/parser/transformer.hpp:311-382`

Key helper methods:
- `TransformExpressionList()` - Transform list of expressions
- `TransformOrderBy()` - Transform ORDER BY clause
- `TransformGroupBy()` - Transform GROUP BY clause
- `TransformFrom()` - Transform FROM clause
- `TransformStringList()` - Convert PGList to vector<string>
- `TransformAlias()` - Transform alias
- `TransformTypeName()` - Transform type name
- `SetQueryLocation()` - Set query location for error messages

### Stack Depth Checking

File: `src/parser/transformer.cpp:47-56`

Prevents stack overflow on deeply nested queries:

```cpp
StackChecker<Transformer> Transformer::StackCheck(idx_t extra_stack) {
    auto &root = RootTransformer();
    D_ASSERT(root.stack_depth != DConstants::INVALID_INDEX);
    if (root.stack_depth + extra_stack >= options.max_expression_depth) {
        throw ParserException("Max expression depth limit of %lld exceeded. "
                            "Use \"SET max_expression_depth TO x\" to increase.",
                            options.max_expression_depth);
    }
    return StackChecker<Transformer>(root, extra_stack);
}
```

---

## 8. Parser Testing

### Test Framework

DuckDB uses **sqllogictest** (`.test` files) for parser testing. Strongly prefer this over C++ tests.

From `CLAUDE.md:23-35`:
- **Fast tests**: `make unit` or `make unittest` (~1 minute)
- **All tests**: `make allunit` (~1 hour)
- **Direct execution**: `build/debug/test/unittest`
- **Tagged tests**: `build/debug/test/unittest "[tag]"`
- **Specific test**: `build/debug/test/unittest "TestName"`

### Test File Structure

Example: `test/sql/parser/star_expression.test:1-50`

```
# name: test/sql/parser/star_expression.test
# description: Test star expression in different places
# group: [parser]

statement ok
PRAGMA enable_verification

statement ok
CREATE TABLE integers AS SELECT 42 i, 84 j UNION ALL SELECT 13, 14

# Should error - star not allowed in WHERE
statement error
SELECT * FROM integers WHERE *
----
Use COLUMNS(*) instead

# Should work - COLUMNS(*) allowed
query II
SELECT * FROM integers WHERE COLUMNS(*) IS NULL ORDER BY ALL
----

# Should error - not supported
statement error
SELECT * FROM integers GROUP BY COLUMNS(*)
----
not supported
```

### Test Directives

- `statement ok` - Statement should succeed
- `statement error` - Statement should fail
- `query <types>` - Query should succeed with result
  - Types: `I` = integer, `T` = text, `R` = real, etc.
- `----` - Separator between query and expected result/error
- `# comment` - Comments

### Parser-Specific Test Locations

```
test/sql/parser/
├── star_expression.test
├── columns_aliases.test
├── trailing_commas.test
├── join_alias.test
├── division_operator_precedence.test
├── from_first.test
├── indirection.test
└── ...
```

### Writing Good Parser Tests

1. **Test positive cases** - Valid syntax should parse
2. **Test negative cases** - Invalid syntax should error with good message
3. **Test edge cases** - Boundary conditions, empty inputs
4. **Test combinations** - Features combined together
5. **Test error locations** - Errors point to right place

Example comprehensive test:
```
# Positive case
statement ok
SELECT x FROM t WHERE x > 10

# Negative case - syntax error
statement error
SELECT x FROM t WHERE > 10
----
syntax error

# Edge case - empty list
statement ok
SELECT x FROM t WHERE x IN ()

# Combination - multiple features
statement ok
SELECT x, COUNT(*) FROM t WHERE x > 10 GROUP BY x HAVING COUNT(*) > 5
```

---

## 9. Error Handling in Parser

### Parser Exception Types

File: `src/parser/parser.cpp:265`

```cpp
throw ParserException::SyntaxError(query, parser_error, parser_error_location);
```

### QueryErrorContext

File: `src/include/duckdb/parser/query_error_context.hpp:19-31`

```cpp
class QueryErrorContext {
public:
    QueryErrorContext(const ParsedExpression &expr);
    explicit QueryErrorContext(optional_idx query_location_p = optional_idx())
        : query_location(query_location_p) {}

    optional_idx query_location;  // Location in query where error occurred

    static string Format(const string &query,
                        const string &error_message,
                        optional_idx error_loc,
                        bool add_line_indicator = true);
};
```

### Setting Query Locations

File: `src/parser/transformer.cpp:235-247`

```cpp
void Transformer::SetQueryLocation(ParsedExpression &expr, int query_location) {
    if (query_location < 0) {
        return;
    }
    expr.SetQueryLocation(optional_idx(static_cast<idx_t>(query_location)));
}

void Transformer::SetQueryLocation(TableRef &ref, int query_location) {
    if (query_location < 0) {
        return;
    }
    ref.query_location = optional_idx(static_cast<idx_t>(query_location));
}
```

### Error Message Guidelines

From parser code:

1. **Be specific** - "Column 'x' not found" not "Error"
2. **Suggest fixes** - "Use COLUMNS(*) instead"
3. **Include location** - Point to exact position in query
4. **Be user-friendly** - Explain what went wrong

Example from `test/sql/parser/star_expression.test:12-14`:
```
statement error
SELECT * FROM integers WHERE *
----
Use COLUMNS(*) instead
```

### Unicode Space Handling

File: `src/parser/parser.cpp:59-166`

Parser strips unicode spaces and replaces with regular spaces to avoid parsing issues:

```cpp
bool Parser::StripUnicodeSpaces(const string &query_str, string &new_query) {
    // Handles various unicode space characters:
    // U+00A0 (non-breaking space)
    // U+2000 to U+200B (various spaces)
    // U+3000 (ideographic space)
    // etc.
    // ...
}
```

---

## 10. Common Parser Patterns

### Pattern 1: List Construction

Left-recursive rules build lists:

```yacc
expr_list:
    a_expr                      { $$ = list_make1($1); }
    | expr_list ',' a_expr      { $$ = lappend($1, $3); }
;
```

### Pattern 2: Optional Clauses

Use `opt_` prefix for optional elements:

```yacc
opt_where_clause:
    WHERE a_expr                { $$ = $2; }
    | /* EMPTY */               { $$ = NULL; }
;
```

### Pattern 3: Nested Subqueries

Handle parentheses correctly:

```yacc
select_with_parens:
    '(' select_no_parens ')'    { $$ = $2; }
    | '(' select_with_parens ')' { $$ = $2; }
;
```

### Pattern 4: Expression Transformation

File: `src/parser/transform/expression/transform_expression.cpp:94-102`

```cpp
void Transformer::TransformExpressionList(duckdb_libpgquery::PGList &list,
                                         vector<unique_ptr<ParsedExpression>> &result) {
    for (auto node = list.head; node != nullptr; node = node->next) {
        auto target = PGPointerCast<duckdb_libpgquery::PGNode>(node->data.ptr_value);
        auto expr = TransformExpression(*target);
        result.push_back(std::move(expr));
    }
}
```

### Pattern 5: Statement Location Tracking

File: `src/parser/parser.cpp:326-338`

```cpp
if (!statements.empty()) {
    auto &last_statement = statements.back();
    last_statement->stmt_length = query.size() - last_statement->stmt_location;
    for (auto &statement : statements) {
        statement->query = query.substr(statement->stmt_location,
                                       statement->stmt_length);
        statement->stmt_location = 0;
        statement->stmt_length = statement->query.size();
    }
}
```

### Pattern 6: Parser Extension Points

File: `src/parser/parser.cpp:208-232`

Extensions can override parser:

```cpp
if (options.extensions) {
    for (auto &ext : *options.extensions) {
        if (!ext.parser_override) {
            continue;
        }
        auto result = ext.parser_override(ext.parser_info.get(), query);
        if (result.type == ParserExtensionResultType::PARSE_SUCCESSFUL) {
            statements = std::move(result.statements);
            return;
        }
    }
}
```

### Pattern 7: Cast Helper Templates

File: `src/include/duckdb/parser/transformer.hpp:396-403`

```cpp
template <class T>
static T &PGCast(duckdb_libpgquery::PGNode &node) {
    return reinterpret_cast<T &>(node);
}

template <class T>
static optional_ptr<T> PGPointerCast(void *ptr) {
    return optional_ptr<T>(reinterpret_cast<T *>(ptr));
}
```

---

## 11. Extending the Grammar

### Adding a New Statement Type

From `third_party/libpg_query/README.md:127-130`:

1. Create `new_statement.y` in `grammar/statements/`
2. Create `new_statement.yh` in `grammar/types/`
3. Add to `grammar/statements.list`

### Grammar File Structure

Example: `third_party/libpg_query/grammar/statements/pragma.y`

```yacc
/*****************************************************************************
 *
 *    PRAGMA Statements
 *
 *****************************************************************************/

PragmaStmt:
    PRAGMA_P ColId
    {
        PGPragmaStmt *n = makeNode(PGPragmaStmt);
        n->name = $2;
        n->args = NULL;
        n->kind = PG_PRAGMA_TYPE_NOTHING;
        $$ = (PGNode *) n;
    }
    | PRAGMA_P ColId '=' var_value
    {
        PGPragmaStmt *n = makeNode(PGPragmaStmt);
        n->name = $2;
        n->args = NULL;
        n->kind = PG_PRAGMA_TYPE_ASSIGNMENT;
        n->parse_value = $4;
        $$ = (PGNode *) n;
    }
    | PRAGMA_P ColId '(' var_list ')'
    {
        PGPragmaStmt *n = makeNode(PGPragmaStmt);
        n->name = $2;
        n->args = $4;
        n->kind = PG_PRAGMA_TYPE_CALL;
        $$ = (PGNode *) n;
    }
;
```

### Keyword Placement Strategy

From `third_party/libpg_query/README.md:114-126`:

**Preference order:**
1. `unreserved_keywords.list` - Best choice, least restrictive
2. Combination of `column_name_keywords.list`, `func_name_keywords.list`, `type_name_keywords.list`
3. `reserved_keywords.list` - Last resort, most restrictive

**Why unreserved is better:**
- Users can still use the keyword as identifiers
- Doesn't break existing code
- More SQL standard compliant

### Modifying Select Grammar

Most expression/query features go in `select.y`:

```yacc
// Example: Add new expression type
a_expr:
    // ... existing rules ...
    | MY_NEW_EXPR '(' a_expr ')'
    {
        PGFuncCall *n = makeNode(PGFuncCall);
        n->funcname = SystemFuncName("my_new_expr");
        n->args = list_make1($3);
        n->location = @1;
        $$ = (PGNode *) n;
    }
;
```

### Testing Grammar Changes

After modifying grammar:

1. Regenerate grammar: `python3 scripts/generate_grammar.py`
2. Build DuckDB: `make debug`
3. Run parser tests: `make unittest`
4. Fix any issues
5. Add new tests for your feature

---

## 12. Case Studies: How Existing Features Were Added

### Case Study 1: LAMBDA Functions

Lambda functions allow inline function definitions: `x -> x + 1`

**Grammar Addition:**

Keywords added to `unreserved_keywords.list`:
```
LAMBDA
```

Grammar rule in `select.y`:
```yacc
LambdaFunction:
    LAMBDA param_list ':' a_expr
    {
        PGLambdaFunction *n = makeNode(PGLambdaFunction);
        n->parameters = $2;
        n->function = $4;
        n->location = @1;
        $$ = (PGNode *) n;
    }
;
```

**Transform Implementation:**

File: `src/parser/transform/expression/transform_lambda.cpp`

Transform creates `LambdaExpression`:
```cpp
unique_ptr<ParsedExpression> Transformer::TransformLambda(
    duckdb_libpgquery::PGLambdaFunction &node) {

    auto result = make_uniq<LambdaExpression>();

    // Transform parameters
    for (auto c = node.parameters->head; c != nullptr; c = lnext(c)) {
        auto param = PGPointerCast<duckdb_libpgquery::PGNode>(c->data.ptr_value);
        // ... transform parameter ...
        result->parameters.push_back(/* ... */);
    }

    // Transform body
    result->expression = TransformExpression(*node.function);

    return std::move(result);
}
```

**Expression Class:**

File: `src/include/duckdb/parser/expression/lambda_expression.hpp:25`
```cpp
class LambdaExpression : public ParsedExpression {
public:
    static constexpr const ExpressionClass TYPE = ExpressionClass::LAMBDA;

    vector<string> parameters;
    unique_ptr<ParsedExpression> expression;

    // ... methods ...
};
```

### Case Study 2: PIVOT Clause

PIVOT transforms rows into columns.

**Grammar Addition:**

Keywords to `unreserved_keywords.list`:
```
PIVOT
UNPIVOT
```

Grammar rule in `select.y`:
```yacc
PivotExpr:
    table_ref PIVOT '(' pivot_column_list FOR pivot_value_list ')'
    {
        PGPivotExpr *n = makeNode(PGPivotExpr);
        n->source = $1;
        n->aggrs = $4;
        n->unpivot_columns = $6;
        n->pivot = true;
        $$ = (PGNode *) n;
    }
;
```

**Transform Implementation:**

File: `src/parser/transform/tableref/transform_pivot.cpp`

Complex transformation with multiple stages:
1. Transform source table
2. Transform pivot columns
3. Transform aggregate functions
4. Create pivot metadata

### Case Study 3: POSITIONAL References (#1, #2)

Allows referencing columns by position.

**Lexer Addition:**

File: `third_party/libpg_query/scan.l`

```c
{decdigit}+     { /* existing number rule */ }
#{decdigit}+    {
                    yylval->ival = atoi(yytext + 1);
                    return POSITIONAL_REFERENCE;
                }
```

**Grammar Rule:**

```yacc
c_expr:
    // ... existing rules ...
    | POSITIONAL_REFERENCE
    {
        PGPositionalReference *n = makeNode(PGPositionalReference);
        n->position = $1;
        n->location = @1;
        $$ = (PGNode *) n;
    }
;
```

**Transform:**

File: `src/parser/transform/expression/transform_positional_reference.cpp`

```cpp
unique_ptr<ParsedExpression> Transformer::TransformPositionalReference(
    duckdb_libpgquery::PGPositionalReference &node) {

    auto result = make_uniq<PositionalReferenceExpression>(node.position);
    SetQueryLocation(*result, node.location);
    return std::move(result);
}
```

### Case Study 4: COLUMNS(*) Expression

Allows selecting all columns programmatically.

**Grammar Rule:**

```yacc
c_expr:
    // ... existing rules ...
    | COLUMNS '(' a_expr ')'
    {
        PGFuncCall *n = makeNode(PGFuncCall);
        n->funcname = SystemFuncName("columns");
        n->args = list_make1($3);
        n->location = @1;
        $$ = (PGNode *) n;
    }
;
```

**Transform:**

Transformed as regular function call, special handling in binder.

**Error Handling:**

From `test/sql/parser/star_expression.test:12-14`:
```
statement error
SELECT * FROM integers WHERE *
----
Use COLUMNS(*) instead
```

Good error message guides users to correct syntax.

### Case Study 5: MERGE INTO Statement

Complex multi-clause statement for conditional insert/update.

**Grammar File:**

New file: `third_party/libpg_query/grammar/statements/merge_into.y`

```yacc
MergeIntoStmt:
    MERGE INTO qualified_name opt_as_alias
    USING table_ref
    ON a_expr
    match_clauses
    {
        PGMergeIntoStmt *n = makeNode(PGMergeIntoStmt);
        n->relation = $3;
        n->alias = $4;
        n->source = $6;
        n->on_condition = $8;
        n->when_clauses = $9;
        $$ = (PGNode *) n;
    }
;

match_clauses:
    match_clause                        { $$ = list_make1($1); }
    | match_clauses match_clause        { $$ = lappend($1, $2); }
;

match_clause:
    WHEN MATCHED opt_match_condition THEN match_action
    {
        PGMatchAction *n = makeNode(PGMatchAction);
        n->matched = true;
        n->condition = $3;
        n->action = $5;
        $$ = (PGNode *) n;
    }
    | WHEN NOT MATCHED opt_match_condition THEN INSERT opt_column_list VALUES '(' expr_list ')'
    {
        PGMatchAction *n = makeNode(PGMatchAction);
        n->matched = false;
        n->condition = $4;
        n->insert_columns = $7;
        n->insert_values = $10;
        $$ = (PGNode *) n;
    }
;
```

**Transform:**

File: `src/parser/transform/statement/transform_merge_into.cpp`

```cpp
unique_ptr<SQLStatement> Transformer::TransformMergeInto(
    duckdb_libpgquery::PGMergeIntoStmt &stmt) {

    auto result = make_uniq<MergeIntoStatement>();

    // Transform target table
    result->table = TransformRangeVar(*stmt.relation);
    if (stmt.alias) {
        result->table->alias = stmt.alias->aliasname;
    }

    // Transform source
    result->source = TransformTableRefNode(*stmt.source);

    // Transform ON condition
    result->on_condition = TransformExpression(*stmt.on_condition);

    // Transform WHEN clauses
    for (auto node = stmt.when_clauses->head; node; node = node->next) {
        auto action_node = PGPointerCast<duckdb_libpgquery::PGMatchAction>(
            node->data.ptr_value);
        result->actions.push_back(TransformMergeIntoAction(*action_node));
    }

    return std::move(result);
}
```

### Common Themes

1. **Start with grammar** - Define syntax first
2. **Use existing patterns** - Follow established conventions
3. **Transform incrementally** - Build up complex structures
4. **Test thoroughly** - Both positive and negative cases
5. **Error messages matter** - Guide users to correct syntax
6. **Documentation** - Update docs and add examples

---

## Quick Reference

### File Locations

| Component | Location |
|-----------|----------|
| Main parser | `src/parser/parser.cpp` |
| Transformer | `src/parser/transformer.cpp` |
| Expressions | `src/include/duckdb/parser/expression/*.hpp` |
| Statements | `src/include/duckdb/parser/statement/*.hpp` |
| TableRefs | `src/include/duckdb/parser/tableref/*.hpp` |
| Transform logic | `src/parser/transform/` |
| Grammar files | `third_party/libpg_query/grammar/` |
| Parser tests | `test/sql/parser/` |

### Key Classes

| Class | Purpose | File |
|-------|---------|------|
| `Parser` | Main parser entry | `parser.hpp:30` |
| `Transformer` | PG to DuckDB AST | `transformer.hpp:46` |
| `SQLStatement` | Base statement | `sql_statement.hpp:20` |
| `ParsedExpression` | Base expression | `parsed_expression.hpp:30` |
| `TableRef` | Table reference | `tableref.hpp:20` |
| `QueryErrorContext` | Error context | `query_error_context.hpp:19` |

### Build Commands

```bash
# Rebuild grammar
cd third_party/libpg_query
python3 scripts/generate_grammar.py

# Rebuild lexer
python3 scripts/generate_flex.py

# Build DuckDB
cd ../..
make debug

# Run parser tests
make unittest
```

### Useful Patterns

```cpp
// Cast PG node
auto &select = PGCast<duckdb_libpgquery::PGSelectStmt>(node);

// Transform expression list
TransformExpressionList(list, result);

// Create statement
auto result = make_uniq<SelectStatement>();

// Set query location
SetQueryLocation(*expr, location);

// Check optional node
if (node.where_clause) {
    result->where = TransformExpression(*node.where_clause);
}
```

---

## Conclusion

The DuckDB parser is a sophisticated system that transforms SQL text into a structured AST. By understanding:
- The flow from SQL to AST via libpg_query
- The statement/expression/tableref class hierarchy
- The transformer pattern for converting PG nodes to DuckDB nodes
- The grammar extension mechanism
- Proper testing and error handling

You can effectively extend and maintain the parser subsystem. Remember to:
- Prefer unreserved keywords
- Follow existing patterns
- Test thoroughly
- Provide good error messages
- Document your changes

For questions, refer to the codebase examples and follow the patterns established by existing features.
