# DuckDB Storage Subsystem Guide

This comprehensive guide covers the storage subsystem architecture for DuckDB developers. It explores the internals of how DuckDB stores, retrieves, and manages data both in-memory and on disk.

## Table of Contents

1. [Storage Architecture Overview](#storage-architecture-overview)
2. [In-Memory vs Persistent Storage](#in-memory-vs-persistent-storage)
3. [Row Groups and Segments](#row-groups-and-segments)
4. [Compression Techniques](#compression-techniques)
5. [Column Storage Format](#column-storage-format)
6. [Buffer Manager](#buffer-manager)
7. [Block Management](#block-management)
8. [Index Structures](#index-structures)
9. [ART Index Implementation](#art-index-implementation)
10. [Storage Operations](#storage-operations)
11. [Checkpointing and WAL](#checkpointing-and-wal)
12. [ATTACH and Multi-Database Support](#attach-and-multi-database-support)

---

## Storage Architecture Overview

DuckDB's storage subsystem is built around a columnar, block-oriented architecture designed for analytical workloads. The main components are:

### Core Components

**StorageManager** (`src/include/duckdb/storage/storage_manager.hpp`):
- Top-level coordinator for all storage operations
- Two implementations:
  - `SingleFileStorageManager`: Standard persistent storage in a single `.db` file
  - In-memory storage for temporary databases
- Manages:
  - Block allocation and storage
  - Write-Ahead Log (WAL)
  - Checkpoint creation
  - Database metadata

```cpp
class StorageManager {
    AttachedDatabase &db;
    string path;                           // Database file path
    unique_ptr<WriteAheadLog> wal;        // Transaction log
    bool read_only;                        // Read-only mode flag
    optional_idx storage_version;          // Serialization version
    StorageOptions storage_options;        // Configuration
};

class SingleFileStorageManager : public StorageManager {
    unique_ptr<BlockManager> block_manager;    // Block I/O
    unique_ptr<TableIOManager> table_io_manager;  // Table-specific I/O
};
```

**DataTable** (`src/include/duckdb/storage/data_table.hpp`):
- Physical representation of a table
- Contains metadata and references to data storage
- Manages row groups, indexes, and statistics
- Handles MVCC (Multi-Version Concurrency Control) through versioning

```cpp
class DataTable {
    shared_ptr<DataTableInfo> info;                    // Table metadata
    vector<ColumnDefinition> column_definitions;       // Schema
    mutex append_lock;                                 // Serializes appends
    shared_ptr<RowGroupCollection> row_groups;        // Data storage
    atomic<DataTableVersion> version;                  // MAIN_TABLE, ALTERED, or DROPPED
};
```

### Storage Hierarchy

```
Database
  └─ StorageManager
      ├─ BlockManager (manages blocks on disk)
      ├─ BufferManager (manages memory)
      └─ DataTables
          └─ RowGroupCollection
              └─ RowGroup (typically 122,880 rows)
                  └─ ColumnData (one per column)
                      └─ ColumnSegment (max ~256KB compressed)
                          ├─ Compressed data blocks
                          └─ Statistics
```

---

## In-Memory vs Persistent Storage

DuckDB supports two storage modes that share most of the same code paths:

### In-Memory Storage

**Characteristics:**
- Database path is `:memory:` or empty string
- Uses `InMemoryBlockManager` instead of file-backed storage
- All data stored in RAM managed by BufferManager
- No durability guarantees (data lost on crash)
- Optional automatic checkpointing to prevent memory exhaustion

**Block Manager:**
```cpp
// In-memory blocks are never written to disk
class InMemoryBlockManager : public BlockManager {
    // Blocks stored in memory only
    // No file I/O operations
    virtual bool InMemory() override { return true; }
};
```

**When to use:**
- Temporary analysis workloads
- Testing and development
- Fast data processing with no persistence needs

### Persistent Storage

**Characteristics:**
- Database stored in `.db` file
- Uses `SingleFileBlockManager` for disk I/O
- Supports ACID transactions via WAL
- Data survives process restarts
- Memory-mapped I/O for performance

**File Structure:**
```
database.db        # Main data file
  ├─ Header block (block 0)
  ├─ Metadata blocks (catalog, schema)
  ├─ Data blocks (table data)
  └─ Free list (recycled blocks)

database.db.wal    # Write-Ahead Log (optional)
  └─ Transaction records
```

**Block Manager Implementation:**
```cpp
class SingleFileBlockManager : public BlockManager {
    unique_ptr<FileHandle> handle;     // File handle
    idx_t iteration_count;              // For concurrency
    idx_t max_block;                    // Highest block ID
    BlockIndexManager index_manager;    // Free block tracking
};
```

### Hybrid Approach

DuckDB allows mixing storage modes:
- Main database persistent, temp tables in-memory
- Attached databases can have different storage modes
- Spilling to disk when memory limit exceeded

---

## Row Groups and Segments

DuckDB organizes data hierarchically into row groups and column segments for efficient columnar storage.

### Row Groups

**RowGroup** (`src/include/duckdb/storage/table/row_group.hpp`):
- Horizontal slice of a table (default: 122,880 rows)
- Contains one `ColumnData` per table column
- Maintains version information for MVCC
- Unit of parallelization for scans

```cpp
class RowGroup : public SegmentBase<RowGroup> {
    reference<RowGroupCollection> collection;               // Parent collection
    atomic<optional_ptr<RowVersionManager>> version_info;  // MVCC data
    shared_ptr<RowVersionManager> owned_version_info;
    vector<shared_ptr<ColumnData>> columns;                // Column data
    vector<MetaBlockPointer> column_pointers;              // Disk pointers
    vector<MetaBlockPointer> deletes_pointers;             // Delete info
    atomic<idx_t> allocation_size;                         // Memory usage
};
```

**Size Calculation:**
```cpp
// Default row group size
static constexpr idx_t ROW_GROUP_SIZE = 122880;  // ~120K rows

// This provides good balance between:
// - Parallelization granularity
// - Metadata overhead
// - Compression effectiveness
// - Memory usage
```

**Row Group Collection:**
```cpp
class RowGroupCollection {
    BlockManager &block_manager;
    const idx_t row_group_size;                    // Configurable
    atomic<idx_t> total_rows;
    vector<LogicalType> types;
    shared_ptr<RowGroupSegmentTree> row_groups;   // Segment tree for fast lookup
    TableStatistics stats;                         // Aggregate statistics
};
```

### Column Segments

**ColumnSegment** (`src/include/duckdb/storage/table/column_segment.hpp`):
- Stores a chunk of a single column within a row group
- Maximum size: `block_size` (typically 256KB compressed)
- Two types:
  - **Transient**: In-memory, being actively appended to
  - **Persistent**: Written to disk, immutable

```cpp
class ColumnSegment : public SegmentBase<ColumnSegment> {
    DatabaseInstance &db;
    LogicalType type;
    idx_t type_size;
    ColumnSegmentType segment_type;        // TRANSIENT or PERSISTENT
    SegmentStatistics stats;               // Min/max/null_count
    shared_ptr<BlockHandle> block;         // Block handle

    reference<CompressionFunction> function;  // Compression method
    block_id_t block_id;                      // Persistent block ID
    idx_t offset;                             // Offset within block
    idx_t segment_size;                       // Allocated size
    unique_ptr<CompressedSegmentState> segment_state;  // Compression state
};
```

**Segment Lifecycle:**

1. **Creation**: Transient segment allocated in memory
```cpp
auto segment = ColumnSegment::CreateTransientSegment(
    db, compression_function, type, start, segment_size, block_manager
);
```

2. **Append**: Data appended to transient segment
```cpp
idx_t appended = segment.Append(append_state, data, offset, count);
```

3. **Finalize**: Segment finalized when full
```cpp
idx_t bytes_used = segment.FinalizeAppend(append_state);
```

4. **Checkpoint**: Convert to persistent during checkpoint
```cpp
segment.ConvertToPersistent(context, &block_manager, block_id);
```

### ColumnData

**ColumnData** (`src/include/duckdb/storage/table/column_data.hpp`):
- Manages all segments for a single column across a row group
- Handles updates through `UpdateSegment`
- Maintains column statistics

```cpp
class ColumnData {
    idx_t start;                          // Starting row
    atomic<idx_t> count;                  // Row count
    BlockManager &block_manager;
    DataTableInfo &info;
    idx_t column_index;
    LogicalType type;

    ColumnSegmentTree data;               // Segment tree for fast lookup
    mutex update_lock;
    unique_ptr<UpdateSegment> updates;    // Update tracking
    unique_ptr<SegmentStatistics> stats;  // Column statistics
    atomic<idx_t> allocation_size;        // Memory usage
};
```

**Specialized Column Data Types:**

- **StandardColumnData**: Regular columns
- **ListColumnData**: Variable-length lists
- **StructColumnData**: Struct columns with child columns
- **ArrayColumnData**: Fixed-length arrays
- **ValidityColumnData**: Validity (NULL) masks
- **RowIdColumnData**: Row identifiers (internal)

---

## Compression Techniques

DuckDB employs sophisticated compression to reduce storage size and improve I/O performance.

### Compression Function Architecture

**CompressionFunction** (`src/include/duckdb/function/compression_function.hpp`):

Each compression method implements a standard interface:

```cpp
class CompressionFunction {
    CompressionType type;              // UNCOMPRESSED, RLE, DICTIONARY, etc.
    PhysicalType data_type;           // INT32, VARCHAR, etc.

    // Analysis phase: determine best compression
    compression_init_analyze_t init_analyze;
    compression_analyze_t analyze;
    compression_final_analyze_t final_analyze;

    // Compression phase: compress the data
    compression_init_compression_t init_compression;
    compression_compress_data_t compress;
    compression_compress_finalize_t compress_finalize;

    // Decompression phase: read the data
    compression_init_segment_scan_t init_scan;
    compression_scan_vector_t scan_vector;
    compression_scan_partial_t scan_partial;
    compression_fetch_row_t fetch_row;
    compression_skip_t skip;

    // Append phase (for transient segments)
    compression_init_append_t init_append;
    compression_append_t append;
    compression_finalize_append_t finalize_append;
};
```

### Available Compression Methods

**1. Uncompressed** (`src/storage/compression/uncompressed.cpp`)
- No compression applied
- Fastest access (direct memory access)
- Used when compression doesn't help
- Default for small segments

**2. Constant** (`src/storage/compression/numeric_constant.cpp`)
- All values in segment are identical
- Stores single value + count
- Extreme compression ratio (100:1+)
- Example: `1, 1, 1, 1, ...` → stores just `1`

**3. Run-Length Encoding (RLE)** (`src/storage/compression/rle.cpp`)
- Compresses consecutive duplicate values
- Stores: (value, run_length) pairs
- Excellent for sorted/grouped data
- Example: `1,1,1,2,2,3,3,3,3` → `(1,3), (2,2), (3,4)`

**4. Dictionary Compression** (`src/storage/compression/dictionary_compression.cpp`)
- Builds dictionary of unique values
- Replaces values with small integer indexes
- Great for low-cardinality columns (e.g., categories)
- Example: `"red", "blue", "red", "green", "blue"` →
  ```
  Dictionary: [0="red", 1="blue", 2="green"]
  Data: [0, 1, 0, 2, 1]
  ```

**5. Dictionary + FSST** (`src/storage/compression/dict_fsst.cpp`)
- Combines dictionary with FSST string compression
- FSST learns symbol table from data
- Excellent for text with repeated patterns
- Used for VARCHAR columns

**6. Bit Packing** (`src/storage/compression/bitpacking.cpp`)
- Packs integers using minimum required bits
- Analyzes min/max to determine bit width
- Example: Values 0-15 need only 4 bits (not 32)
- Frame-of-reference: stores offset + packed deltas

**7. ALP (Adaptive Lossless Compression)** (`src/storage/compression/alp/`)
- For floating-point numbers (FLOAT, DOUBLE)
- Adaptively chooses encoding based on data patterns
- Near-lossless compression of scientific data
- Variants: ALP and ALPRD (for doubles)

**8. Chimp** (`src/storage/compression/chimp/`)
- Time-series optimized compression
- Detects patterns in sequential numeric data
- XOR-based encoding for minimal changes
- Excellent for sensor data, metrics

**9. Patas** (`src/storage/compression/patas/`)
- For IEEE 754 floating-point data
- Exploits floating-point representation patterns
- Good compression with fast decompression

**10. ZSTD** (`src/storage/compression/zstd.cpp`)
- General-purpose compression using Zstandard
- Fallback when specialized methods don't apply
- Configurable compression levels

**11. FSST** (`src/storage/compression/fsst.cpp`)
- Fast Static Symbol Table compression for strings
- Learns symbol table from sample of data
- Very fast decompression

**12. Roaring Bitmaps** (`src/storage/compression/roaring/`)
- For compressed bitmap indexes
- Efficiently stores sparse bit sets
- Used internally for certain index types

### Compression Selection Process

DuckDB automatically selects the best compression for each segment:

```cpp
// 1. Initialize analysis state for each candidate
for (auto &function : compression_functions) {
    auto state = function.init_analyze(column_data, physical_type);
    analyze_states.push_back(std::move(state));
}

// 2. Analyze data
for (idx_t i = 0; i < vector_count; i++) {
    for (auto &state : analyze_states) {
        bool can_compress = function.analyze(*state, vectors[i], count);
        if (!can_compress) {
            // Remove this compression method
            remove(state);
        }
    }
}

// 3. Score each method
idx_t best_score = DConstants::INVALID_INDEX;
CompressionFunction *best_function = nullptr;

for (auto &state : analyze_states) {
    idx_t score = function.final_analyze(*state);
    if (score < best_score) {  // Lower is better (bytes used)
        best_score = score;
        best_function = &function;
    }
}

// 4. Use best compression function
```

**Compression Statistics:**

Each segment tracks compression effectiveness:

```cpp
struct SegmentStatistics {
    BaseStatistics statistics;    // Min, max, has_null, etc.
    idx_t uncompressed_size;     // Original size
    idx_t compressed_size;       // Actual disk size
    CompressionType compression_type;
};
```

---

## Column Storage Format

DuckDB uses a pure columnar format where each column is stored independently.

### Physical Layout

**On-Disk Structure:**

```
ColumnSegment:
  ┌─────────────────────────────────┐
  │ Segment Header (metadata)       │
  │  - type_id                      │
  │  - compression_type             │
  │  - count                        │
  │  - statistics (min/max/nulls)   │
  ├─────────────────────────────────┤
  │ Compression-Specific Header     │
  │  (e.g., dictionary, parameters) │
  ├─────────────────────────────────┤
  │ Compressed Data                 │
  │  (format varies by compression) │
  ├─────────────────────────────────┤
  │ Optional: String Heap           │
  │  (for VARCHAR overflow)         │
  └─────────────────────────────────┘
```

### Data Pointers

**DataPointer** (`src/include/duckdb/storage/data_pointer.hpp`):
- References to segment data on disk
- Used for lazy loading and checkpointing

```cpp
struct DataPointer {
    idx_t row_start;              // Starting row in table
    idx_t tuple_count;            // Number of rows
    block_id_t block_id;          // Block containing segment
    uint32_t offset;              // Offset within block
    CompressionType compression_type;
    BaseStatistics statistics;    // Segment statistics
    unique_ptr<ColumnSegmentState> segment_state;  // Compression state
};
```

**Persistent Column Data:**

```cpp
struct PersistentColumnData {
    PhysicalType physical_type;
    vector<DataPointer> pointers;          // One per segment
    vector<PersistentColumnData> child_columns;  // For nested types
    bool has_updates;                      // Uncommitted updates
};
```

### Nested Types

**List/Array Storage:**

```cpp
class ListColumnData : public ColumnData {
    shared_ptr<ColumnData> child_column;   // Actual element data
    // List column stores offsets into child_column
};

// Example: [[1,2], [3,4,5], [6]]
// Parent column: [0, 2, 5, 6]  (offsets)
// Child column:  [1, 2, 3, 4, 5, 6]  (elements)
```

**Struct Storage:**

```cpp
class StructColumnData : public ColumnData {
    vector<shared_ptr<ColumnData>> child_columns;  // One per field
    // Each field stored as separate column
};

// Example: STRUCT(a INT, b VARCHAR)
// Stored as two separate columns side-by-side
```

### String Storage

**Short Strings (≤ 12 bytes):**
- Stored inline within the segment
- No indirection or heap allocation
- Very fast access

**Long Strings (> 12 bytes):**
- Stored in string heap (separate blocks)
- Segment contains pointers to heap

```cpp
// String representation
struct string_t {
    union {
        struct {
            uint32_t length;
            char prefix[4];
            char *ptr;      // Points to heap for long strings
        } pointer;
        struct {
            uint32_t length;
            char inlined[12];  // Short strings stored here
        } inlined;
    } value;
};
```

### Validity (NULL) Mask

- Separate validity bit vector for each segment
- 1 bit per value (1 = valid, 0 = NULL)
- Often compressed separately
- Can use empty validity compression if no NULLs

---

## Buffer Manager

The BufferManager is responsible for managing memory and providing a unified interface for both in-memory and disk-backed data.

### Architecture

**BufferManager** (`src/include/duckdb/storage/buffer_manager.hpp`):

```cpp
class BufferManager {
    // Allocate temporary memory
    virtual shared_ptr<BlockHandle> AllocateTemporaryMemory(
        MemoryTag tag, idx_t block_size, bool can_destroy = true) = 0;

    // Allocate block-based memory
    virtual shared_ptr<BlockHandle> AllocateMemory(
        MemoryTag tag, BlockManager *block_manager, bool can_destroy = true) = 0;

    // Pin a block handle (load into memory if needed)
    virtual BufferHandle Pin(shared_ptr<BlockHandle> &handle) = 0;

    // Unpin a block handle (allow eviction)
    virtual void Unpin(shared_ptr<BlockHandle> &handle) = 0;

    // Memory limits
    virtual idx_t GetUsedMemory() const = 0;
    virtual idx_t GetMaxMemory() const = 0;
};
```

**StandardBufferManager** (`src/include/duckdb/storage/standard_buffer_manager.hpp`):

```cpp
class StandardBufferManager : public BufferManager {
    unique_ptr<BufferPool> buffer_pool;              // Memory pool
    unique_ptr<TemporaryMemoryManager> temp_manager; // Temp allocations

    DatabaseInstance &db;
    unique_ptr<TemporaryFileManager> temp_file_manager;  // Spill to disk
};
```

### Block Handles

**BlockHandle** (`src/include/duckdb/storage/buffer/block_handle.hpp`):
- Represents a single block of memory
- Reference counted (shared_ptr)
- Can be pinned (in use) or unpinned (evictable)
- Supports loading from disk on demand

```cpp
class BlockHandle : public enable_shared_from_this<BlockHandle> {
    BlockManager &block_manager;

    mutex lock;                           // Synchronization
    atomic<BlockState> state;            // LOADED or UNLOADED
    atomic<int32_t> readers;             // Number of active pins
    const block_id_t block_id;
    const MemoryTag tag;                 // What memory is used for

    unique_ptr<FileBuffer> buffer;       // Actual data
    BufferPoolReservation memory_charge; // Memory accounting

    atomic<idx_t> eviction_seq_num;      // For LRU eviction
    atomic<int64_t> lru_timestamp_msec;  // Last access time
};
```

**BufferHandle** (`src/include/duckdb/storage/buffer/buffer_handle.hpp`):
- RAII wrapper around BlockHandle
- Automatically unpins on destruction
- Provides pointer to data

```cpp
class BufferHandle {
    shared_ptr<BlockHandle> handle;
    optional_ptr<FileBuffer> node;

    data_ptr_t Ptr() const {
        return node->buffer;  // Raw pointer to data
    }
};

// Usage:
BufferHandle handle = buffer_manager.Pin(block_handle);
data_ptr_t data = handle.Ptr();  // Access data
// Auto-unpins when handle goes out of scope
```

### Buffer Pool

**BufferPool** (`src/include/duckdb/storage/buffer/buffer_pool.hpp`):
- Manages memory limits across the entire database
- Implements eviction policy (LRU-based)
- Tracks memory usage per MemoryTag

```cpp
class BufferPool {
    atomic<idx_t> maximum_memory;    // Memory limit

    // Eviction queues by buffer type
    vector<unique_ptr<EvictionQueue>> queues;

    // Memory usage tracking (per-tag)
    struct MemoryUsage {
        atomic<int64_t> memory_usage[MEMORY_TAG_COUNT + 1];
        // Cached counters for performance
    } memory_usage;

    TemporaryMemoryManager &temporary_memory_manager;
};
```

**Eviction Policy:**

```cpp
// When memory limit exceeded:
EvictionResult EvictBlocks(MemoryTag tag, idx_t extra_memory,
                          idx_t memory_limit) {
    while (GetUsedMemory() + extra_memory > memory_limit) {
        // Find oldest unpinned block
        auto victim = eviction_queue.Pop();

        if (victim.CanUnload()) {
            // Write to temp file if needed
            if (victim.MustWriteToTemporaryFile()) {
                WriteTemporaryBuffer(victim);
            }

            // Release memory
            victim.Unload();
        }
    }
}
```

### Memory Tags

**MemoryTag** enum categorizes memory usage:

```cpp
enum class MemoryTag : uint8_t {
    BASE_TABLE,        // Table data
    HASH_TABLE,        // Hash tables (joins, aggregates)
    PARQUET_READER,    // Parquet file buffers
    CSV_READER,        // CSV parsing
    ORDER_BY,          // Sort operations
    ART_INDEX,         // ART index nodes
    EXTENSION,         // Extension allocations
    // ... many more
};
```

Used for:
- Memory profiling and debugging
- Per-query memory limits
- Prioritizing evictions

### Temporary Files

When memory is exhausted, blocks can spill to temporary files:

```cpp
class TemporaryFileManager {
    string temp_directory;
    vector<unique_ptr<TemporaryFileHandle>> files;

    void WriteTemporaryBuffer(MemoryTag tag, block_id_t block_id,
                             FileBuffer &buffer);
    unique_ptr<FileBuffer> ReadTemporaryBuffer(MemoryTag tag,
                                              BlockHandle &block);
};
```

---

## Block Management

Blocks are the fundamental unit of disk storage in DuckDB.

### Block Structure

**Block Size:**
- Default: 256KB (262,144 bytes)
- Configurable via `--block_size` or DBConfig
- Must be set at database creation (immutable)

**Block Layout:**

```
Block (256 KB):
  ┌──────────────────────────────┐
  │ Block Header (8 bytes)       │
  │  - Checksum (CRC32)          │
  ├──────────────────────────────┤
  │ User Data                    │
  │  (~256 KB - header)          │
  │                              │
  │  Available for storage:      │
  │  - Segments                  │
  │  - Metadata                  │
  │  - Strings                   │
  └──────────────────────────────┘
```

### BlockManager

**BlockManager** (`src/include/duckdb/storage/block_manager.hpp`):

```cpp
class BlockManager {
    BufferManager &buffer_manager;
    optional_idx block_alloc_size;    // Total block size
    optional_idx block_header_size;   // Header overhead

    // Core operations
    virtual unique_ptr<Block> CreateBlock(block_id_t block_id,
                                         FileBuffer *source_buffer) = 0;
    virtual block_id_t GetFreeBlockId() = 0;
    virtual void MarkBlockAsFree(block_id_t block_id) = 0;
    virtual void MarkBlockAsModified(block_id_t block_id) = 0;

    virtual void Read(QueryContext context, Block &block) = 0;
    virtual void Write(FileBuffer &block, block_id_t block_id) = 0;

    // Block tracking
    unordered_map<block_id_t, weak_ptr<BlockHandle>> blocks;
    unique_ptr<MetadataManager> metadata_manager;
};
```

**SingleFileBlockManager** (`src/include/duckdb/storage/single_file_block_manager.hpp`):

```cpp
class SingleFileBlockManager : public BlockManager {
    DatabaseInstance &db;
    unique_ptr<FileHandle> handle;  // Database file

    idx_t iteration_count;           // For MVCC
    idx_t max_block;                 // Highest allocated block

    BlockIndexManager index_manager; // Free list
    idx_t free_list_id;              // Head of free list
    vector<shared_ptr<BlockHandle>> modified_blocks;  // For checkpoint
};
```

### Block Allocation

**Free List Management:**

```cpp
// Allocate new block
block_id_t GetFreeBlockId() {
    if (free_list_id != INVALID_BLOCK) {
        // Reuse free block
        block_id_t result = free_list_id;

        // Read next free block from free list
        Block free_block = ReadBlock(free_list_id);
        free_list_id = free_block.next_free;

        return result;
    } else {
        // Allocate new block at end of file
        return ++max_block;
    }
}

// Free block
void MarkBlockAsFree(block_id_t block_id) {
    // Add to head of free list
    Block block;
    block.next_free = free_list_id;
    WriteBlock(block_id, block);

    free_list_id = block_id;
}
```

### Metadata Manager

**MetadataManager** (`src/include/duckdb/storage/metadata/metadata_manager.hpp`):
- Manages small metadata blocks (1/64th of regular block size)
- Used for catalog, table metadata, index metadata
- Provides sub-block allocation within metadata blocks

```cpp
class MetadataManager {
    static constexpr idx_t METADATA_BLOCK_COUNT = 64;  // Per storage block

    BlockManager &block_manager;
    unordered_map<block_id_t, MetadataBlock> blocks;

    MetadataHandle AllocateHandle();  // Allocate metadata space
    MetadataHandle Pin(const MetadataPointer &pointer);  // Pin metadata
};

struct MetadataBlock {
    shared_ptr<BlockHandle> block;
    block_id_t block_id;
    vector<uint8_t> free_blocks;  // Bitmap of free sub-blocks
    atomic<bool> dirty;
};
```

### Partial Block Manager

**PartialBlockManager** (`src/include/duckdb/storage/partial_block_manager.hpp`):
- Manages partially-filled blocks during writes
- Allows multiple small segments to share a block
- Reduces internal fragmentation

```cpp
class PartialBlockManager {
    // Track partially-filled blocks
    map<idx_t, PartialBlock> partially_filled_blocks;

    // Allocate space within a partial block
    BlockPointer Allocate(idx_t size);

    // Flush partial blocks to disk
    void FlushPartialBlocks();
};
```

---

## Index Structures

DuckDB supports various index types for efficient data access.

### Index Base Class

**Index** (`src/include/duckdb/storage/index.hpp`):

```cpp
class Index {
    vector<column_t> column_ids;          // Indexed columns
    TableIOManager &table_io_manager;
    AttachedDatabase &db;

    virtual const string &GetIndexType() const = 0;
    virtual IndexConstraintType GetConstraintType() const = 0;

    // Operations
    virtual ErrorData Insert(IndexLock &lock, DataChunk &data,
                            Vector &row_ids) = 0;
    virtual void Delete(IndexLock &lock, DataChunk &entries,
                       Vector &row_ids) = 0;
    virtual void CommitDrop(IndexLock &lock) = 0;
};
```

**BoundIndex** (base for concrete implementations):

```cpp
class BoundIndex : public Index {
    string name;
    IndexConstraintType constraint_type;  // PRIMARY, UNIQUE, FOREIGN, NONE
    vector<unique_ptr<Expression>> unbound_expressions;

    atomic<idx_t> initial_index_size;
    mutex lock;

    // Constraint verification
    virtual void VerifyConstraint(DataChunk &chunk,
                                 IndexAppendInfo &info,
                                 ConflictManager &manager) = 0;
};
```

### Index Types

**1. ART (Adaptive Radix Tree) - Primary Index:**
- See detailed section below
- Default index type for PRIMARY KEY, UNIQUE, FOREIGN KEY
- Supports range queries and prefix searches

**2. Internal Indexes:**

**HashIndex** (for hash joins):
- Not persistent
- Built on-the-fly for hash joins
- Uses ChainedHashTable

**SortedIndex** (for ORDER BY):
- Temporary index for sorted data
- Used in merge operations

### Index Constraints

```cpp
enum class IndexConstraintType {
    NONE,       // No constraint (e.g., non-unique index)
    PRIMARY,    // Primary key constraint
    UNIQUE,     // Unique constraint
    FOREIGN,    // Foreign key constraint
};
```

### Index Storage

**IndexStorageInfo:**

```cpp
struct IndexStorageInfo {
    string name;
    idx_t index_storage_version;
    BlockPointer root_block_pointer;      // Root of index structure
    vector<BlockPointer> allocator_infos;  // Memory allocator state
    case_insensitive_map_t<Value> options;  // Index-specific options
};
```

**Serialization:**

```cpp
// Write index to disk
IndexStorageInfo SerializeToDisk(QueryContext context) {
    // Allocate blocks for index data
    // Write index nodes to blocks
    // Return pointers to root and allocators
}

// Load index from disk (lazy)
void InitializeFromStorage(IndexStorageInfo &info) {
    root_block_pointer = info.root_block_pointer;
    // Don't load data until first access
}
```

---

## ART Index Implementation

The Adaptive Radix Tree (ART) is DuckDB's primary index structure.

### ART Overview

**ART** (`src/include/duckdb/execution/index/art/art.hpp`):

```cpp
class ART : public BoundIndex {
    Node tree;  // Root node

    // Fixed-size allocators for different node types
    shared_ptr<array<unsafe_unique_ptr<FixedSizeAllocator>,
                     ALLOCATOR_COUNT>> allocators;

    bool owns_data;              // True if this ART owns its allocators
    bool verify_max_key_len;     // Check key length limits
    uint8_t prefix_count;        // Bytes in prefix

    static constexpr idx_t MAX_KEY_LEN = 8192;
    static constexpr uint8_t ALLOCATOR_COUNT = 9;
};
```

### Node Types

**Node** (`src/include/duckdb/execution/index/art/node.hpp`):

```cpp
enum class NType : uint8_t {
    PREFIX = 1,        // Prefix compression
    LEAF = 2,          // Leaf node (row IDs)
    NODE_4 = 3,        // 1-4 children
    NODE_16 = 4,       // 5-16 children
    NODE_48 = 5,       // 17-48 children
    NODE_256 = 6,      // 49-256 children
    LEAF_INLINED = 7,  // Inlined leaf (small)
    NODE_7_LEAF = 8,   // Node4 + leaf combined
    NODE_15_LEAF = 9,  // Node16 + leaf combined
    NODE_256_LEAF = 10,// Node256 + leaf combined
};

class Node : public IndexPointer {
    // Get/set node type
    NType GetType() const;

    // Child operations
    void ReplaceChild(const ART &art, uint8_t byte, Node child) const;
    static void InsertChild(ART &art, Node &node, uint8_t byte, Node child);
    static void DeleteChild(ART &art, Node &node, Node &prefix,
                           uint8_t byte);

    // Traversal
    const unsafe_optional_ptr<Node> GetChild(ART &art, uint8_t byte) const;
    const unsafe_optional_ptr<Node> GetNextChild(ART &art,
                                                 uint8_t &byte) const;
};
```

### Node Structures

**Node4**: 1-4 children
```cpp
struct Node4 {
    uint8_t count;           // Number of children (1-4)
    uint8_t key[4];         // Keys (partial bytes)
    Node children[4];       // Child pointers
};
// Linear search through keys
```

**Node16**: 5-16 children
```cpp
struct Node16 {
    uint8_t count;
    uint8_t key[16];        // Keys in sorted order
    Node children[16];
};
// SIMD-accelerated search
```

**Node48**: 17-48 children
```cpp
struct Node48 {
    uint8_t count;
    uint8_t child_index[256];   // key → child index (255 = no child)
    Node children[48];          // Actual children
};
// Direct lookup: child = children[child_index[key]]
```

**Node256**: 49-256 children
```cpp
struct Node256 {
    uint16_t count;
    Node children[256];     // Direct array of children (null if missing)
};
// Direct lookup: child = children[key]
```

**Leaf**: Stores row IDs
```cpp
struct Leaf {
    row_t row_ids[];  // Array of row IDs (variable size)
    idx_t count;      // Number of row IDs

    // For UNIQUE: single row_id
    // For non-UNIQUE: multiple row_ids
};
```

### Prefix Compression

**Prefix** node type eliminates common prefixes:

```cpp
// Without prefix compression:
//   "database", "data", "datum"
//   Store: d-a-t-a-b-a-s-e, d-a-t-a, d-a-t-u-m

// With prefix compression:
//   Common prefix: "dat"
//   Store once: Prefix("dat")
//            └─ a → "base", "a"
//            └─ u → "um"

struct Prefix {
    uint8_t bytes[PREFIX_SIZE];  // Prefix bytes
    uint8_t count;               // Number of bytes in prefix
    Node child;                  // Single child
};
```

### Key Encoding

**ARTKey**: Keys are normalized to byte arrays for comparison

```cpp
// Signed integers: flip sign bit to make negatives sort before positives
// Floats: special encoding to handle -0, NaN, etc.
// Strings: UTF-8 bytes with null terminator

template <bool IS_NOT_NULL = false>
void GenerateKeys(ArenaAllocator &allocator, DataChunk &input,
                 unsafe_vector<ARTKey> &keys) {
    // Transform each column into comparable byte array
    // Concatenate bytes for compound keys
}
```

### Operations

**Insert:**

```cpp
ErrorData Insert(IndexLock &l, DataChunk &data, Vector &row_ids) {
    // Generate keys from data
    GenerateKeys(allocator, data, keys);

    for (idx_t i = 0; i < count; i++) {
        // Traverse tree to find insertion point
        Node *node = &tree;
        idx_t depth = 0;

        while (!node->IsAnyLeaf()) {
            uint8_t byte = keys[i][depth];

            // Check prefix match
            if (node->GetType() == NType::PREFIX) {
                // Verify prefix matches
            }

            // Get/create child
            node = node->GetChildMutable(art, byte);
            depth++;
        }

        // Insert row_id into leaf
        InsertIntoLeaf(node, row_ids[i]);
    }
}
```

**Lookup:**

```cpp
bool SearchEqual(ARTKey &key, idx_t max_count, set<row_t> &row_ids) {
    Node *node = &tree;
    idx_t depth = 0;

    while (!node->IsAnyLeaf()) {
        uint8_t byte = key[depth];

        // Check prefix
        if (node->GetType() == NType::PREFIX) {
            if (!PrefixMatches(node, key, depth)) {
                return true;  // Not found, scan complete
            }
        }

        // Get child
        node = node->GetChild(art, byte);
        if (!node) {
            return true;  // Not found
        }
        depth++;
    }

    // Extract row IDs from leaf
    ExtractRowIds(node, row_ids);
    return row_ids.size() < max_count;
}
```

**Range Scan:**

```cpp
bool SearchGreater(ARTKey &key, bool equal, idx_t max_count,
                  set<row_t> &row_ids) {
    // Find first key >= search key
    // Use GetNextChild() to iterate forward
    // Collect row IDs until max_count reached
}
```

### Fixed-Size Allocators

Each node type has a dedicated allocator:

```cpp
class FixedSizeAllocator {
    idx_t segment_size;      // Size of each allocation segment
    idx_t allocation_size;   // Size of each node

    // Segments of pre-allocated nodes
    vector<BufferHandle> buffers;

    // Free list
    vector<idx_t> free_list;
};

// Allocators for ART:
allocators[0] = FixedSizeAllocator(sizeof(Node4));
allocators[1] = FixedSizeAllocator(sizeof(Node16));
allocators[2] = FixedSizeAllocator(sizeof(Node48));
allocators[3] = FixedSizeAllocator(sizeof(Node256));
allocators[4] = FixedSizeAllocator(sizeof(Leaf));
allocators[5] = FixedSizeAllocator(sizeof(Prefix));
// ... etc
```

### Persistence

**Serialization:**

```cpp
IndexStorageInfo SerializeToDisk(QueryContext context) {
    // 1. Allocate contiguous blocks for allocators
    for (auto &allocator : allocators) {
        BlockPointer ptr = allocator->Serialize(context);
        info.allocator_infos.push_back(ptr);
    }

    // 2. Root node already in allocator
    info.root_block_pointer = GetRootPointer();

    return info;
}
```

**Deserialization (lazy):**

```cpp
void Deserialize(const BlockPointer &pointer) {
    // Just store pointer, don't load yet
    for (idx_t i = 0; i < ALLOCATOR_COUNT; i++) {
        allocators[i]->SetBlockPointer(info.allocator_infos[i]);
    }

    // Tree will be loaded on first access
}
```

---

## Storage Operations

Core operations that interact with the storage layer.

### Scan Operations

**Table Scan:**

```cpp
class DataTable {
    void InitializeScan(ClientContext &context, DuckTransaction &transaction,
                       TableScanState &state,
                       const vector<StorageIndex> &column_ids,
                       optional_ptr<TableFilterSet> table_filters = nullptr) {
        row_groups->InitializeScan(context, state, column_ids, table_filters);
    }

    void Scan(DuckTransaction &transaction, DataChunk &result,
             TableScanState &state) {
        // Scan from row groups
        row_groups->Scan(transaction, result, state);

        // Apply local updates if needed
        local_storage->Scan(transaction, result, state);
    }
};
```

**Parallel Scan:**

```cpp
// Initialize parallel scan state
void InitializeParallelScan(ClientContext &context,
                           ParallelTableScanState &state) {
    idx_t row_group_count = row_groups->GetRowGroupCount();

    // Each thread gets different row groups
    state.current_row_group = 0;
    state.max_row_group = row_group_count;
}

// Each thread calls this to get next work unit
bool NextParallelScan(ClientContext &context,
                     ParallelTableScanState &state,
                     TableScanState &scan_state) {
    idx_t row_group_idx = state.current_row_group.fetch_add(1);

    if (row_group_idx >= state.max_row_group) {
        return false;  // No more work
    }

    // Initialize scan on this row group
    InitializeScanOnRowGroup(row_group_idx, scan_state);
    return true;
}
```

### Insert Operations

**Append:**

```cpp
void DataTable::InitializeAppend(DuckTransaction &transaction,
                                TableAppendState &state) {
    // Lock for appending
    AppendLock(state);

    // Initialize append state
    row_groups->InitializeAppend(transaction, state);
}

void DataTable::Append(DataChunk &chunk, TableAppendState &state) {
    // Append to row groups
    bool new_row_group = row_groups->Append(chunk, state);

    // Update indexes
    if (!indexes.Empty()) {
        AppendToIndexes(indexes, chunk, state.current_row);
    }

    // Update statistics
    UpdateStatistics(chunk);
}

void DataTable::FinalizeAppend(DuckTransaction &transaction,
                              TableAppendState &state) {
    row_groups->FinalizeAppend(transaction, state);
}
```

**Row Group Append:**

```cpp
bool RowGroupCollection::Append(DataChunk &chunk, TableAppendState &state) {
    auto current_row_group = GetRowGroup(state.row_group_idx);

    if (current_row_group->count + chunk.size() > row_group_size) {
        // Need new row group
        AppendRowGroup(state.lock, state.row_start);
        return true;
    }

    // Append to current row group
    current_row_group->Append(state, chunk);
    return false;
}
```

### Update Operations

**Update:**

```cpp
void DataTable::Update(TableUpdateState &state, ClientContext &context,
                      Vector &row_ids, const vector<PhysicalIndex> &column_ids,
                      DataChunk &data) {
    // Updates stored in UpdateSegment
    row_groups->Update(transaction, *this, row_ids, column_ids, data);

    // Update indexes
    // Delete old entries
    RemoveFromIndexes(state, old_data, row_ids);
    // Insert new entries
    AppendToIndexes(indexes, new_data, row_ids);
}
```

**UpdateSegment** (`src/include/duckdb/storage/table/update_segment.hpp`):
- Stores uncommitted updates separate from base data
- Organized by vector index (STANDARD_VECTOR_SIZE = 2048 rows)
- Merged into base data during checkpoint

```cpp
class UpdateSegment {
    ColumnData &column_data;
    StorageLock lock;
    unique_ptr<UpdateNode> root;   // B-tree of updates
    SegmentStatistics stats;
    StringHeap heap;                // For string updates
};

struct UpdateInfo {
    transaction_t transaction_id;
    row_t *ids;                     // Row IDs being updated
    idx_t count;
    data_ptr_t tuple_data;         // New values
    UpdateInfo *next;              // Linked list of updates
};
```

### Delete Operations

**Delete:**

```cpp
idx_t DataTable::Delete(TableDeleteState &state, ClientContext &context,
                       Vector &row_ids, idx_t count) {
    // Mark rows as deleted in version info
    idx_t deleted = row_groups->Delete(transaction, *this,
                                      row_ids.GetData<row_t>(), count);

    // Remove from indexes
    RemoveFromIndexes(context, row_ids, count);

    return deleted;
}
```

**Version Info:**

```cpp
class RowVersionManager {
    // Tracks inserts and deletes per row
    vector<TransactionVersionInfo> version_info;

    struct TransactionVersionInfo {
        transaction_t transaction_id;
        atomic<idx_t> insert_count;   // Rows inserted by transaction
        atomic<idx_t> delete_count;   // Rows deleted by transaction
        // Bitmaps for fine-grained tracking
    };
};
```

### Transaction Visibility

```cpp
bool RowGroup::Fetch(TransactionData transaction, idx_t row) {
    auto version_info = GetVersionInfo();
    if (!version_info) {
        return true;  // No versioning, row visible
    }

    idx_t row_in_group = row - start;

    // Check if row was deleted
    if (version_info->IsDeleted(transaction, row_in_group)) {
        return false;
    }

    // Check if row was inserted after transaction start
    if (version_info->IsInsertedAfter(transaction, row_in_group)) {
        return false;
    }

    return true;  // Row is visible
}
```

---

## Checkpointing and WAL

DuckDB uses Write-Ahead Logging (WAL) and checkpointing for durability and crash recovery.

### Write-Ahead Log

**WriteAheadLog** (`src/include/duckdb/storage/write_ahead_log.hpp`):

```cpp
class WriteAheadLog {
    AttachedDatabase &database;
    mutex wal_lock;
    unique_ptr<BufferedFileWriter> writer;
    string wal_path;                    // database.db.wal
    atomic<idx_t> wal_size;

    // Write operations to WAL
    void WriteCreateTable(const TableCatalogEntry &entry);
    void WriteInsert(DataChunk &chunk);
    void WriteUpdate(DataChunk &chunk, const vector<column_t> &column_path);
    void WriteDelete(DataChunk &chunk);
    void WriteAlter(CatalogEntry &entry, const AlterInfo &info);

    void Flush();  // fsync
};
```

**WAL Format:**

```
WAL File:
  ┌────────────────────────────────┐
  │ WAL Header                     │
  │  - version                     │
  │  - timestamp                   │
  ├────────────────────────────────┤
  │ WAL Entry 1                    │
  │  - type (CREATE/INSERT/etc)    │
  │  - table_id                    │
  │  - payload                     │
  ├────────────────────────────────┤
  │ WAL Entry 2                    │
  │  ...                           │
  ├────────────────────────────────┤
  │ Checkpoint Marker              │
  │  - checkpoint_id               │
  └────────────────────────────────┘
```

**WAL Entry Types:**

```cpp
enum class WALType : uint8_t {
    CREATE_TABLE = 1,
    DROP_TABLE = 2,
    ALTER_INFO = 3,
    CREATE_SCHEMA = 4,
    // ... catalog operations

    USE_TABLE = 10,      // Set current table for subsequent operations
    INSERT_TUPLE = 11,   // Insert rows
    DELETE_TUPLE = 12,   // Delete rows
    UPDATE_TUPLE = 13,   // Update rows
    ROW_GROUP_DATA = 14, // Bulk row group data

    CHECKPOINT = 99,     // Checkpoint marker
};
```

**Writing to WAL:**

```cpp
void WriteInsert(DataChunk &chunk) {
    lock_guard<mutex> lock(wal_lock);

    // Ensure WAL is initialized
    auto &writer = Initialize();

    // Write entry header
    writer.Write<WALType>(WALType::INSERT_TUPLE);
    writer.Write<idx_t>(chunk.size());

    // Serialize chunk data
    chunk.Serialize(writer);

    wal_size += writer.GetTotalWritten();
}
```

**WAL Replay:**

```cpp
unique_ptr<WriteAheadLog> WriteAheadLog::Replay(
        FileSystem &fs, AttachedDatabase &database, const string &wal_path) {

    auto handle = fs.OpenFile(wal_path, ...);

    WriteAheadLogDeserializer deserializer(database, handle);

    while (deserializer.HasMore()) {
        // Read entry type
        WALType type = deserializer.Read<WALType>();

        switch (type) {
        case WALType::CREATE_TABLE:
            ReplayCreateTable(deserializer);
            break;
        case WALType::INSERT_TUPLE:
            ReplayInsert(deserializer);
            break;
        case WALType::UPDATE_TUPLE:
            ReplayUpdate(deserializer);
            break;
        case WALType::DELETE_TUPLE:
            ReplayDelete(deserializer);
            break;
        case WALType::CHECKPOINT:
            // Truncate WAL at checkpoint
            break;
        }
    }
}
```

### Checkpointing

**CheckpointManager** (`src/include/duckdb/storage/checkpoint_manager.hpp`):

```cpp
class SingleFileCheckpointWriter : public CheckpointWriter {
    optional_ptr<ClientContext> context;
    unique_ptr<MetadataWriter> metadata_writer;         // Schema/catalog
    unique_ptr<MetadataWriter> table_metadata_writer;  // Table metadata
    PartialBlockManager partial_block_manager;         // Block sharing
    CheckpointType checkpoint_type;                    // FULL or PARTIAL
};
```

**Checkpoint Process:**

```cpp
void CreateCheckpoint() {
    // 1. Acquire checkpoint lock (blocks other checkpoints)
    auto checkpoint_lock = GetCheckpointLock();

    // 2. Write catalog entries
    WriteSchema(main_schema, serializer);
    for (auto &table : tables) {
        WriteTable(table, serializer);
    }

    // 3. Write table data
    for (auto &table : tables) {
        auto writer = GetTableDataWriter(table);
        table.data_table->Checkpoint(*writer, serializer);
    }

    // 4. Write metadata block
    MetaBlockPointer metadata_ptr = metadata_writer->GetMetaBlockPointer();

    // 5. Update header with new metadata pointer
    DatabaseHeader header;
    header.metadata_pointer = metadata_ptr;
    block_manager.WriteHeader(context, header);

    // 6. Flush all blocks
    block_manager.FileSync();

    // 7. Write checkpoint marker to WAL
    wal->WriteCheckpoint(metadata_ptr);

    // 8. Delete WAL (or truncate, based on options)
    if (options.wal_action == CheckpointWALAction::DELETE_WAL) {
        wal->Delete();
    }
}
```

**Table Checkpoint:**

```cpp
void DataTable::Checkpoint(TableDataWriter &writer, Serializer &serializer) {
    // Get shared checkpoint lock (prevents writes)
    auto lock = GetSharedCheckpointLock();

    // Write row groups
    row_groups->Checkpoint(writer, global_stats);

    // Write indexes
    for (auto &index : indexes.Indexes()) {
        index->SerializeToDisk(context, options);
    }
}

void RowGroupCollection::Checkpoint(TableDataWriter &writer,
                                   TableStatistics &global_stats) {
    for (auto &row_group : row_groups) {
        // Write persistent row groups as-is
        if (row_group.IsPersistent()) {
            row_group.SerializeRowGroupInfo();
            continue;
        }

        // Convert transient row groups to persistent
        auto write_data = row_group.WriteToDisk(writer);
        auto pointer = row_group.Checkpoint(write_data, writer, global_stats);

        // Update row group to reference new disk location
        row_group.Initialize(pointer);
    }
}
```

**RowGroup WriteToDisk:**

```cpp
RowGroupWriteData RowGroup::WriteToDisk(RowGroupWriter &writer) {
    RowGroupWriteData result;

    // Checkpoint each column
    for (idx_t col_idx = 0; col_idx < columns.size(); col_idx++) {
        auto &column = columns[col_idx];

        // Create checkpoint state
        auto checkpoint_state = column->Checkpoint(
            *this, writer.GetCheckpointInfo(col_idx)
        );

        result.states.push_back(std::move(checkpoint_state));
        result.statistics.push_back(column->GetStatistics());
    }

    return result;
}
```

**ColumnData Checkpoint:**

```cpp
unique_ptr<ColumnCheckpointState> StandardColumnData::Checkpoint(
        RowGroup &row_group, ColumnCheckpointInfo &info) {

    auto checkpoint_state = CreateCheckpointState(
        row_group, info.GetPartialBlockManager()
    );

    // Analyze and compress each segment
    for (auto &segment : data) {
        if (segment.IsPersistent()) {
            // Already on disk, just record pointer
            checkpoint_state->AddExistingSegment(segment);
        } else {
            // Compress and write transient segment
            CompressSegment(segment, checkpoint_state);
        }
    }

    return checkpoint_state;
}
```

### Automatic Checkpointing

```cpp
bool SingleFileStorageManager::AutomaticCheckpoint(idx_t estimated_wal_bytes) {
    // Check if WAL size exceeds threshold
    auto wal_size = GetWALSize();

    // Default threshold: 16MB
    static constexpr idx_t DEFAULT_CHECKPOINT_THRESHOLD = 16 * 1024 * 1024;

    if (wal_size > DEFAULT_CHECKPOINT_THRESHOLD) {
        // Trigger checkpoint in background
        CheckpointOptions options;
        options.action = CheckpointAction::CHECKPOINT_IF_REQUIRED;
        CreateCheckpoint(context, options);
        return true;
    }

    return false;
}
```

---

## ATTACH and Multi-Database Support

DuckDB supports attaching multiple databases to a single connection.

### AttachedDatabase

**AttachedDatabase** (`src/include/duckdb/main/attached_database.hpp`):

```cpp
class AttachedDatabase {
    DatabaseInstance &db;              // Parent instance
    string name;                       // Database alias
    unique_ptr<Catalog> catalog;       // Database catalog
    unique_ptr<StorageManager> storage;  // Storage manager
    AttachOptions options;             // Attach configuration

    bool is_temporary;                 // Temporary database
    bool is_system;                    // System database
};
```

**Attaching a Database:**

```cpp
// SQL: ATTACH 'path/to/db.duckdb' AS other_db;

void AttachDatabase(ClientContext &context, const string &path,
                   const string &alias, AttachOptions options) {
    // Create new attached database
    auto attached = make_uniq<AttachedDatabase>(db, alias, path, options);

    // Initialize storage manager
    attached->Initialize(context);

    // Load catalog from storage
    attached->GetCatalog().LoadFromStorage(context);

    // Register in database manager
    db.GetDatabaseManager().AddDatabase(std::move(attached));
}
```

**Cross-Database Queries:**

```sql
-- Query across databases
SELECT *
FROM main_db.schema.table1
JOIN other_db.schema.table2
  ON table1.id = table2.id;

-- Copy data between databases
INSERT INTO other_db.schema.target
SELECT * FROM main_db.schema.source;
```

**Implementation:**

```cpp
// Catalog resolution
optional_ptr<CatalogEntry> Catalog::GetEntry(
        ClientContext &context, const string &schema, const string &name) {

    // Try this catalog first
    auto entry = GetSchema(schema)->GetEntry(name);
    if (entry) {
        return entry;
    }

    // Search attached databases if allowed
    if (context.config.search_path_includes_attached) {
        for (auto &attached_db : db.GetDatabaseManager().GetDatabases()) {
            entry = attached_db->GetCatalog().GetEntry(schema, name);
            if (entry) {
                return entry;
            }
        }
    }

    return nullptr;
}
```

### System Catalog

The system catalog provides metadata about all attached databases:

```sql
-- List attached databases
SELECT * FROM duckdb_databases();

-- List tables across all databases
SELECT database_name, schema_name, table_name
FROM duckdb_tables()
ORDER BY database_name, schema_name, table_name;

-- List columns across all databases
SELECT database_name, schema_name, table_name, column_name, data_type
FROM duckdb_columns()
WHERE database_name = 'other_db';
```

### Read-Only Databases

```cpp
// Attach read-only database
AttachOptions options;
options.access_mode = AccessMode::READ_ONLY;
AttachDatabase(context, "readonly.duckdb", "readonly_db", options);

// Storage manager respects read-only flag
class SingleFileStorageManager {
    bool read_only;

    void Initialize() {
        if (read_only) {
            // Open file in read-only mode
            // Don't create WAL
            // Block all write operations
        }
    }
};
```

### Detaching Databases

```cpp
// SQL: DETACH other_db;

void DetachDatabase(ClientContext &context, const string &alias) {
    auto &db_manager = db.GetDatabaseManager();

    // Get attached database
    auto attached = db_manager.GetDatabase(alias);

    // Cannot detach system databases
    if (attached->IsSystem()) {
        throw BinderException("Cannot detach system database");
    }

    // Checkpoint if needed
    if (!attached->IsReadOnly()) {
        attached->GetStorageManager().CreateCheckpoint(context);
    }

    // Remove from manager
    db_manager.RemoveDatabase(alias);
}
```

---

## Best Practices for Storage Development

### Memory Management

1. **Always use RAII for buffer handles:**
```cpp
// Good
BufferHandle handle = buffer_manager.Pin(block_handle);
// ... use handle ...
// Auto-unpins when scope exits

// Bad
auto handle = buffer_manager.Pin(block_handle);
buffer_manager.Unpin(handle);  // Manual unpin, error-prone
```

2. **Use appropriate memory tags:**
```cpp
auto handle = buffer_manager.AllocateMemory(
    MemoryTag::HASH_TABLE,  // Tag for profiling
    block_manager,
    can_destroy
);
```

3. **Respect memory limits:**
```cpp
// Check before large allocations
if (buffer_manager.GetUsedMemory() + size > buffer_manager.GetMaxMemory()) {
    // Trigger eviction or fail gracefully
}
```

### Compression

1. **Let DuckDB choose compression automatically**
2. **For custom types, implement CompressionFunction interface**
3. **Test compression effectiveness with `PRAGMA storage_info`**

```sql
PRAGMA storage_info('my_table');
-- Shows compression ratios per segment
```

### Concurrency

1. **Use proper locking for shared structures:**
```cpp
// Table append requires lock
TableAppendState state;
data_table.AppendLock(state);
data_table.Append(chunk, state);
```

2. **Minimize lock contention:**
```cpp
// Good: Lock per operation
{
    lock_guard<mutex> lock(mutex);
    // Quick operation
}

// Bad: Hold lock during I/O
lock_guard<mutex> lock(mutex);
PerformSlowIO();  // Blocks other threads
```

### Performance

1. **Enable parallel scans for large tables**
2. **Use statistics for early pruning:**
```cpp
if (!row_group->CheckZonemap(filters)) {
    continue;  // Skip entire row group
}
```

3. **Batch operations:**
```cpp
// Good: Batch inserts
data_table.Append(chunk);  // 2048 rows

// Bad: Row-at-a-time
for (auto &row : rows) {
    data_table.InsertRow(row);
}
```

### Debugging

1. **Use `PRAGMA disable_checkpoint` for debugging WAL issues**
2. **Enable storage verification:**
```cpp
#ifdef DEBUG
data_table.Verify();
row_group.Verify();
#endif
```

3. **Inspect internal structures:**
```sql
PRAGMA storage_info('table');       -- Segment information
PRAGMA database_size;                -- Space usage
SELECT * FROM duckdb_indexes();      -- Index details
```

---

## Additional Resources

### Source Code Organization

- `src/storage/` - Core storage implementation
  - `buffer/` - Buffer manager and block handles
  - `checkpoint/` - Checkpointing logic
  - `compression/` - Compression algorithms
  - `metadata/` - Metadata management
  - `table/` - Row groups, segments, column data
  - `statistics/` - Storage statistics

- `src/include/duckdb/storage/` - Header files
- `src/execution/index/art/` - ART index implementation
- `test/sql/storage/` - Storage tests

### Related Documentation

- `CLAUDE.md` - Build instructions and coding guidelines
- `learning/architecture-deep-dive.md` - Overall architecture
- `learning/vectorized-execution.md` - Execution engine details

### Key Papers and References

- **ART**: "The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases" (Leis et al., 2013)
- **Compression**: DuckDB uses various techniques from database research
- **MVCC**: Multi-Version Concurrency Control for read consistency

---

## Glossary

- **Block**: Fixed-size unit of storage (default 256KB)
- **Row Group**: Horizontal partition of table (~122K rows)
- **Segment**: Vertical partition of column (max ~256KB compressed)
- **Buffer Manager**: Memory management layer
- **Block Manager**: Disk I/O management layer
- **WAL**: Write-Ahead Log for durability
- **Checkpoint**: Process of persisting in-memory data
- **MVCC**: Multi-Version Concurrency Control
- **ART**: Adaptive Radix Tree (index structure)
- **Metadata**: Information about data structure (not the data itself)
- **Transient**: In-memory, not yet persisted
- **Persistent**: Written to disk, immutable

---

## Conclusion

DuckDB's storage subsystem is a sophisticated columnar storage engine designed for analytical workloads. Key design principles include:

1. **Columnar storage** for efficient analytical queries
2. **Automatic compression** to minimize storage and I/O
3. **Lazy loading** to reduce memory usage
4. **MVCC** for read consistency without locks
5. **Efficient indexing** via ART for point and range queries
6. **Flexible memory management** supporting both in-memory and disk-backed storage

Understanding these components is essential for:
- Contributing to DuckDB storage development
- Optimizing query performance
- Debugging storage-related issues
- Implementing new storage features or extensions

For further exploration, review the source code in `src/storage/` and run experiments with `PRAGMA storage_info` to observe internal behavior.
