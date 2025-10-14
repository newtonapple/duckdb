# DuckDB Transaction Internals

This document provides a comprehensive deep dive into DuckDB's transaction management system for developers. It covers the implementation of Multi-Version Concurrency Control (MVCC), transaction lifecycle, commit/rollback mechanisms, and concurrency patterns.

## Table of Contents

1. [Transaction Architecture Overview](#transaction-architecture-overview)
2. [MVCC Implementation](#mvcc-implementation)
3. [Isolation Levels](#isolation-levels)
4. [Transaction Lifecycle](#transaction-lifecycle)
5. [Undo Buffer and Versioning](#undo-buffer-and-versioning)
6. [Conflict Detection and Resolution](#conflict-detection-and-resolution)
7. [Commit and Rollback Mechanisms](#commit-and-rollback-mechanisms)
8. [Concurrent Transaction Handling](#concurrent-transaction-handling)
9. [Lock-Free Data Structures](#lock-free-data-structures)
10. [Transaction Testing](#transaction-testing)
11. [Common Concurrency Patterns](#common-concurrency-patterns)

---

## Transaction Architecture Overview

DuckDB's transaction system is implemented across multiple key components in the `src/transaction/` directory:

### Core Classes

#### Transaction Hierarchy

```
Transaction (abstract base)
    └── DuckTransaction (concrete implementation)
```

**File:** `src/include/duckdb/transaction/transaction.hpp`

The base `Transaction` class provides:
- A reference to the `TransactionManager`
- A weak pointer to the `ClientContext`
- An atomic `active_query` counter (transaction_t)
- Read-only flag tracking

**File:** `src/include/duckdb/transaction/duck_transaction.hpp`

`DuckTransaction` extends this with:
- `start_time`: The transaction's start timestamp
- `transaction_id`: Unique transaction identifier
- `commit_id`: Commit timestamp (0 if not committed)
- `highest_active_query`: Used for cleanup coordination
- `catalog_version`: Tracks catalog changes
- `undo_buffer`: Stores version information for rollback
- `storage`: Local storage for uncommitted changes
- `write_lock`: Checkpoint coordination lock
- `sequence_usage`: Tracks sequence values used
- `modified_tables`: Tables touched by this transaction
- `active_locks`: Table-level checkpoint locks

#### TransactionManager Hierarchy

```
TransactionManager (abstract base)
    └── DuckTransactionManager (concrete implementation)
```

**File:** `src/include/duckdb/transaction/duck_transaction_manager.hpp`

The `DuckTransactionManager` manages:
- **Transaction Timestamps:**
  - `current_start_timestamp`: Monotonically increasing start time
  - `current_transaction_id`: High watermark transaction ID
  - `lowest_active_id`: Smallest active transaction ID
  - `lowest_active_start`: Smallest active start timestamp
  - `last_commit`: Most recent commit timestamp

- **Transaction Collections:**
  - `active_transactions`: Currently running transactions
  - `recently_committed_transactions`: Committed but still needed for MVCC
  - `old_transactions`: Awaiting garbage collection

- **Synchronization:**
  - `transaction_lock`: Protects transaction lists
  - `checkpoint_lock`: Coordinates checkpointing
  - `start_transaction_lock`: Prevents new transactions during FORCE CHECKPOINT
  - `wal_lock`: Serializes WAL writes
  - `cleanup_lock`: Coordinates cleanup operations
  - `cleanup_queue`: Ordered cleanup queue

### Key Design Principles

1. **Timestamp-Based MVCC:** Transactions use timestamps to determine visibility
2. **Lock-Free Reads:** Read operations don't block writers
3. **Optimistic Concurrency:** Transactions assume no conflicts until commit
4. **Deferred Cleanup:** Old versions are cleaned up asynchronously
5. **Write-Ahead Logging:** Changes are logged before commit

---

## MVCC Implementation

DuckDB implements Multi-Version Concurrency Control using a timestamp-based approach. Each tuple has version information that determines which transactions can see it.

### Timestamp Ranges

**File:** `src/transaction/duck_transaction_manager.cpp`

```cpp
// Transaction timestamps start at 2
current_start_timestamp = 2;

// Transaction IDs start very high (much higher than start_timestamp)
current_transaction_id = TRANSACTION_ID_START; // Typically 4611686018427387902
```

This large gap between timestamps and transaction IDs is crucial:
- **Start timestamps** (`< TRANSACTION_ID_START`): Used for committed data
- **Transaction IDs** (`>= TRANSACTION_ID_START`): Used for uncommitted data

### Version Visibility Rules

**File:** `src/storage/table/chunk_info.cpp`

DuckDB uses two operators to determine visibility:

```cpp
struct TransactionVersionOperator {
    static bool UseInsertedVersion(transaction_t start_time, transaction_t transaction_id, transaction_t id) {
        return id < start_time || id == transaction_id;
    }

    static bool UseDeletedVersion(transaction_t start_time, transaction_t transaction_id, transaction_t id) {
        return !UseInsertedVersion(start_time, transaction_id, id);
    }
};
```

**Visibility Logic:**
- A row is visible if:
  - It was inserted before the transaction started (`insert_id < start_time`), OR
  - It was inserted by this transaction (`insert_id == transaction_id`)
  - AND it wasn't deleted, OR was deleted after the transaction started

### Row Version Manager

**File:** `src/include/duckdb/storage/table/row_version_manager.hpp`

The `RowVersionManager` tracks version information for each vector (group of rows):

```cpp
class RowVersionManager {
private:
    mutex version_lock;
    idx_t start;  // Starting row number
    vector<unique_ptr<ChunkInfo>> vector_info;  // Per-vector version info
    bool has_changes;
    vector<MetaBlockPointer> storage_pointers;  // For checkpointing
};
```

Each vector has associated `ChunkInfo` that tracks:
1. **ChunkConstantInfo:** All rows have the same version (optimized case)
2. **ChunkVectorInfo:** Per-row version tracking (general case)

### Chunk Version Information

**File:** `src/include/duckdb/storage/table/chunk_info.hpp`

#### ChunkConstantInfo (Optimized)

Used when all rows in a chunk have the same insert/delete status:

```cpp
class ChunkConstantInfo : public ChunkInfo {
    transaction_t insert_id;   // When the chunk was inserted
    transaction_t delete_id;   // When the chunk was deleted (NOT_DELETED_ID if not deleted)
};
```

#### ChunkVectorInfo (General)

Used when rows have different version information:

```cpp
class ChunkVectorInfo : public ChunkInfo {
    transaction_t inserted[STANDARD_VECTOR_SIZE];  // Per-row insert timestamps
    transaction_t insert_id;                       // Fallback if all same
    bool same_inserted_id;                         // Optimization flag

    transaction_t deleted[STANDARD_VECTOR_SIZE];   // Per-row delete timestamps
    bool any_deleted;                              // Optimization flag
};
```

**Key Operations:**
- `GetSelVector()`: Returns visible row indices for a transaction
- `Fetch()`: Checks if a specific row is visible
- `Delete()`: Marks rows as deleted by a transaction
- `CommitDelete()`: Makes deletes permanent
- `Cleanup()`: Removes version info no longer needed

---

## Isolation Levels

DuckDB implements **Serializable** isolation by default. This is the strongest isolation level and prevents all anomalies.

### Implementation Approach

Unlike some databases that use multiple isolation levels, DuckDB achieves serializability through:

1. **Snapshot Isolation:** Each transaction sees a consistent snapshot
2. **Write Conflict Detection:** Concurrent writes to the same data are detected
3. **Catalog Validation:** Schema changes are validated at commit time

### Visibility Rules

**File:** `src/storage/table/chunk_info.cpp`

The core visibility check:

```cpp
static bool UseVersion(TransactionData transaction, transaction_t id) {
    return TransactionVersionOperator::UseInsertedVersion(
        transaction.start_time,
        transaction.transaction_id,
        id
    );
}
```

This ensures:
- **Read Your Own Writes:** Transaction sees its own uncommitted changes
- **No Dirty Reads:** Transaction only sees committed data from others
- **Repeatable Reads:** Same query returns same results within a transaction
- **No Phantom Reads:** No new rows appear/disappear during transaction

### Conflict Detection

**File:** `src/transaction/commit_state.cpp`

At commit time, DuckDB validates that modified tables haven't been altered:

```cpp
void CommitState::CommitEntry(UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::INSERT_TUPLE: {
        auto info = reinterpret_cast<AppendInfo *>(data);
        if (!info->table->IsMainTable()) {
            auto table_name = info->table->GetTableName();
            auto table_modification = info->table->TableModification();
            throw TransactionException(
                "Attempting to modify table %s but another transaction has %s this table",
                table_name, table_modification
            );
        }
        // Commit the append
        info->table->CommitAppend(commit_id, info->start_row, info->count);
        break;
    }
    // Similar checks for DELETE_TUPLE and UPDATE_TUPLE
    }
}
```

---

## Transaction Lifecycle

### 1. Transaction Start

**File:** `src/transaction/duck_transaction_manager.cpp`

```cpp
Transaction &DuckTransactionManager::StartTransaction(ClientContext &context) {
    // Obtain start_transaction_lock for write transactions (prevents FORCE CHECKPOINT)
    unique_ptr<lock_guard<mutex>> start_lock;
    if (!meta_transaction.IsReadOnly()) {
        start_lock = make_uniq<lock_guard<mutex>>(start_transaction_lock);
    }

    lock_guard<mutex> lock(transaction_lock);

    // Assign timestamps
    transaction_t start_time = current_start_timestamp++;
    transaction_t transaction_id = current_transaction_id++;

    // Update lowest active tracking
    if (active_transactions.empty()) {
        lowest_active_start = start_time;
        lowest_active_id = transaction_id;
    }

    // Create and register transaction
    auto transaction = make_uniq<DuckTransaction>(
        *this, context, start_time, transaction_id, last_committed_version
    );
    active_transactions.push_back(std::move(transaction));
    return transaction_ref;
}
```

**Key Points:**
- Timestamps are assigned under lock to ensure ordering
- Read-only transactions skip `start_transaction_lock` for better concurrency
- Each transaction gets a unique `start_time` and `transaction_id`
- The `lowest_active_*` values are updated for garbage collection

### 2. Transaction Execution

During execution, transactions:

#### Track Modifications

**File:** `src/transaction/duck_transaction.cpp`

```cpp
void DuckTransaction::PushDelete(DataTable &table, RowVersionManager &info,
                                 idx_t vector_idx, row_t rows[], idx_t count,
                                 idx_t base_row) {
    ModifyTable(table);  // Track that we modified this table

    // Optimize: check if rows are consecutive
    bool is_consecutive = true;
    for (idx_t i = 0; i < count; i++) {
        if (rows[i] != row_t(i)) {
            is_consecutive = false;
            break;
        }
    }

    // Allocate undo entry
    idx_t alloc_size = sizeof(DeleteInfo);
    if (!is_consecutive) {
        alloc_size += sizeof(uint16_t) * count;
    }

    auto undo_entry = undo_buffer.CreateEntry(UndoFlags::DELETE_TUPLE, alloc_size);
    auto delete_info = reinterpret_cast<DeleteInfo *>(undo_entry.Ptr());
    // ... populate delete_info
}
```

Similar methods exist for:
- `PushAppend()`: Track inserted rows
- `PushCatalogEntry()`: Track catalog changes
- `CreateUpdateInfo()`: Track updated rows
- `PushSequenceUsage()`: Track sequence usage

#### Maintain Local Storage

**File:** `src/include/duckdb/transaction/local_storage.hpp`

Uncommitted appends go into transaction-local storage:

```cpp
class LocalStorage {
private:
    ClientContext &context;
    DuckTransaction &transaction;
    LocalTableManager table_manager;  // Per-table local storage
};
```

This ensures:
- Uncommitted data is not visible to other transactions
- Rollback is efficient (just discard local storage)
- Scans can combine committed and local data

### 3. Transaction Commit

**File:** `src/transaction/duck_transaction_manager.cpp`

The commit process has several phases:

```cpp
ErrorData DuckTransactionManager::CommitTransaction(ClientContext &context,
                                                     Transaction &transaction_p) {
    auto &transaction = transaction_p.Cast<DuckTransaction>();
    unique_lock<mutex> t_lock(transaction_lock);

    // Phase 1: Check if we can checkpoint
    unique_ptr<StorageLockKey> lock;
    auto undo_properties = transaction.GetUndoProperties();
    auto checkpoint_decision = CanCheckpoint(transaction, lock, undo_properties);

    // Phase 2: Write to WAL (if not checkpointing)
    unique_ptr<StorageCommitState> commit_state;
    if (!checkpoint_decision.can_checkpoint && transaction.ShouldWriteToWAL(db)) {
        // Release transaction lock, hold WAL lock
        t_lock.unlock();
        unique_ptr<lock_guard<mutex>> held_wal_lock = make_uniq<lock_guard<mutex>>(wal_lock);
        error = transaction.WriteToWAL(db, commit_state);
        t_lock.lock();
    }

    // Phase 3: Assign commit timestamp
    transaction_t commit_id = GetCommitTimestamp();

    // Phase 4: Commit the transaction
    if (!error.HasError()) {
        error = transaction.Commit(db, commit_id, std::move(commit_state));
    }

    // Phase 5: Remove from active transactions
    auto cleanup_info = RemoveTransaction(transaction, store_transaction);

    // Phase 6: Schedule cleanup
    if (cleanup_info->ScheduleCleanup()) {
        lock_guard<mutex> q_lock(cleanup_queue_lock);
        cleanup_queue.emplace(std::move(cleanup_info));
    }

    // Phase 7: Perform cleanup
    t_lock.unlock();
    {
        lock_guard<mutex> c_lock(cleanup_lock);
        // Process cleanup queue
    }

    // Phase 8: Checkpoint if decided
    if (checkpoint_decision.can_checkpoint) {
        storage_manager.CreateCheckpoint(context, options);
    }

    return error;
}
```

**Important Design Decisions:**

1. **WAL Writing Outside Transaction Lock:** Improves concurrency for read-only transactions
2. **Cleanup After Commit:** Old versions are cleaned up asynchronously
3. **Automatic Checkpointing:** Large transactions trigger checkpoints
4. **Ordered Cleanup Queue:** Ensures cleanup happens in transaction order

### 4. Transaction Rollback

**File:** `src/transaction/duck_transaction_manager.cpp`

Rollback is simpler than commit:

```cpp
void DuckTransactionManager::RollbackTransaction(Transaction &transaction_p) {
    auto &transaction = transaction_p.Cast<DuckTransaction>();

    ErrorData error;
    {
        lock_guard<mutex> t_lock(transaction_lock);
        error = transaction.Rollback();

        auto cleanup_info = RemoveTransaction(transaction);
        if (cleanup_info->ScheduleCleanup()) {
            lock_guard<mutex> q_lock(cleanup_queue_lock);
            cleanup_queue.emplace(std::move(cleanup_info));
        }
    }

    // Process cleanup outside lock
    {
        lock_guard<mutex> c_lock(cleanup_lock);
        // ... cleanup processing
    }

    if (error.HasError()) {
        throw FatalException("Failed to rollback transaction: %s", error.Message());
    }
}
```

**File:** `src/transaction/duck_transaction.cpp`

```cpp
ErrorData DuckTransaction::Rollback() {
    try {
        storage->Rollback();    // Discard local storage
        undo_buffer.Rollback(); // Undo all changes
        return ErrorData();
    } catch (std::exception &ex) {
        return ErrorData(ex);
    }
}
```

---

## Undo Buffer and Versioning

The undo buffer is central to DuckDB's MVCC implementation. It stores information needed to rollback or cleanup transactions.

### Undo Buffer Structure

**File:** `src/include/duckdb/transaction/undo_buffer.hpp`

```cpp
class UndoBuffer {
public:
    explicit UndoBuffer(DuckTransaction &transaction, ClientContext &context);

    // Create an undo entry
    UndoBufferReference CreateEntry(UndoFlags type, idx_t len);

    // Check if changes were made
    bool ChangesMade();
    UndoBufferProperties GetProperties();

    // Lifecycle operations
    void Cleanup(transaction_t lowest_active_transaction);
    void WriteToWAL(WriteAheadLog &wal, optional_ptr<StorageCommitState> commit_state);
    void Commit(UndoBuffer::IteratorState &iterator_state, transaction_t commit_id);
    void RevertCommit(UndoBuffer::IteratorState &iterator_state, transaction_t transaction_id);
    void Rollback();

private:
    DuckTransaction &transaction;
    UndoBufferAllocator allocator;  // Manages memory for undo entries
};
```

### Undo Entry Types

**File:** `src/include/duckdb/common/enums/undo_flags.hpp`

```cpp
enum class UndoFlags : uint32_t {
    EMPTY_ENTRY = 0,
    CATALOG_ENTRY = 1,     // Catalog changes (CREATE/DROP/ALTER)
    INSERT_TUPLE = 2,      // Inserted rows
    DELETE_TUPLE = 3,      // Deleted rows
    UPDATE_TUPLE = 4,      // Updated rows
    SEQUENCE_VALUE = 5,    // Sequence usage
    ATTACHED_DATABASE = 6  // Database attachment
};
```

### Undo Entry Structure

**File:** `src/transaction/undo_buffer.cpp`

Each entry has a header followed by data:

```
[UndoFlags (4 bytes)][Length (4 bytes)][Data (variable)]
```

```cpp
constexpr uint32_t UNDO_ENTRY_HEADER_SIZE = sizeof(UndoFlags) + sizeof(uint32_t);

UndoBufferReference UndoBuffer::CreateEntry(UndoFlags type, idx_t len) {
    idx_t alloc_len = AlignValue<idx_t>(len + UNDO_ENTRY_HEADER_SIZE);
    auto handle = allocator.Allocate(alloc_len);
    auto data = handle.Ptr();

    // Write header
    Store<UndoFlags>(type, data);
    data += sizeof(UndoFlags);
    Store<uint32_t>(UnsafeNumericCast<uint32_t>(alloc_len - UNDO_ENTRY_HEADER_SIZE), data);

    handle.position += UNDO_ENTRY_HEADER_SIZE;
    return handle;
}
```

### Specific Undo Structures

#### Delete Info

**File:** `src/include/duckdb/transaction/delete_info.hpp`

```cpp
struct DeleteInfo {
    DataTable *table;
    RowVersionManager *version_info;
    idx_t vector_idx;
    idx_t count;
    idx_t base_row;
    bool is_consecutive;  // Optimization: rows are 0, 1, 2, ...

    uint16_t *GetRows();  // Per-row identifiers (if not consecutive)
};
```

#### Update Info

**File:** `src/include/duckdb/transaction/update_info.hpp`

```cpp
struct UpdateInfo {
    UpdateSegment *segment;
    DataTable *table;
    idx_t column_index;
    atomic<transaction_t> version_number;
    idx_t vector_index;
    sel_t N;     // Number of updated tuples
    sel_t max;   // Maximum capacity
    UndoBufferPointer prev;  // Previous version
    UndoBufferPointer next;  // Next version

    // Followed by:
    // sel_t tuples[max];  // Row IDs
    // T values[max];      // Updated values

    bool AppliesToTransaction(transaction_t start_time, transaction_t transaction_id) {
        return version_number > start_time && version_number != transaction_id;
    }
};
```

**Key Design:** Update info forms a version chain. When reading, DuckDB walks the chain to find the right version.

#### Append Info

**File:** `src/include/duckdb/transaction/append_info.hpp`

```cpp
struct AppendInfo {
    DataTable *table;
    idx_t start_row;  // First row appended
    idx_t count;      // Number of rows
};
```

### Undo Buffer Operations

#### Iteration

**File:** `src/transaction/undo_buffer.cpp`

The undo buffer supports forward and reverse iteration:

```cpp
template <class T>
void UndoBuffer::IterateEntries(UndoBuffer::IteratorState &state, T &&callback) {
    // Iterate from tail to head (insertion order)
    state.current = allocator.tail.get();
    while (state.current) {
        state.handle = allocator.buffer_manager.Pin(state.current->block);
        state.start = state.handle.Ptr();
        state.end = state.start + state.current->position;

        while (state.start < state.end) {
            UndoFlags type = Load<UndoFlags>(state.start);
            state.start += sizeof(UndoFlags);

            uint32_t len = Load<uint32_t>(state.start);
            state.start += sizeof(uint32_t);

            callback(type, state.start);
            state.start += len;
        }
        state.current = state.current->prev;
    }
}

template <class T>
void UndoBuffer::ReverseIterateEntries(T &&callback) {
    // Iterate from head to tail (reverse insertion order)
    auto current = allocator.head.get();
    while (current) {
        // Load all entries in this chunk
        vector<pair<UndoFlags, data_ptr_t>> nodes;
        // ... populate nodes ...

        // Process in reverse order
        for (idx_t i = nodes.size(); i > 0; i--) {
            callback(nodes[i - 1].first, nodes[i - 1].second);
        }
        current = current->next.get();
    }
}
```

**Why Both Directions?**
- **Forward (Commit/Cleanup):** Process changes in order they were made
- **Reverse (Rollback):** Undo changes in reverse order

#### Commit

**File:** `src/transaction/undo_buffer.cpp`

```cpp
void UndoBuffer::Commit(UndoBuffer::IteratorState &iterator_state, transaction_t commit_id) {
    CommitState state(transaction, commit_id);
    IterateEntries(iterator_state, [&](UndoFlags type, data_ptr_t data) {
        state.CommitEntry(type, data);
    });
}
```

**File:** `src/transaction/commit_state.cpp`

For each undo entry, the commit state:
- **INSERT_TUPLE:** Updates version info with commit_id
- **DELETE_TUPLE:** Marks rows as committed-deleted
- **UPDATE_TUPLE:** Sets version_number to commit_id
- **CATALOG_ENTRY:** Updates catalog timestamps

#### Rollback

**File:** `src/transaction/undo_buffer.cpp`

```cpp
void UndoBuffer::Rollback() {
    RollbackState state(transaction);
    ReverseIterateEntries([&](UndoFlags type, data_ptr_t data) {
        state.RollbackEntry(type, data);
    });
}
```

**File:** `src/transaction/rollback_state.cpp`

For each undo entry (in reverse):
- **INSERT_TUPLE:** Reverts the append
- **DELETE_TUPLE:** Unmarks rows as deleted (sets to NOT_DELETED_ID)
- **UPDATE_TUPLE:** Removes update from version chain
- **CATALOG_ENTRY:** Undoes catalog change

#### Cleanup

**File:** `src/transaction/undo_buffer.cpp`

```cpp
void UndoBuffer::Cleanup(transaction_t lowest_active_transaction) {
    CleanupState state(QueryContext(), lowest_active_transaction);
    UndoBuffer::IteratorState iterator_state;
    IterateEntries(iterator_state, [&](UndoFlags type, data_ptr_t data) {
        state.CleanupEntry(type, data);
    });
}
```

Cleanup removes version information that no transaction can see anymore. This is safe because `lowest_active_transaction` ensures no active transaction needs the old versions.

---

## Conflict Detection and Resolution

DuckDB uses an optimistic concurrency control approach: conflicts are detected at commit time rather than during execution.

### Write-Write Conflicts

**File:** `src/transaction/commit_state.cpp`

When a transaction commits, it checks if any modified table has been altered:

```cpp
void CommitState::CommitEntry(UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::INSERT_TUPLE:
    case UndoFlags::DELETE_TUPLE:
    case UndoFlags::UPDATE_TUPLE: {
        auto info = /* load info from data */;
        if (!info->table->IsMainTable()) {
            // Table has been modified (dropped/altered) by another transaction
            auto table_name = info->table->GetTableName();
            auto table_modification = info->table->TableModification();
            throw TransactionException(
                "Attempting to modify table %s but another transaction has %s this table",
                table_name, table_modification
            );
        }
        // Proceed with commit
        break;
    }
    }
}
```

The `IsMainTable()` check detects:
- Table was dropped by another transaction
- Table was altered (schema changed) by another transaction

### Catalog Conflicts

**File:** `src/transaction/duck_transaction_manager.cpp`

Catalog changes are tracked with version numbers:

```cpp
void DuckTransactionManager::PushCatalogEntry(Transaction &transaction_p,
                                              CatalogEntry &entry,
                                              data_ptr_t extra_data,
                                              idx_t extra_data_size) {
    auto &transaction = transaction_p.Cast<DuckTransaction>();

    // Mark transaction as having catalog changes
    transaction.catalog_version = ++last_uncommitted_catalog_version;
    transaction.PushCatalogEntry(entry, extra_data, extra_data_size);
}
```

At commit:

```cpp
if (transaction.catalog_version >= TRANSACTION_ID_START) {
    // Commit catalog changes
    transaction.catalog_version = ++last_committed_version;
}
```

This ensures catalog changes are visible in order.

### Checkpoint Conflicts

**File:** `src/transaction/duck_transaction_manager.cpp`

When deciding whether to checkpoint on commit:

```cpp
CheckpointDecision DuckTransactionManager::CanCheckpoint(
    DuckTransaction &transaction,
    unique_ptr<StorageLockKey> &lock,
    const UndoBufferProperties &undo_properties) {

    // Try to upgrade to exclusive checkpoint lock
    lock = transaction.TryGetCheckpointLock();
    if (!lock) {
        return CheckpointDecision("Failed to obtain checkpoint lock");
    }

    // Check if other transactions are active
    bool has_other_transactions = false;
    for (auto &active_transaction : active_transactions) {
        if (!RefersToSameObject(*active_transaction, transaction)) {
            has_other_transactions = true;
            break;
        }
    }

    if (has_other_transactions) {
        if (undo_properties.has_updates || undo_properties.has_dropped_entries) {
            // Cannot checkpoint: other transactions might need old data
            string other_transactions;
            for (auto &t : active_transactions) {
                if (!RefersToSameObject(*t, transaction)) {
                    other_transactions += "[" + to_string(t->transaction_id) + "]";
                }
            }
            return CheckpointDecision(
                "Transaction has performed updates and there are other transactions active\n"
                "Active transactions: " + other_transactions
            );
        }
        // Can do concurrent checkpoint
        checkpoint_type = CheckpointType::CONCURRENT_CHECKPOINT;
    }

    return CheckpointDecision(checkpoint_type);
}
```

### Update Conflicts (Version Chains)

**File:** `src/include/duckdb/transaction/update_info.hpp`

Update conflicts are handled through version chains:

```cpp
struct UpdateInfo {
    atomic<transaction_t> version_number;
    UndoBufferPointer prev;
    UndoBufferPointer next;

    bool AppliesToTransaction(transaction_t start_time, transaction_t transaction_id) {
        // This update is visible if:
        // - It was committed after this transaction started, OR
        // - It's not committed and not by this transaction
        return version_number > start_time && version_number != transaction_id;
    }

    template <class T>
    static void UpdatesForTransaction(UpdateInfo &current,
                                      transaction_t start_time,
                                      transaction_t transaction_id,
                                      T &&callback) {
        // Walk the version chain
        if (current.AppliesToTransaction(start_time, transaction_id)) {
            callback(current);
        }
        auto update_ptr = current.next;
        while (update_ptr.IsSet()) {
            auto pin = update_ptr.Pin();
            auto &info = Get(pin);
            if (info.AppliesToTransaction(start_time, transaction_id)) {
                callback(info);
            }
            update_ptr = info.next;
        }
    }
};
```

This allows multiple transactions to update the same row - each transaction sees the appropriate version.

---

## Commit and Rollback Mechanisms

### Commit Process Details

The commit process has multiple phases to ensure atomicity and durability.

#### Phase 1: Write to WAL

**File:** `src/transaction/duck_transaction.cpp`

```cpp
ErrorData DuckTransaction::WriteToWAL(AttachedDatabase &db,
                                      unique_ptr<StorageCommitState> &commit_state) noexcept {
    ErrorData error_data;
    try {
        auto &storage_manager = db.GetStorageManager();
        auto log = storage_manager.GetWAL();

        // Generate commit state
        commit_state = storage_manager.GenStorageCommitState(*log);

        // Write local storage to WAL
        storage->Commit(commit_state.get());

        // Write undo buffer to WAL
        undo_buffer.WriteToWAL(*log, commit_state.get());

        // If we wrote optimistic data, ensure it's persisted
        if (commit_state->HasRowGroupData()) {
            storage_manager.GetBlockManager().FileSync();
        }
    } catch (std::exception &ex) {
        error_data = ErrorData(ex);
    }

    if (commit_state && error_data.HasError()) {
        try {
            commit_state->RevertCommit();
            commit_state.reset();
        } catch (std::exception &) {
            // Ignore revert errors
        }
    }

    return error_data;
}
```

**Key Points:**
- Local storage (appends) written first
- Undo buffer (deletes/updates) written second
- Optimistic writes are fsynced if necessary
- On error, the WAL is truncated (RevertCommit)

#### Phase 2: In-Memory Commit

**File:** `src/transaction/duck_transaction.cpp`

```cpp
ErrorData DuckTransaction::Commit(AttachedDatabase &db,
                                  transaction_t new_commit_id,
                                  unique_ptr<StorageCommitState> commit_state) noexcept {
    this->commit_id = new_commit_id;

    if (!ChangesMade()) {
        return ErrorData();  // Read-only transaction
    }

    UndoBuffer::IteratorState iterator_state;
    try {
        // Commit local storage
        storage->Commit(commit_state.get());

        // Commit undo buffer (updates version info)
        undo_buffer.Commit(iterator_state, commit_id);

        // Flush WAL if we wrote to it
        if (commit_state) {
            commit_state->FlushCommit();
        }

        return ErrorData();
    } catch (std::exception &ex) {
        // Revert the commit
        undo_buffer.RevertCommit(iterator_state, this->transaction_id);
        if (commit_state) {
            commit_state->RevertCommit();
        }
        return ErrorData(ex);
    }
}
```

**Important:** If in-memory commit fails after WAL write, the commit is reverted both in-memory and in the WAL.

#### Phase 3: Make Visible

**File:** `src/transaction/commit_state.cpp`

When committing undo entries, version information is updated:

```cpp
void CommitState::CommitEntry(UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::INSERT_TUPLE: {
        auto info = reinterpret_cast<AppendInfo *>(data);
        info->table->CommitAppend(commit_id, info->start_row, info->count);
        break;
    }
    case UndoFlags::DELETE_TUPLE: {
        auto info = reinterpret_cast<DeleteInfo *>(data);
        info->version_info->CommitDelete(info->vector_idx, commit_id, *info);
        break;
    }
    case UndoFlags::UPDATE_TUPLE: {
        auto info = reinterpret_cast<UpdateInfo *>(data);
        info->version_number = commit_id;  // Atomic update
        break;
    }
    case UndoFlags::CATALOG_ENTRY: {
        auto catalog_entry = Load<CatalogEntry *>(data);
        CatalogSet::UpdateTimestamp(catalog_entry->Parent(), commit_id);
        // Drop old catalog entry data if applicable
        CommitEntryDrop(*catalog_entry, data + sizeof(CatalogEntry *));
        break;
    }
    }
}
```

### Rollback Process Details

#### Phase 1: Revert Changes

**File:** `src/transaction/duck_transaction.cpp`

```cpp
ErrorData DuckTransaction::Rollback() {
    try {
        storage->Rollback();
        undo_buffer.Rollback();
        return ErrorData();
    } catch (std::exception &ex) {
        return ErrorData(ex);
    }
}
```

**File:** `src/transaction/local_storage.cpp`

```cpp
void LocalStorage::Rollback() {
    // For each table with local data
    auto tables = table_manager.MoveEntries();
    for (auto &entry : tables) {
        auto &storage = entry.second;
        storage->Rollback();  // Discard local row groups
    }
}
```

#### Phase 2: Undo Buffer Rollback

**File:** `src/transaction/rollback_state.cpp`

```cpp
void RollbackState::RollbackEntry(UndoFlags type, data_ptr_t data) {
    switch (type) {
    case UndoFlags::INSERT_TUPLE: {
        auto info = reinterpret_cast<AppendInfo *>(data);
        info->table->RevertAppend(transaction, info->start_row, info->count);
        break;
    }
    case UndoFlags::DELETE_TUPLE: {
        auto info = reinterpret_cast<DeleteInfo *>(data);
        // Set delete_id back to NOT_DELETED_ID
        info->version_info->CommitDelete(info->vector_idx, NOT_DELETED_ID, *info);
        break;
    }
    case UndoFlags::UPDATE_TUPLE: {
        auto info = reinterpret_cast<UpdateInfo *>(data);
        info->segment->RollbackUpdate(*info);  // Remove from version chain
        break;
    }
    case UndoFlags::CATALOG_ENTRY: {
        auto catalog_entry = Load<CatalogEntry *>(data);
        catalog_entry->set->Undo(*catalog_entry);  // Revert catalog change
        break;
    }
    case UndoFlags::ATTACHED_DATABASE: {
        auto db = Load<AttachedDatabase *>(data);
        DatabaseManager::Get(db->GetDatabase()).DetachInternal(db->name);
        break;
    }
    }
}
```

**Key Insight:** Rollback uses `NOT_DELETED_ID` for deletes, which makes rows visible again. For updates, the update info is removed from the version chain.

---

## Concurrent Transaction Handling

DuckDB's transaction system is designed to handle high concurrency with minimal blocking.

### Transaction Removal and Cleanup

**File:** `src/transaction/duck_transaction_manager.cpp`

When a transaction finishes, it goes through several stages:

```cpp
unique_ptr<DuckCleanupInfo> DuckTransactionManager::RemoveTransaction(
    DuckTransaction &transaction,
    bool store_transaction) noexcept {

    auto cleanup_info = make_uniq<DuckCleanupInfo>();

    // Find transaction and compute new lowest values
    idx_t t_index = active_transactions.size();
    auto lowest_start_time = TRANSACTION_ID_START;
    auto lowest_transaction_id = MAX_TRANSACTION_ID;
    auto lowest_active_query = MAXIMUM_QUERY_ID;

    for (idx_t i = 0; i < active_transactions.size(); i++) {
        if (active_transactions[i].get() == &transaction) {
            t_index = i;
            continue;
        }
        lowest_start_time = MinValue(lowest_start_time, active_transactions[i]->start_time);
        lowest_transaction_id = MinValue(lowest_transaction_id, active_transactions[i]->transaction_id);
        lowest_active_query = MinValue(lowest_active_query,
                                      active_transactions[i]->active_query.load());
    }

    lowest_active_start = lowest_start_time;
    lowest_active_id = lowest_transaction_id;

    // Decide what to do with the transaction
    auto current_transaction = std::move(active_transactions[t_index]);
    auto current_query = DatabaseManager::Get(db).ActiveQueryNumber();

    if (store_transaction) {
        if (transaction.commit_id != 0) {
            // Committed: add to recently_committed_transactions
            recently_committed_transactions.push_back(std::move(current_transaction));
        } else {
            // Rolled back: add to old_transactions
            current_transaction->highest_active_query = current_query;
            old_transactions.push_back(std::move(current_transaction));
        }
    } else if (transaction.ChangesMade()) {
        // Can cleanup immediately
        current_transaction->awaiting_cleanup = true;
        cleanup_info->transactions.push_back(std::move(current_transaction));
    }

    cleanup_info->lowest_start_time = lowest_start_time;
    active_transactions.unsafe_erase_at(t_index);

    // Move recently_committed to old_transactions if safe
    idx_t i = 0;
    for (; i < recently_committed_transactions.size(); i++) {
        if (recently_committed_transactions[i]->commit_id >= lowest_start_time) {
            break;  // Still needed by active transactions
        }

        recently_committed_transactions[i]->awaiting_cleanup = true;
        recently_committed_transactions[i]->highest_active_query = current_query;
        old_transactions.push_back(std::move(recently_committed_transactions[i]));
    }

    if (i > 0) {
        recently_committed_transactions.erase(
            recently_committed_transactions.begin(),
            recently_committed_transactions.begin() + i
        );
    }

    // Move old_transactions to cleanup if safe
    i = active_transactions.empty() ? old_transactions.size() : 0;
    for (; i < old_transactions.size(); i++) {
        if (old_transactions[i]->highest_active_query >= lowest_active_query) {
            break;  // Still might be accessed by active queries
        }
    }

    if (i > 0) {
        for (idx_t t_idx = 0; t_idx < i; t_idx++) {
            cleanup_info->transactions.push_back(std::move(old_transactions[t_idx]));
        }
        old_transactions.erase(
            old_transactions.begin(),
            old_transactions.begin() + i
        );
    }

    return cleanup_info;
}
```

**Transaction Lifecycle After Completion:**

1. **Active Transactions:** Currently running
2. **Recently Committed Transactions:** Committed but needed for MVCC visibility
3. **Old Transactions:** No longer needed for MVCC, waiting for queries to finish
4. **Cleanup Queue:** Ready to be cleaned up
5. **Cleaned Up:** Memory freed

### Cleanup Queue Processing

**File:** `src/transaction/duck_transaction_manager.cpp`

Cleanup is done outside the transaction lock:

```cpp
// In CommitTransaction or RollbackTransaction:

// Schedule cleanup
if (cleanup_info->ScheduleCleanup()) {
    lock_guard<mutex> q_lock(cleanup_queue_lock);
    cleanup_queue.emplace(std::move(cleanup_info));
}

// Release transaction lock
t_lock.unlock();

// Process cleanup
{
    lock_guard<mutex> c_lock(cleanup_lock);
    unique_ptr<DuckCleanupInfo> top_cleanup_info;
    {
        lock_guard<mutex> q_lock(cleanup_queue_lock);
        if (!cleanup_queue.empty()) {
            top_cleanup_info = std::move(cleanup_queue.front());
            cleanup_queue.pop();
        }
    }
    if (top_cleanup_info) {
        top_cleanup_info->Cleanup();  // Calls undo_buffer.Cleanup()
    }
}
```

**Why Separate Locks?**
- `cleanup_lock`: Only one thread can do cleanup at a time
- `cleanup_queue_lock`: Multiple threads can add to the queue
- `transaction_lock`: Not held during cleanup (improves concurrency)

### Checkpoint Coordination

**File:** `src/include/duckdb/storage/storage_lock.hpp`

```cpp
class StorageLock {
public:
    unique_ptr<StorageLockKey> GetExclusiveLock();
    unique_ptr<StorageLockKey> GetSharedLock();
    unique_ptr<StorageLockKey> TryGetExclusiveLock();
    unique_ptr<StorageLockKey> TryUpgradeCheckpointLock(StorageLockKey &lock);
};
```

**File:** `src/transaction/duck_transaction.cpp`

Write transactions hold a shared checkpoint lock:

```cpp
void DuckTransaction::SetReadWrite() {
    Transaction::SetReadWrite();
    // Obtain shared checkpoint lock to prevent concurrent checkpoints
    write_lock = transaction_manager.SharedCheckpointLock();
}
```

At commit, the transaction tries to upgrade to exclusive:

```cpp
unique_ptr<StorageLockKey> DuckTransaction::TryGetCheckpointLock() {
    if (!write_lock) {
        throw InternalException("TryUpgradeCheckpointLock - but thread has no shared lock!?");
    }
    return transaction_manager.TryUpgradeCheckpointLock(*write_lock);
}
```

**Checkpoint Locking Strategy:**

- **Shared Lock:** Held by active write transactions
  - Prevents checkpointing during transaction
  - Multiple writers can hold shared lock simultaneously

- **Exclusive Lock:** Needed for checkpointing
  - Can only be obtained when no shared locks exist
  - Or when only one shared lock exists (the upgrading transaction)

### Table-Level Locks

**File:** `src/transaction/duck_transaction.cpp`

Transactions track which tables they modify:

```cpp
void DuckTransaction::ModifyTable(DataTable &tbl) {
    lock_guard<mutex> guard(modified_tables_lock);
    auto table_ref = reference<DataTable>(tbl);
    auto entry = modified_tables.find(table_ref);
    if (entry != modified_tables.end()) {
        return;  // Already tracked
    }
    // Hold owning reference to prevent table deletion
    modified_tables.insert(make_pair(table_ref, tbl.shared_from_this()));
}
```

For scanning, transactions obtain checkpoint locks on tables:

```cpp
shared_ptr<CheckpointLock> DuckTransaction::SharedLockTable(DataTableInfo &info) {
    unique_lock<mutex> transaction_lock(active_locks_lock);
    auto entry = active_locks.find(info);
    if (entry == active_locks.end()) {
        entry = active_locks.insert(entry,
                                   make_pair(std::ref(info),
                                            make_uniq<ActiveTableLock>()));
    }
    auto &active_table_lock = *entry->second;
    transaction_lock.unlock();

    lock_guard<mutex> table_lock(active_table_lock.checkpoint_lock_mutex);
    auto checkpoint_lock = active_table_lock.checkpoint_lock.lock();

    if (checkpoint_lock) {
        return checkpoint_lock;  // Reuse existing lock
    }

    // Obtain new lock
    checkpoint_lock = make_shared_ptr<CheckpointLock>(info.GetSharedLock());
    active_table_lock.checkpoint_lock = checkpoint_lock;
    return checkpoint_lock;
}
```

This ensures:
- Tables can't be checkpointed while being scanned
- Multiple scans can share the same lock (reference counted)
- Locks are released when scan completes

---

## Lock-Free Data Structures

DuckDB minimizes locking for read operations using atomic operations and lock-free techniques.

### Atomic Transaction State

**File:** `src/include/duckdb/transaction/transaction.hpp`

```cpp
class Transaction {
public:
    atomic<transaction_t> active_query;  // Current query number
    // ...
};
```

**File:** `src/include/duckdb/transaction/duck_transaction.hpp`

```cpp
class DuckTransaction : public Transaction {
public:
    atomic<idx_t> catalog_version;  // Catalog version
    // ...
};
```

**File:** `src/include/duckdb/transaction/duck_transaction_manager.hpp`

```cpp
class DuckTransactionManager {
private:
    atomic<transaction_t> lowest_active_id;
    atomic<transaction_t> lowest_active_start;
    atomic<transaction_t> last_commit;
    atomic<idx_t> last_uncommitted_catalog_version;
};
```

These atomics allow lock-free reads:

```cpp
transaction_t DuckTransactionManager::LowestActiveId() const {
    return lowest_active_id;  // Atomic read, no lock needed
}

transaction_t DuckTransactionManager::LowestActiveStart() const {
    return lowest_active_start;
}

transaction_t DuckTransactionManager::GetLastCommit() const {
    return last_commit;
}
```

### Atomic Version Numbers

**File:** `src/include/duckdb/transaction/update_info.hpp`

```cpp
struct UpdateInfo {
    atomic<transaction_t> version_number;
    // ...

    bool AppliesToTransaction(transaction_t start_time, transaction_t transaction_id) {
        // Lock-free read of version_number
        return version_number > start_time && version_number != transaction_id;
    }
};
```

The `version_number` is updated atomically during commit:

```cpp
// In CommitState::CommitEntry()
info->version_number = commit_id;  // Atomic store
```

This allows readers to check version visibility without locks.

### Lock-Free Chunk Info Access

**File:** `src/include/duckdb/storage/table/chunk_info.hpp`

Chunk info structures use atomic operations for delete markers:

```cpp
class ChunkVectorInfo : public ChunkInfo {
public:
    transaction_t inserted[STANDARD_VECTOR_SIZE];
    transaction_t deleted[STANDARD_VECTOR_SIZE];
    // ...
};
```

During delete:

```cpp
idx_t ChunkVectorInfo::Delete(transaction_t transaction_id, row_t rows[], idx_t count) {
    idx_t actual_delete_count = 0;
    for (idx_t i = 0; i < count; i++) {
        if (deleted[rows[i]] == NOT_DELETED_ID) {
            deleted[rows[i]] = transaction_id;
            rows[actual_delete_count++] = rows[i];
        }
    }
    any_deleted = true;
    return actual_delete_count;
}
```

During read:

```cpp
idx_t ChunkVectorInfo::GetSelVector(transaction_t start_time, transaction_t transaction_id,
                                   SelectionVector &sel_vector, idx_t max_count) const {
    idx_t count = 0;
    for (idx_t i = 0; i < max_count; i++) {
        if (UseInsertedVersion(start_time, transaction_id, inserted[i]) &&
            !UseDeletedVersion(start_time, transaction_id, deleted[i])) {
            sel_vector.set_index(count++, i);
        }
    }
    return count;
}
```

**Key Point:** Reads don't acquire locks. They perform lock-free visibility checks using transaction timestamps.

### Read-Write Transaction Separation

The transaction manager uses multiple locks for different purposes:

**File:** `src/include/duckdb/transaction/duck_transaction_manager.hpp`

```cpp
class DuckTransactionManager {
private:
    mutex transaction_lock;        // Protects transaction lists
    StorageLock checkpoint_lock;   // Coordinates checkpointing (read-write lock)
    mutex start_transaction_lock;  // Only for FORCE CHECKPOINT
    mutex wal_lock;                // Serializes WAL writes
    mutex cleanup_lock;            // One cleanup at a time
    mutex cleanup_queue_lock;      // Protects cleanup queue
};
```

**Concurrency Strategy:**

1. **Read-only transactions:**
   - Don't acquire `start_transaction_lock`
   - Don't hold checkpoint locks
   - Can start and commit while WAL writes happen

2. **Write transactions:**
   - Hold shared checkpoint lock during execution
   - Release `transaction_lock` during WAL write
   - Hold `wal_lock` briefly to serialize WAL writes

3. **Checkpointing:**
   - Acquires exclusive checkpoint lock
   - Blocks new write transactions (via `start_transaction_lock` if FORCE)
   - Read-only transactions can proceed

---

## Transaction Testing

DuckDB has extensive tests for transaction functionality.

### Test Locations

- `test/sql/transactions/` - SQL-based transaction tests
- `test/appender/test_appender_transactions.cpp` - Appender transaction tests
- Various `*_transaction*.test` files throughout the test suite

### Common Test Patterns

#### Basic Transaction Semantics

**File:** `test/sql/transactions/test_transaction_local_data.test`

Tests that transactions see their own writes:

```sql
statement ok
CREATE TABLE integers(i INTEGER)

statement ok
BEGIN TRANSACTION

statement ok
INSERT INTO integers VALUES (1), (2), (3)

query I
SELECT * FROM integers ORDER BY i
----
1
2
3

statement ok
ROLLBACK
```

#### Isolation Testing

**File:** `test/sql/transactions/test_multi_transaction_append.test`

Tests that concurrent transactions are isolated:

```sql
statement ok con1
BEGIN TRANSACTION

statement ok con2
BEGIN TRANSACTION

statement ok con1
INSERT INTO integers VALUES (1)

# con2 should not see con1's uncommitted insert
query I con2
SELECT COUNT(*) FROM integers
----
0

statement ok con1
COMMIT

# Now con2 should see it (in a new transaction)
statement ok con2
COMMIT

statement ok con2
BEGIN TRANSACTION

query I con2
SELECT COUNT(*) FROM integers
----
1
```

#### Conflict Detection

**File:** `test/sql/transactions/conflict_drop_then_delete.test`

Tests that conflicting operations are detected:

```sql
statement ok con1
BEGIN TRANSACTION

statement ok con2
BEGIN TRANSACTION

statement ok con1
DROP TABLE integers

statement ok con2
DELETE FROM integers WHERE i > 5

# con1 commits first
statement ok con1
COMMIT

# con2 should fail - table was dropped
statement error con2
COMMIT
----
Attempting to modify table integers but another transaction has dropped this table
```

#### Version Chain Testing

**File:** `test/sql/transactions/test_index_versioned_updates.test`

Tests that multiple updates create proper version chains:

```sql
statement ok con1
BEGIN TRANSACTION

statement ok con2
BEGIN TRANSACTION

statement ok con1
UPDATE integers SET i = i + 10 WHERE i = 1

statement ok con1
COMMIT

# con2 should still see old value (started before con1 committed)
query I con2
SELECT * FROM integers WHERE i = 1
----
1

statement ok con2
COMMIT

# New transaction should see new value
query I
SELECT * FROM integers WHERE i = 11
----
11
```

#### Cleanup Testing

Tests verify that version information is properly cleaned up after transactions end:

```sql
# Create transaction that updates data
statement ok
BEGIN TRANSACTION

statement ok
UPDATE integers SET i = i + 1

statement ok
COMMIT

# Create and commit another transaction
statement ok
BEGIN TRANSACTION

statement ok
UPDATE integers SET i = i + 1

statement ok
COMMIT

# At this point, the first transaction's version info should be cleanable
# (Verified internally by DuckDB's cleanup mechanism)
```

### Testing Checkpointing

**File:** `test/sql/checkpoint/test_checkpoint.test`

Tests automatic and manual checkpointing:

```sql
# Insert enough data to trigger automatic checkpoint
statement ok
INSERT INTO integers SELECT * FROM range(1000000)

statement ok
COMMIT

# Manual checkpoint
statement ok
CHECKPOINT

# Verify data is still correct
query I
SELECT COUNT(*) FROM integers
----
1000000
```

### Stress Testing

**File:** `test/sql/parallelism/interquery/concurrent_append_transactions.test_slow`

Tests many concurrent transactions:

```sql
loop i 0 100

statement ok con${i}
BEGIN TRANSACTION

statement ok con${i}
INSERT INTO integers VALUES (${i})

statement ok con${i}
COMMIT

endloop

# Verify all inserts succeeded
query I
SELECT COUNT(*) FROM integers
----
100
```

---

## Common Concurrency Patterns

### Pattern 1: Read Your Own Writes

**Implementation:** Local storage + visibility checks

When a transaction inserts data:

```cpp
// In DuckTransaction::PushAppend()
void DuckTransaction::PushAppend(DataTable &table, idx_t start_row, idx_t row_count) {
    ModifyTable(table);
    auto undo_entry = undo_buffer.CreateEntry(UndoFlags::INSERT_TUPLE, sizeof(AppendInfo));
    auto append_info = reinterpret_cast<AppendInfo *>(undo_entry.Ptr());
    append_info->table = &table;
    append_info->start_row = start_row;
    append_info->count = row_count;
}
```

The data goes into `LocalStorage`, which is scanned during queries:

```cpp
// Table scan combines committed and local data
void DataTable::Scan(Transaction &transaction, DataChunk &result) {
    // Scan committed data
    ScanCommitted(transaction, result);

    // Scan transaction-local data
    auto &local_storage = LocalStorage::Get(transaction);
    local_storage.Scan(result);
}
```

### Pattern 2: Optimistic Locking

**Implementation:** No locks during execution, validate at commit

Transactions execute without acquiring locks:

```cpp
// No locks acquired here
statement ok
UPDATE integers SET i = i + 1 WHERE i = 5
```

At commit time, conflicts are detected:

```cpp
// In CommitState::CommitEntry()
if (!info->table->IsMainTable()) {
    throw TransactionException("Table was modified by another transaction");
}
```

### Pattern 3: Version Chains for Updates

**Implementation:** Linked list of update versions

Each update creates a new version:

```cpp
// In DuckTransaction::CreateUpdateInfo()
UndoBufferReference DuckTransaction::CreateUpdateInfo(idx_t type_size,
                                                      DataTable &data_table,
                                                      idx_t entries) {
    idx_t alloc_size = UpdateInfo::GetAllocSize(type_size);
    auto undo_entry = undo_buffer.CreateEntry(UndoFlags::UPDATE_TUPLE, alloc_size);
    auto &update_info = UpdateInfo::Get(undo_entry);
    UpdateInfo::Initialize(update_info, data_table, transaction_id);
    return undo_entry;
}
```

Updates are linked:

```cpp
struct UpdateInfo {
    UndoBufferPointer prev;  // Previous version
    UndoBufferPointer next;  // Next version
    // ...
};
```

When reading, walk the chain to find the right version:

```cpp
template <class T>
static void UpdatesForTransaction(UpdateInfo &current,
                                  transaction_t start_time,
                                  transaction_t transaction_id,
                                  T &&callback) {
    // Check current version
    if (current.AppliesToTransaction(start_time, transaction_id)) {
        callback(current);
    }

    // Walk the chain
    auto update_ptr = current.next;
    while (update_ptr.IsSet()) {
        auto pin = update_ptr.Pin();
        auto &info = Get(pin);
        if (info.AppliesToTransaction(start_time, transaction_id)) {
            callback(info);
        }
        update_ptr = info.next;
    }
}
```

### Pattern 4: Garbage Collection

**Implementation:** Three-stage lifecycle

Transactions move through stages:

1. **Active:** In `active_transactions` list
2. **Recently Committed:** In `recently_committed_transactions` list
3. **Old:** In `old_transactions` list
4. **Cleanup Queue:** In `cleanup_queue`

The transition happens based on visibility:

```cpp
// A committed transaction stays in recently_committed until:
if (transaction->commit_id < lowest_start_time) {
    // No active transaction needs this version info
    old_transactions.push_back(std::move(transaction));
}

// A transaction in old_transactions stays there until:
if (transaction->highest_active_query < lowest_active_query) {
    // No active query is scanning this data
    cleanup_info->transactions.push_back(std::move(transaction));
}
```

### Pattern 5: Checkpoint Coordination

**Implementation:** Upgradeable read-write lock

Write transactions hold shared lock:

```cpp
void DuckTransaction::SetReadWrite() {
    write_lock = transaction_manager.SharedCheckpointLock();
}
```

At commit, try to upgrade:

```cpp
lock = transaction.TryGetCheckpointLock();  // Try to upgrade to exclusive
if (lock && can_checkpoint) {
    storage_manager.CreateCheckpoint(context, options);
}
```

If upgrade fails, write to WAL instead:

```cpp
if (!checkpoint_decision.can_checkpoint && transaction.ShouldWriteToWAL(db)) {
    held_wal_lock = make_uniq<lock_guard<mutex>>(wal_lock);
    error = transaction.WriteToWAL(db, commit_state);
}
```

### Pattern 6: Sequence Usage Tracking

**Implementation:** Per-transaction sequence cache

Sequences are tracked in the transaction:

```cpp
void DuckTransaction::PushSequenceUsage(SequenceCatalogEntry &sequence,
                                       const SequenceData &data) {
    lock_guard<mutex> l(sequence_lock);
    auto entry = sequence_usage.find(sequence);
    if (entry == sequence_usage.end()) {
        auto undo_entry = undo_buffer.CreateEntry(UndoFlags::SEQUENCE_VALUE,
                                                  sizeof(SequenceValue));
        auto sequence_info = reinterpret_cast<SequenceValue *>(undo_entry.Ptr());
        sequence_info->entry = &sequence;
        sequence_info->usage_count = data.usage_count;
        sequence_info->counter = data.counter;
        sequence_usage.emplace(sequence, *sequence_info);
    } else {
        // Update existing entry
        auto &sequence_info = entry->second.get();
        sequence_info.usage_count = data.usage_count;
        sequence_info.counter = data.counter;
    }
}
```

On rollback, sequence values are restored:

```cpp
// In cleanup, the sequence is reset to its pre-transaction state
```

This ensures:
- Sequence values are transaction-isolated
- Rollback doesn't create gaps in sequences (within a transaction)
- Committed values are properly persisted

---

## Summary

DuckDB's transaction system is a sophisticated implementation of MVCC that provides:

1. **Serializability:** The strongest isolation level, preventing all anomalies
2. **High Concurrency:** Lock-free reads, optimistic writes
3. **Efficiency:** Deferred cleanup, automatic checkpointing
4. **Reliability:** WAL logging, atomic commit/rollback
5. **Simplicity:** Single isolation level, predictable behavior

Key architectural decisions:

- **Timestamp-based MVCC:** Simple visibility rules
- **Undo buffer:** Efficient rollback and cleanup
- **Local storage:** Fast transaction-local operations
- **Version chains:** Handle concurrent updates
- **Deferred cleanup:** Minimize critical section time
- **Lock-free reads:** No blocking between readers and writers

The implementation balances performance, correctness, and maintainability, making it suitable for both OLAP workloads (large scans) and OLTP workloads (many small transactions).

---

## Related Source Files

### Core Transaction Files
- `src/transaction/transaction.cpp` - Base transaction class
- `src/transaction/duck_transaction.cpp` - Main transaction implementation
- `src/transaction/duck_transaction_manager.cpp` - Transaction manager
- `src/transaction/undo_buffer.cpp` - Undo buffer implementation
- `src/transaction/local_storage.cpp` - Transaction-local storage
- `src/transaction/commit_state.cpp` - Commit logic
- `src/transaction/rollback_state.cpp` - Rollback logic
- `src/transaction/cleanup_state.cpp` - Cleanup logic

### Version Management Files
- `src/storage/table/chunk_info.cpp` - Version information per chunk
- `src/storage/table/row_version_manager.cpp` - Per-row-group version tracking
- `src/storage/table/update_segment.cpp` - Update version chains

### Locking and Coordination
- `src/storage/storage_lock.cpp` - Storage-level locks
- `src/storage/checkpoint_manager.cpp` - Checkpoint coordination

### Tests
- `test/sql/transactions/` - SQL-based transaction tests
- `test/sql/parallelism/interquery/` - Concurrent transaction tests

---

**Note:** This document reflects the DuckDB transaction implementation as of the current codebase. Implementation details may change as the system evolves.
