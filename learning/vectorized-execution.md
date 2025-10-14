# DuckDB Vectorized Execution Engine

This document provides a deep technical dive into DuckDB's vectorized execution engine, intended for developers working on or extending DuckDB's core execution system.

## Table of Contents
1. [What is Vectorized Execution](#what-is-vectorized-execution)
2. [Vector Class Architecture](#vector-class-architecture)
3. [DataChunk Class](#datachunk-class)
4. [Vector Types](#vector-types)
5. [Selection Vectors](#selection-vectors)
6. [NULL Handling](#null-handling)
7. [How Operators Process Vectors](#how-operators-process-vectors)
8. [UnifiedVectorFormat](#unifiedvectorformat)
9. [Vector Operations and Primitives](#vector-operations-and-primitives)
10. [Performance Implications](#performance-implications)
11. [Writing Vectorized Code](#writing-vectorized-code)
12. [Common Patterns](#common-patterns)
13. [Debugging Vectorized Operations](#debugging-vectorized-operations)

## What is Vectorized Execution

Vectorized execution is a query processing model where operations are performed on batches (vectors) of data rather than individual tuples. This approach provides several critical advantages:

### Key Benefits

1. **Cache Efficiency**: Processing data in batches improves CPU cache utilization by reducing instruction cache misses through tight loops.

2. **SIMD Exploitation**: Modern CPUs can process multiple data elements in a single instruction (Single Instruction, Multiple Data). Vectorized code is amenable to auto-vectorization by compilers.

3. **Reduced Interpretation Overhead**: Instead of interpreting operations tuple-by-tuple (as in iterator models like Volcano), vectorized execution amortizes the cost of function calls and branching over many tuples.

4. **Better Compiler Optimization**: Tight loops over arrays enable aggressive compiler optimizations like loop unrolling, inlining, and prefetching.

### Vector Size

DuckDB uses a standard vector size defined in `/src/include/duckdb/common/vector_size.hpp`:

```cpp
#define DEFAULT_STANDARD_VECTOR_SIZE 2048U
```

This means operations typically process 2048 tuples at a time. This size is a careful balance:
- Large enough to amortize function call overhead
- Small enough to fit in CPU cache (L1/L2)
- Power of 2 for efficient bitmasking operations

## Vector Class Architecture

The `Vector` class (defined in `/src/include/duckdb/common/types/vector.hpp` and implemented in `/src/common/types/vector.cpp`) is the fundamental data structure for vectorized execution.

### Core Components

```cpp
class Vector {
protected:
    //! The vector type specifies how the data is physically stored
    VectorType vector_type;
    //! The logical type of the elements stored
    LogicalType type;
    //! Pointer to the actual data
    data_ptr_t data;
    //! The validity mask (NULL indicators)
    ValidityMask validity;
    //! Main buffer holding the vector data
    buffer_ptr<VectorBuffer> buffer;
    //! Auxiliary buffer (e.g., for string data)
    buffer_ptr<VectorBuffer> auxiliary;
    //! Cached hash values for dictionaries
    buffer_ptr<VectorBuffer> cached_hashes;
};
```

### Key Design Principles

1. **Type Safety**: Vectors are strongly typed with both logical types (SQL types like INTEGER, VARCHAR) and physical types (in-memory representations like INT32, VARCHAR).

2. **Reference Semantics**: Vectors can reference other vectors' data without copying. This is crucial for zero-copy operations.

3. **Lazy Materialization**: Vectors can represent data in compressed forms (DICTIONARY_VECTOR, SEQUENCE_VECTOR) and only materialize when necessary.

4. **Memory Management**: Vectors use smart pointers (`buffer_ptr`) for automatic memory management. No manual `new`/`delete`.

### Construction Patterns

```cpp
// Create an owning vector with allocated data
Vector v1(LogicalType::INTEGER);  // Allocates STANDARD_VECTOR_SIZE integers

// Create a vector with specific capacity
Vector v2(LogicalType::VARCHAR, 1024);

// Create a reference vector
Vector v3(other_vector);

// Create a constant vector
Vector v4(Value::INTEGER(42));

// Create a vector from a slice
Vector v5(other_vector, offset, end);
```

## DataChunk Class

The `DataChunk` class (defined in `/src/include/duckdb/common/types/data_chunk.hpp` and implemented in `/src/common/types/data_chunk.cpp`) represents a horizontal slice of a relation - a collection of vectors with the same cardinality.

### Structure

```cpp
class DataChunk {
public:
    //! The vectors owned by the DataChunk
    vector<Vector> data;

private:
    //! The number of tuples stored
    idx_t count;
    //! The maximum number of tuples that can be stored
    idx_t capacity;
    //! Vector caches for resetting
    vector<VectorCache> vector_caches;
};
```

### Key Characteristics

1. **Columnar Layout**: Each column is stored as a separate Vector, enabling column-oriented processing.

2. **Unified Cardinality**: All vectors in a DataChunk have the same logical count (number of valid tuples).

3. **Capacity vs Count**:
   - `capacity`: Maximum tuples the chunk can hold (typically STANDARD_VECTOR_SIZE)
   - `count`: Actual number of valid tuples currently in the chunk

4. **Reusability**: DataChunks can be `Reset()` to clear data and reuse allocated memory, avoiding allocations in tight loops.

### Initialization

```cpp
// Initialize with types
vector<LogicalType> types = {LogicalType::INTEGER, LogicalType::VARCHAR};
DataChunk chunk;
chunk.Initialize(allocator, types);

// Initialize without allocating data
chunk.InitializeEmpty(types);
```

### Common Operations

```cpp
// Set cardinality
chunk.SetCardinality(100);

// Access individual values
Value val = chunk.GetValue(col_idx, row_idx);
chunk.SetValue(col_idx, row_idx, val);

// Copy to another chunk
chunk.Copy(target_chunk);

// Flatten all vectors (remove compression)
chunk.Flatten();

// Reset for reuse
chunk.Reset();
```

## Vector Types

DuckDB supports multiple physical vector types, each optimized for different data patterns. These are defined in `/src/include/duckdb/common/enums/vector_type.hpp`:

```cpp
enum class VectorType : uint8_t {
    FLAT_VECTOR,       // Standard uncompressed vector
    FSST_VECTOR,       // String data compressed with FSST
    CONSTANT_VECTOR,   // Single constant value
    DICTIONARY_VECTOR, // Selection vector on top of another vector
    SEQUENCE_VECTOR    // Arithmetic sequence
};
```

### FLAT_VECTOR

The standard representation: a contiguous array of values with a validity mask.

**Memory Layout**:
```
data:     [v0] [v1] [v2] [v3] ... [vN]
validity: [1]  [1]  [0]  [1]  ... [1]  (bitpacked)
```

**When Used**: Default for most operations, after materialization.

**Access Pattern**:
```cpp
auto data = FlatVector::GetData<int32_t>(vector);
auto &validity = FlatVector::Validity(vector);
for (idx_t i = 0; i < count; i++) {
    if (validity.RowIsValid(i)) {
        int32_t value = data[i];
        // Process value
    }
}
```

### CONSTANT_VECTOR

Represents a single value repeated for all rows.

**Memory Layout**:
```
data:     [constant_value]
validity: [1] or [0]
```

**When Used**:
- Constant expressions in queries (`SELECT 42`)
- After filtering results in a single value
- Optimizing comparisons with constants

**Benefits**:
- O(1) memory regardless of vector size
- Operations can short-circuit

**Access Pattern**:
```cpp
if (vector.GetVectorType() == VectorType::CONSTANT_VECTOR) {
    if (ConstantVector::IsNull(vector)) {
        // All values are NULL
    } else {
        auto value = *ConstantVector::GetData<int32_t>(vector);
        // Single value applies to all rows
    }
}
```

### DICTIONARY_VECTOR

A selection vector (indirection layer) pointing into a child vector. This enables sharing data between vectors without copying.

**Memory Layout**:
```
sel_vector: [0] [5] [2] [5] [1] ... (indices into child)
child:      [a] [b] [c] [d] [e] [f] ...
```

**When Used**:
- After filtering with a selection vector
- Deduplication (multiple rows point to same value)
- Join results (probe side references build side)

**Benefits**:
- Zero-copy slicing and filtering
- Memory sharing for duplicate values
- Lazy materialization

**Access Pattern**:
```cpp
auto &sel = DictionaryVector::SelVector(vector);
auto &child = DictionaryVector::Child(vector);
auto child_data = FlatVector::GetData<int32_t>(child);

for (idx_t i = 0; i < count; i++) {
    idx_t child_idx = sel.get_index(i);
    int32_t value = child_data[child_idx];
}
```

### SEQUENCE_VECTOR

Represents an arithmetic sequence: `start + i * increment`.

**Memory Layout**:
```
buffer: [start] [increment] [count]
```

**When Used**:
- `generate_series()` function
- Row number generation
- Dense integer ranges

**Benefits**:
- O(1) memory
- No computation until materialization

**Access Pattern**:
```cpp
int64_t start, increment, sequence_count;
SequenceVector::GetSequence(vector, start, increment, sequence_count);

for (idx_t i = 0; i < count; i++) {
    int64_t value = start + i * increment;
}
```

### FSST_VECTOR

Strings compressed using the FSST (Fast Static Symbol Table) compression scheme.

**When Used**:
- String data with high redundancy
- Storage and wire transfer

**Access**: Must be decompressed to FLAT_VECTOR before processing.

## Selection Vectors

Selection vectors (defined in `/src/include/duckdb/common/types/selection_vector.hpp`) are arrays of indices that specify which rows are "selected" from a vector.

### Structure

```cpp
struct SelectionVector {
private:
    sel_t *sel_vector;  // Array of uint16_t indices
    buffer_ptr<SelectionData> selection_data;
};
```

### Purpose

Selection vectors enable **zero-copy filtering**. Instead of physically removing rows, we create an indirection layer:

```
Original data:  [10] [20] [30] [40] [50]
Filter: val > 20
SelectionVector: [2, 3, 4]  (indices of 30, 40, 50)
```

### Usage Patterns

```cpp
SelectionVector sel(count);

// Filter operation
idx_t result_count = 0;
for (idx_t i = 0; i < count; i++) {
    if (data[i] > threshold) {
        sel.set_index(result_count++, i);
    }
}

// Apply selection to create dictionary vector
Vector result(input);
result.Slice(sel, result_count);
```

### Incremental Selection Vector

For operations that don't filter, DuckDB provides an identity selection vector:

```cpp
auto sel = FlatVector::IncrementalSelectionVector();
// sel[i] == i for all i
```

This allows generic code to work with and without selection vectors uniformly.

## NULL Handling

NULLs are represented using a compact bitpacked validity mask (defined in `/src/include/duckdb/common/types/validity_mask.hpp`).

### ValidityMask Structure

```cpp
struct ValidityMask {
private:
    validity_t *validity_mask;  // uint64_t array
    static constexpr idx_t BITS_PER_VALUE = 64;
};
```

### Memory Layout

Each `uint64_t` stores validity for 64 rows:

```
validity_mask[0]: [bit0] [bit1] ... [bit63]  (rows 0-63)
validity_mask[1]: [bit0] [bit1] ... [bit63]  (rows 64-127)
```

- Bit = 1: Row is valid (NOT NULL)
- Bit = 0: Row is NULL

### All-Valid Optimization

If `validity_mask == nullptr`, all rows are valid. This saves memory for the common case of no NULLs:

```cpp
if (validity.AllValid()) {
    // Fast path: no NULL checks needed
    for (idx_t i = 0; i < count; i++) {
        process(data[i]);
    }
} else {
    // Slow path: check validity
    for (idx_t i = 0; i < count; i++) {
        if (validity.RowIsValid(i)) {
            process(data[i]);
        }
    }
}
```

### Operations

```cpp
ValidityMask validity;

// Check if row is valid
if (validity.RowIsValid(row_idx)) { ... }

// Set row to NULL
validity.SetInvalid(row_idx);

// Set row to NOT NULL
validity.SetValid(row_idx);

// Check if all rows are valid
if (validity.AllValid()) { ... }

// Ensure mask is allocated (for writing)
validity.EnsureWritable();

// Combine two masks (AND operation)
validity.Combine(other_validity, count);
```

### NULL Propagation

Most operations propagate NULLs following SQL semantics:

```cpp
// Binary operation example
for (idx_t i = 0; i < count; i++) {
    bool left_valid = left_validity.RowIsValid(i);
    bool right_valid = right_validity.RowIsValid(i);

    if (left_valid && right_valid) {
        result_data[i] = left_data[i] + right_data[i];
        result_validity.SetValid(i);
    } else {
        result_validity.SetInvalid(i);
    }
}
```

## How Operators Process Vectors

Physical operators in DuckDB's execution engine (see `/src/include/duckdb/execution/physical_operator.hpp`) process data in DataChunks using a **push-based** model.

### Execution Flow

1. **Source Operator** (e.g., TableScan) produces a DataChunk
2. **Pipeline Operators** (e.g., Filter, Projection) transform the chunk
3. **Sink Operator** (e.g., Hash Join build, Aggregate) consumes the chunk

### Operator Interface

```cpp
class PhysicalOperator {
    // Process a chunk of input data
    virtual OperatorResultType Execute(
        ExecutionContext &context,
        DataChunk &input,
        DataChunk &chunk,
        GlobalOperatorState &gstate,
        OperatorState &state
    ) const;

    // For source operators
    virtual SourceResultType GetData(
        ExecutionContext &context,
        DataChunk &chunk,
        OperatorSourceInput &input
    ) const;
};
```

### Example: Filter Operator

```cpp
// Simplified filter implementation
OperatorResultType Filter::Execute(
    ExecutionContext &context,
    DataChunk &input,
    DataChunk &output,
    GlobalOperatorState &gstate,
    OperatorState &state
) const {
    // Evaluate filter condition
    Vector predicate_result(LogicalType::BOOLEAN);
    ExpressionExecutor::Execute(filter_expr, input, predicate_result);

    // Create selection vector from predicate
    SelectionVector sel(STANDARD_VECTOR_SIZE);
    idx_t result_count = 0;

    auto pred_data = FlatVector::GetData<bool>(predicate_result);
    auto &pred_validity = FlatVector::Validity(predicate_result);

    for (idx_t i = 0; i < input.size(); i++) {
        if (pred_validity.RowIsValid(i) && pred_data[i]) {
            sel.set_index(result_count++, i);
        }
    }

    // Slice input vectors with selection
    output.Slice(input, sel, result_count);

    return result_count > 0 ? OperatorResultType::HAVE_MORE_OUTPUT
                            : OperatorResultType::NEED_MORE_INPUT;
}
```

### Table Scan Pattern

From `/src/execution/operator/scan/physical_table_scan.cpp`:

```cpp
SourceResultType PhysicalTableScan::GetData(
    ExecutionContext &context,
    DataChunk &chunk,
    OperatorSourceInput &input
) const {
    // Invoke table function to fill chunk
    TableFunctionInput data(bind_data.get(),
                           local_state.get(),
                           global_state.get());

    function.function(context.client, data, chunk);

    return chunk.size() == 0
        ? SourceResultType::FINISHED
        : SourceResultType::HAVE_MORE_OUTPUT;
}
```

## UnifiedVectorFormat

The `UnifiedVectorFormat` structure (defined in `/src/include/duckdb/common/types/vector.hpp`) provides a **unified interface** for reading vectors regardless of their internal representation.

### Purpose

Different vector types require different access patterns:
- FLAT_VECTOR: Direct array access
- DICTIONARY_VECTOR: Indirect access through selection vector
- CONSTANT_VECTOR: Single value for all rows

`UnifiedVectorFormat` normalizes these into a single interface.

### Structure

```cpp
struct UnifiedVectorFormat {
    const SelectionVector *sel;  // Selection vector (may be identity)
    data_ptr_t data;              // Pointer to actual data
    ValidityMask validity;        // Validity mask
    SelectionVector owned_sel;    // Storage for selection vector
    PhysicalType physical_type;   // Physical type for type safety
};
```

### Conversion

```cpp
Vector input = ...;
UnifiedVectorFormat vdata;
input.ToUnifiedFormat(count, vdata);

// Now access uniformly:
auto data = UnifiedVectorFormat::GetData<int32_t>(vdata);
for (idx_t i = 0; i < count; i++) {
    auto idx = vdata.sel->get_index(i);
    if (vdata.validity.RowIsValid(idx)) {
        int32_t value = data[idx];
    }
}
```

### Behind the Scenes

**FLAT_VECTOR**:
```cpp
vdata.sel = FlatVector::IncrementalSelectionVector();  // Identity
vdata.data = vector.data;
vdata.validity = vector.validity;
```

**CONSTANT_VECTOR**:
```cpp
vdata.sel = ConstantVector::ZeroSelectionVector();  // All zeros
vdata.data = vector.data;
vdata.validity = vector.validity;
```

**DICTIONARY_VECTOR**:
```cpp
vdata.sel = &DictionaryVector::SelVector(vector);
vdata.data = DictionaryVector::Child(vector).data;
vdata.validity = child.validity;
```

### Usage Example: Join Probe

From `/src/execution/nested_loop_join/nested_loop_join_inner.cpp`:

```cpp
// Convert both sides to unified format
UnifiedVectorFormat left_data, right_data;
left.ToUnifiedFormat(left_size, left_data);
right.ToUnifiedFormat(right_size, right_data);

auto ldata = UnifiedVectorFormat::GetData<int32_t>(left_data);
auto rdata = UnifiedVectorFormat::GetData<int32_t>(right_data);

// Nested loop join
for (idx_t l_idx = 0; l_idx < left_size; l_idx++) {
    auto left_idx = left_data.sel->get_index(l_idx);
    if (!left_data.validity.RowIsValid(left_idx)) continue;

    for (idx_t r_idx = 0; r_idx < right_size; r_idx++) {
        auto right_idx = right_data.sel->get_index(r_idx);
        if (!right_data.validity.RowIsValid(right_idx)) continue;

        if (ldata[left_idx] == rdata[right_idx]) {
            // Match found
        }
    }
}
```

## Vector Operations and Primitives

The `VectorOperations` namespace (defined in `/src/include/duckdb/common/vector_operations/vector_operations.hpp`) provides high-level primitives for operating on vectors.

### Categories

**Comparison Operations**:
```cpp
// result = left == right
VectorOperations::Equals(left, right, result, count);

// result = left < right
VectorOperations::LessThan(left, right, result, count);
```

**Boolean Operations**:
```cpp
// result = left AND right
VectorOperations::And(left, right, result, count);

// result = NOT left
VectorOperations::Not(left, result, count);
```

**NULL Operations**:
```cpp
// result = IS NULL(input)
VectorOperations::IsNull(input, result, count);

// Check if any NULLs exist
bool has_null = VectorOperations::HasNull(input, count);
```

**Hash Operations**:
```cpp
Vector hashes(LogicalType::HASH, count);

// Initial hash
VectorOperations::Hash(input, hashes, count);

// Combine with additional columns
VectorOperations::CombineHash(hashes, next_col, count);
```

**Copy Operations**:
```cpp
// Copy with selection
VectorOperations::Copy(source, target, sel, source_count,
                      source_offset, target_offset);
```

### Copy Implementation Example

From `/src/common/vector_operations/vector_copy.cpp`:

```cpp
void VectorOperations::Copy(
    const Vector &source,
    Vector &target,
    const SelectionVector &sel,
    idx_t source_count,
    idx_t source_offset,
    idx_t target_offset,
    idx_t copy_count
) {
    // Handle different vector types
    switch (source.GetVectorType()) {
    case VectorType::DICTIONARY_VECTOR: {
        // Merge selection vectors
        auto &child = DictionaryVector::Child(source);
        auto &dict_sel = DictionaryVector::SelVector(source);
        auto merged = dict_sel.Slice(sel, source_count);
        Copy(child, target, merged, source_count,
             source_offset, target_offset);
        break;
    }
    case VectorType::CONSTANT_VECTOR: {
        // Special handling for constants
        ...
        break;
    }
    case VectorType::FLAT_VECTOR: {
        // Direct copy with templated function
        auto ldata = FlatVector::GetData<T>(source);
        auto tdata = FlatVector::GetData<T>(target);
        for (idx_t i = 0; i < copy_count; i++) {
            auto source_idx = sel.get_index(source_offset + i);
            tdata[target_offset + i] = ldata[source_idx];
        }
        break;
    }
    }

    // Copy validity mask
    auto &tmask = FlatVector::Validity(target);
    auto &smask = FlatVector::Validity(source);
    tmask.CopySel(smask, sel, source_offset, target_offset, copy_count);
}
```

## Performance Implications

### Memory Access Patterns

**Good: Sequential Access**
```cpp
// Processes data sequentially - cache friendly
auto data = FlatVector::GetData<int32_t>(vector);
for (idx_t i = 0; i < count; i++) {
    process(data[i]);
}
```

**Bad: Random Access**
```cpp
// Random access pattern - cache unfriendly
for (idx_t i = 0; i < count; i++) {
    idx_t random_idx = indices[i];
    process(data[random_idx]);
}
```

### Branch Prediction

**Good: Minimize Branches in Tight Loops**
```cpp
// Branchless code using arithmetic
result[i] = data[i] * (data[i] > 0);
```

**Bad: Branchy Code**
```cpp
if (data[i] > 0) {
    result[i] = data[i];
} else {
    result[i] = 0;
}
```

### Type-Specific Fast Paths

Always check for constant vectors before processing:

```cpp
if (input.GetVectorType() == VectorType::CONSTANT_VECTOR) {
    if (ConstantVector::IsNull(input)) {
        // Set all results to NULL - O(1) operation
        ConstantVector::SetNull(result, true);
        return;
    }

    // Process single value once
    auto value = *ConstantVector::GetData<int32_t>(input);
    auto result_value = expensive_function(value);

    // Store as constant
    result.Reference(Value::INTEGER(result_value));
    return;
}

// Fall back to vector processing
```

### Materialization Overhead

Avoid unnecessary flattening:

```cpp
// BAD: Forces materialization
input.Flatten(count);
auto data = FlatVector::GetData<int32_t>(input);

// GOOD: Use UnifiedVectorFormat
UnifiedVectorFormat vdata;
input.ToUnifiedFormat(count, vdata);
auto data = UnifiedVectorFormat::GetData<int32_t>(vdata);
```

### String Performance

Strings require special handling due to variable length:

```cpp
// Efficient: Avoid redundant string copies
auto source_strings = FlatVector::GetData<string_t>(source);
auto target_strings = FlatVector::GetData<string_t>(target);

for (idx_t i = 0; i < count; i++) {
    if (validity.RowIsValid(i)) {
        // This adds string to target's string heap
        target_strings[i] = StringVector::AddString(target,
                                                    source_strings[i]);
    }
}
```

### Benchmark Insights

From DuckDB's design:
- **2048 vector size**: Optimal for modern CPU cache sizes
- **Bitpacked validity**: 64 validity bits per uint64_t minimizes memory bandwidth
- **Lazy materialization**: Dictionary/sequence vectors defer computation
- **SIMD-friendly**: Tight loops with minimal branching enable auto-vectorization

## Writing Vectorized Code

### Template: Unary Operation

```cpp
template <class T, class RESULT_TYPE>
void UnaryOperation(Vector &input, Vector &result, idx_t count) {
    // Handle constant vector fast path
    if (input.GetVectorType() == VectorType::CONSTANT_VECTOR) {
        result.SetVectorType(VectorType::CONSTANT_VECTOR);
        if (ConstantVector::IsNull(input)) {
            ConstantVector::SetNull(result, true);
        } else {
            auto input_val = *ConstantVector::GetData<T>(input);
            auto result_val = ConstantVector::GetData<RESULT_TYPE>(result);
            *result_val = UnaryFunc::Operation(input_val);
        }
        return;
    }

    // Convert to unified format
    UnifiedVectorFormat vdata;
    input.ToUnifiedFormat(count, vdata);

    // Get data pointers
    auto input_data = UnifiedVectorFormat::GetData<T>(vdata);
    auto result_data = FlatVector::GetData<RESULT_TYPE>(result);
    auto &result_validity = FlatVector::Validity(result);

    // Process vectorized
    for (idx_t i = 0; i < count; i++) {
        auto idx = vdata.sel->get_index(i);
        if (vdata.validity.RowIsValid(idx)) {
            result_data[i] = UnaryFunc::Operation(input_data[idx]);
            result_validity.SetValid(i);
        } else {
            result_validity.SetInvalid(i);
        }
    }
}
```

### Template: Binary Operation

```cpp
template <class LEFT_TYPE, class RIGHT_TYPE, class RESULT_TYPE>
void BinaryOperation(Vector &left, Vector &right, Vector &result, idx_t count) {
    // Handle constant vectors
    if (left.GetVectorType() == VectorType::CONSTANT_VECTOR &&
        right.GetVectorType() == VectorType::CONSTANT_VECTOR) {
        // Both constants - result is constant
        result.SetVectorType(VectorType::CONSTANT_VECTOR);
        if (ConstantVector::IsNull(left) || ConstantVector::IsNull(right)) {
            ConstantVector::SetNull(result, true);
        } else {
            auto lval = *ConstantVector::GetData<LEFT_TYPE>(left);
            auto rval = *ConstantVector::GetData<RIGHT_TYPE>(right);
            auto result_val = ConstantVector::GetData<RESULT_TYPE>(result);
            *result_val = BinaryFunc::Operation(lval, rval);
        }
        return;
    }

    // Convert to unified format
    UnifiedVectorFormat ldata, rdata;
    left.ToUnifiedFormat(count, ldata);
    right.ToUnifiedFormat(count, rdata);

    // Get data pointers
    auto lvals = UnifiedVectorFormat::GetData<LEFT_TYPE>(ldata);
    auto rvals = UnifiedVectorFormat::GetData<RIGHT_TYPE>(rdata);
    auto result_data = FlatVector::GetData<RESULT_TYPE>(result);
    auto &result_validity = FlatVector::Validity(result);

    // Process vectorized
    for (idx_t i = 0; i < count; i++) {
        auto lidx = ldata.sel->get_index(i);
        auto ridx = rdata.sel->get_index(i);

        if (ldata.validity.RowIsValid(lidx) &&
            rdata.validity.RowIsValid(ridx)) {
            result_data[i] = BinaryFunc::Operation(lvals[lidx], rvals[ridx]);
            result_validity.SetValid(i);
        } else {
            result_validity.SetInvalid(i);
        }
    }
}
```

### Template: Filter Operation

```cpp
idx_t FilterOperation(Vector &input, idx_t count, SelectionVector &sel) {
    // Convert to unified format
    UnifiedVectorFormat vdata;
    input.ToUnifiedFormat(count, vdata);

    auto data = UnifiedVectorFormat::GetData<T>(vdata);

    idx_t result_count = 0;
    for (idx_t i = 0; i < count; i++) {
        auto idx = vdata.sel->get_index(i);
        if (vdata.validity.RowIsValid(idx) &&
            FilterFunc::Operation(data[idx])) {
            sel.set_index(result_count++, i);
        }
    }

    return result_count;
}
```

### Best Practices

1. **Always handle CONSTANT_VECTOR first**: Provides massive speedup for constant expressions

2. **Use UnifiedVectorFormat for generic code**: Handles all vector types uniformly

3. **Check validity before processing**: Follow NULL semantics correctly

4. **Minimize allocations**: Reuse DataChunks and Vectors via `Reset()`

5. **Leverage VectorOperations**: Don't reimplement common operations

6. **Type safety**: Use templates and verify physical types match logical types

7. **Avoid unnecessary copies**: Use reference semantics (`Vector::Reference()`)

## Common Patterns

### Pattern: Aggregate Initialization

```cpp
// Initialize aggregate state
struct SumState {
    int64_t sum;
    bool is_set;
};

void Initialize(SumState &state) {
    state.sum = 0;
    state.is_set = false;
}

void Update(Vector &input, idx_t count, SumState &state) {
    UnifiedVectorFormat vdata;
    input.ToUnifiedFormat(count, vdata);
    auto data = UnifiedVectorFormat::GetData<int32_t>(vdata);

    for (idx_t i = 0; i < count; i++) {
        auto idx = vdata.sel->get_index(i);
        if (vdata.validity.RowIsValid(idx)) {
            state.sum += data[idx];
            state.is_set = true;
        }
    }
}

void Finalize(SumState &state, Vector &result, idx_t result_idx) {
    auto result_data = FlatVector::GetData<int64_t>(result);
    auto &result_validity = FlatVector::Validity(result);

    if (state.is_set) {
        result_data[result_idx] = state.sum;
        result_validity.SetValid(result_idx);
    } else {
        result_validity.SetInvalid(result_idx);
    }
}
```

### Pattern: String Operations

```cpp
void StringUppercase(Vector &input, Vector &result, idx_t count) {
    UnifiedVectorFormat vdata;
    input.ToUnifiedFormat(count, vdata);

    auto input_data = UnifiedVectorFormat::GetData<string_t>(vdata);
    auto result_data = FlatVector::GetData<string_t>(result);
    auto &result_validity = FlatVector::Validity(result);

    for (idx_t i = 0; i < count; i++) {
        auto idx = vdata.sel->get_index(i);
        if (vdata.validity.RowIsValid(idx)) {
            auto input_str = input_data[idx];

            // Allocate result string in vector's string heap
            auto result_str = StringVector::EmptyString(result, input_str.GetSize());
            auto result_ptr = result_str.GetDataWriteable();

            // Transform to uppercase
            for (idx_t j = 0; j < input_str.GetSize(); j++) {
                result_ptr[j] = std::toupper(input_str.GetData()[j]);
            }

            result_data[i] = result_str;
            result_validity.SetValid(i);
        } else {
            result_validity.SetInvalid(i);
        }
    }
}
```

### Pattern: Nested Type Processing

```cpp
void ProcessStruct(Vector &input, Vector &result, idx_t count) {
    // Process struct vector
    auto &input_children = StructVector::GetEntries(input);
    auto &result_children = StructVector::GetEntries(result);

    D_ASSERT(input_children.size() == result_children.size());

    // Process each child vector
    for (idx_t child_idx = 0; child_idx < input_children.size(); child_idx++) {
        ProcessVector(*input_children[child_idx],
                     *result_children[child_idx],
                     count);
    }

    // Copy validity from input to result
    auto &input_validity = FlatVector::Validity(input);
    auto &result_validity = FlatVector::Validity(result);
    result_validity.Initialize(input_validity);
}
```

### Pattern: Hash Table Probe

```cpp
void ProbeHashTable(Vector &keys, DataChunk &result,
                   HashTable &ht, idx_t count) {
    // Compute hashes
    Vector hashes(LogicalType::HASH, count);
    VectorOperations::Hash(keys, hashes, count);

    // Convert to unified format
    UnifiedVectorFormat key_data, hash_data;
    keys.ToUnifiedFormat(count, key_data);
    hashes.ToUnifiedFormat(count, hash_data);

    auto key_values = UnifiedVectorFormat::GetData<int32_t>(key_data);
    auto hash_values = UnifiedVectorFormat::GetData<hash_t>(hash_data);

    SelectionVector match_sel(count);
    idx_t match_count = 0;

    // Probe hash table
    for (idx_t i = 0; i < count; i++) {
        auto key_idx = key_data.sel->get_index(i);
        auto hash_idx = hash_data.sel->get_index(i);

        if (!key_data.validity.RowIsValid(key_idx)) continue;

        auto hash = hash_values[hash_idx];
        auto key = key_values[key_idx];

        if (ht.Lookup(hash, key)) {
            match_sel.set_index(match_count++, i);
        }
    }

    // Build result for matches
    result.Slice(match_sel, match_count);
}
```

## Debugging Vectorized Operations

### Verification Functions

DuckDB includes extensive verification in debug builds:

```cpp
// Verify vector consistency
vector.Verify(count);

// Verify entire DataChunk
chunk.Verify();

// Verify selection vector
sel.Verify(count, vector_size);
```

### Common Assertions

```cpp
// Verify vector type
D_ASSERT(vector.GetVectorType() == VectorType::FLAT_VECTOR);

// Verify type compatibility
D_ASSERT(left.GetType() == right.GetType());

// Verify count in bounds
D_ASSERT(count <= STANDARD_VECTOR_SIZE);
D_ASSERT(count <= chunk.GetCapacity());

// Verify selection vector indices
for (idx_t i = 0; i < count; i++) {
    D_ASSERT(sel.get_index(i) < vector_size);
}
```

### Print Functions

```cpp
// Print vector contents
vector.Print(count);

// Print DataChunk
chunk.Print();

// Print selection vector
sel.Print(count);

// Get string representation
string vec_str = vector.ToString(count);
string chunk_str = chunk.ToString();
```

### Debug Patterns

```cpp
#ifdef DEBUG
    // Verify input
    input.Verify(count);

    // Perform operation
    Operation(input, result, count);

    // Verify output
    result.Verify(count);

    // Verify cardinality preserved
    D_ASSERT(result.size() == expected_count);
#else
    Operation(input, result, count);
#endif
```

### Common Issues

**Issue: Incorrect selection vector usage**
```cpp
// WRONG: Using 'i' instead of sel[i]
for (idx_t i = 0; i < count; i++) {
    result[i] = data[i];  // Should be data[sel[i]]
}

// CORRECT:
for (idx_t i = 0; i < count; i++) {
    auto idx = sel.get_index(i);
    result[i] = data[idx];
}
```

**Issue: Validity mask not initialized**
```cpp
// WRONG: Setting values without initializing validity
auto data = FlatVector::GetData<int32_t>(result);
for (idx_t i = 0; i < count; i++) {
    data[i] = compute(i);
}
// Validity mask may be uninitialized!

// CORRECT:
auto data = FlatVector::GetData<int32_t>(result);
auto &validity = FlatVector::Validity(result);
for (idx_t i = 0; i < count; i++) {
    data[i] = compute(i);
    validity.SetValid(i);
}
```

**Issue: Not handling constant vectors**
```cpp
// WRONG: Assuming flat vector
auto data = FlatVector::GetData<int32_t>(input);  // Crashes on constant!

// CORRECT:
if (input.GetVectorType() == VectorType::CONSTANT_VECTOR) {
    // Handle constant
} else {
    // Handle flat/dictionary
    UnifiedVectorFormat vdata;
    input.ToUnifiedFormat(count, vdata);
    auto data = UnifiedVectorFormat::GetData<int32_t>(vdata);
}
```

### GDB Helpers

When debugging with GDB, useful commands:

```
# Print vector type
p vector.vector_type

# Print vector count
p count

# Print data pointer
p/x vector.data

# Print first few elements (for int32)
p ((int32_t*)vector.data)[0]@10

# Print validity mask
p vector.validity.validity_mask[0]

# Check if row is valid
p vector.validity.RowIsValid(5)
```

### Performance Debugging

Use DuckDB's profiling tools:

```sql
-- Enable profiling
PRAGMA enable_profiling;

-- Run query
SELECT ...;

-- View profile
PRAGMA show_profiling_output;
```

Look for:
- High execution time in specific operators
- Excessive materialization (Flatten operations)
- Inefficient selection vector usage

## Summary

DuckDB's vectorized execution engine achieves high performance through:

1. **Batch Processing**: Operating on 2048-element vectors amortizes overhead
2. **Type Polymorphism**: Multiple vector representations (flat, constant, dictionary, sequence)
3. **Lazy Evaluation**: Deferring materialization until necessary
4. **Zero-Copy Operations**: Selection vectors and references avoid data movement
5. **Cache-Friendly Design**: Columnar layout and sequential access patterns
6. **NULL Optimization**: Bitpacked validity masks with all-valid fast path
7. **UnifiedVectorFormat**: Generic code handling all vector types efficiently

When writing vectorized code:
- Always handle CONSTANT_VECTOR fast paths
- Use UnifiedVectorFormat for generic vector access
- Respect NULL semantics with validity masks
- Minimize allocations and copies
- Leverage existing VectorOperations primitives
- Add comprehensive verification in debug builds

This architecture enables DuckDB to achieve orders of magnitude better performance than traditional tuple-at-a-time processing while maintaining clean, maintainable code.
