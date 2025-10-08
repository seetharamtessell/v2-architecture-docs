# Storage Service Architecture

**Crate**: `cloudops-storage-service`
**Language**: Pure Rust
**Status**: Design Complete ✅

## Overview

The Storage Service is a pure Rust crate that provides a unified storage layer for the CloudOps client application. It handles both chat history and AWS estate data using Qdrant as the underlying vector database, with application-level encryption and automated S3 backup.

## Design Goals

1. **Unified Storage**: Single abstraction for all local data (chat + estate)
2. **Privacy-First**: Application-level encryption, data never leaves in plain text
3. **Performance**: Optimized for different access patterns (filtering vs semantic search)
4. **Reliability**: Automatic backups, crash recovery, data integrity
5. **Simplicity**: Clean API, minimal configuration, automatic setup

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│           Tauri Rust Backend                            │
│  ┌───────────────────────────────────────────────────┐  │
│  │     cloudops-storage-service (This Crate)         │  │
│  │                                                    │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  StorageService                             │  │  │
│  │  │  ├─ ChatStorage                             │  │  │
│  │  │  ├─ EstateStorage                           │  │  │
│  │  │  └─ BackupManager                           │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  │                      ↓                             │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  Encryptor (AES-256-GCM)                    │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  │                      ↓                             │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  Qdrant Client                              │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│        Qdrant (Embedded, Filesystem)                    │
│  Collections:                                           │
│  ├─ chat_history (vector_size: 1, dummy vectors)       │
│  └─ aws_estate (vector_size: 384, real embeddings)     │
│                                                         │
│  Storage: ~/.cloudops/data/qdrant/                     │
│  WAL: Enabled                                          │
│  Mode: Memmap (disk-backed)                            │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│        Local Snapshots                                  │
│  Path: ~/.cloudops/snapshots/                          │
│  Format: {collection}_{timestamp}.snapshot              │
│  Retention: 7 days                                      │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│        S3 Backup (Optional)                             │
│  Bucket: s3://backup-bucket/{user_id}/snapshots/       │
│  Schedule: Every N hours (configurable)                 │
│  Retention: 30 days (configurable)                      │
└─────────────────────────────────────────────────────────┘
```

## Core Components

### 1. StorageService

**Entry point** for all storage operations. Initializes and manages all sub-components.

**Responsibilities**:
- Initialize Qdrant client (embedded or remote mode)
- Create collections on first run
- Coordinate between ChatStorage, EstateStorage, and BackupManager
- Health monitoring
- Graceful shutdown

**Lifecycle**:
```rust
StorageService::new(config)
    → Initialize Qdrant client
    → Create collections if not exist
    → Initialize encryption
    → Start auto-backup (if enabled)
    → Return ready service
```

### 2. ChatStorage

**Purpose**: Manage conversation history with minimal overhead.

**Strategy**:
- Use dummy vectors (1 dimension, 4 bytes)
- Filter-based retrieval (no vector search)
- UUID-based point IDs (immutable messages)

**Operations**:
- `append()` - Add message to conversation (no embedding)
- `get_history()` - Retrieve ordered messages (filter by context_id)
- `delete()` - Remove entire conversation
- `get_all_contexts()` - List all conversations

**Access Pattern**: Filter by `context_id` + order by `message_index`

### 3. EstateStorage

**Purpose**: Store AWS resources with semantic search capability.

**Strategy**:
- Real vector embeddings (384 dimensions)
- Semantic search + filter combination
- Deterministic point IDs (prevents duplicates)

**Operations**:
- `upsert_resource()` - Add/update resource (generates embedding)
- `bulk_upsert()` - Batch insert (efficient)
- `search()` - Semantic search + filters
- `get_by_account/region/type()` - Filter-only queries
- `sync_from_aws()` - Sync strategy (upsert + cleanup)

**Access Pattern**: Vector search with filters (resource_type, region, account, state, tags)

### 4. BackupManager

**Purpose**: Automated backup and restore workflows.

**Strategy**:
- Background tokio task for auto-backup
- Create Qdrant snapshots (tar archives)
- Upload to S3 with retention policies
- Support restore from local or S3

**Operations**:
- `create_snapshot()` - Manual snapshot
- `start_auto_backup()` - Start background scheduler
- `restore_from_s3()` - Restore from S3
- `list_snapshots()` - List available backups
- `cleanup_old_snapshots()` - Enforce retention

**Schedule**: Configurable interval (default: 24 hours)

### 5. Encryptor

**Purpose**: Application-level encryption for sensitive data.

**Strategy**:
- AES-256-GCM (authenticated encryption)
- Key stored in OS Keychain (auto-managed)
- Encrypt payloads before storing in Qdrant
- Vectors and filter fields remain unencrypted

**Operations**:
- `new()` - Get/create key from OS keychain
- `encrypt()` - Encrypt string to base64
- `decrypt()` - Decrypt base64 to string

**Key Management**: Automatic via OS Keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service)

## Data Flow

### Chat Message Flow

```
User sends message
    ↓
Frontend → Tauri Command
    ↓
ChatStorage.append(context_id, message)
    ↓
1. Get message count (for index)
2. Encrypt message content
3. Create Point with dummy vector [0.0]
4. Upsert to Qdrant (collection: chat_history)
    ↓
Qdrant stores point (encrypted payload + dummy vector)
```

### Retrieve Chat History

```
Frontend requests history
    ↓
ChatStorage.get_history(context_id)
    ↓
1. Scroll Qdrant (filter by context_id)
2. Order by message_index
3. Decrypt each message
4. Return ordered list
    ↓
Frontend displays messages
```

### AWS Resource Flow

```
Estate Scanner fetches resource
    ↓
EstateStorage.upsert_resource(resource)
    ↓
1. Generate deterministic ID (account-region-resource_id)
2. Generate embedding (via configured provider)
3. Encrypt sensitive metadata
4. Create Point with real vector
5. Upsert to Qdrant (collection: aws_estate)
    ↓
Qdrant stores/updates point (upsert is idempotent)
```

### Search Resources

```
User searches "postgres database"
    ↓
EstateStorage.search(query, filters)
    ↓
1. Generate query embedding
2. Build filter conditions (region, account, type, etc.)
3. Search Qdrant (vector + filters)
4. Decrypt results
5. Return matched resources
    ↓
Frontend displays results
```

### Backup Flow

```
Auto-backup scheduler triggers
    ↓
BackupManager.create_snapshot(collection)
    ↓
1. Qdrant creates snapshot (tar file)
2. Save to ~/.cloudops/snapshots/
3. Upload to S3 (if configured)
4. Cleanup old local snapshots (>7 days)
5. Cleanup old S3 snapshots (>retention_days)
    ↓
Backup complete (non-blocking)
```

## Collection Strategy

### Why Two Collections?

**Different Access Patterns**:

| Aspect | Chat History | AWS Estate |
|--------|-------------|------------|
| Search | Filter-only | Vector + Filter |
| Vectors | Dummy (1D) | Real (384D) |
| Point ID | Random UUID | Deterministic |
| Updates | Never | Frequent |
| Embedding | None | Required |
| Use Case | Ordered messages | Semantic search |

### Collection Independence

- Each collection can be backed up/restored independently
- Different sync schedules possible
- Failures in one don't affect the other
- Can scale differently (chat is tiny, estate can be large)

## Qdrant Mode: Embedded vs Remote

### Embedded Mode (Default, Recommended)

```rust
QdrantConfig {
    mode: QdrantMode::Embedded,
    storage_path: Some("~/.cloudops/data/qdrant/"),
    url: None,
}
```

**Benefits**:
- No separate Qdrant server needed
- Smaller app footprint
- Simpler deployment
- Direct filesystem access

**Use When**: Single-user desktop app (our case)

### Remote Mode (Optional)

```rust
QdrantConfig {
    mode: QdrantMode::Remote,
    storage_path: None,
    url: Some("http://localhost:6334"),
}
```

**Benefits**:
- Can run Qdrant in Docker
- Easier debugging (Qdrant dashboard)
- Can scale horizontally

**Use When**: Development, testing, or advanced users

## Storage Layout

```
~/.cloudops/
├── data/
│   ├── qdrant/                    # Qdrant embedded storage
│   │   ├── collections/
│   │   │   ├── chat_history/
│   │   │   │   ├── storage/       # Point data
│   │   │   │   ├── wal/           # Write-ahead log
│   │   │   │   └── payload_index/ # Indexed fields
│   │   │   └── aws_estate/
│   │   │       ├── storage/
│   │   │       ├── wal/
│   │   │       └── payload_index/
│   │   └── meta.json              # Qdrant metadata
│   └── temp/                      # Temp files (restore, etc.)
└── snapshots/                     # Local snapshots
    ├── chat_history_2025-10-08.snapshot
    └── aws_estate_2025-10-08.snapshot
```

**S3 Layout**:
```
s3://backup-bucket/
└── {user_id}/
    └── snapshots/
        ├── chat_history/
        │   ├── 2025-10-08.snapshot
        │   └── 2025-10-07.snapshot
        └── aws_estate/
            ├── 2025-10-08.snapshot
            └── 2025-10-07.snapshot
```

## Performance Characteristics

### Chat Operations

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Append | O(1) | No embedding, just insert |
| Get history | O(n) | Filter + sort, n = message count |
| Delete conversation | O(n) | Delete all messages in conversation |

**Typical Performance**:
- Append: <1ms
- Get 100 messages: <10ms

### Estate Operations

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Upsert | O(log n) | Embedding + HNSW index update |
| Search | O(k log n) | k = result limit, n = total points |
| Filter-only | O(m) | m = matching points |

**Typical Performance**:
- Upsert: 10-50ms (embedding dominates)
- Search 10k resources: 10-50ms
- Filter-only: <10ms

### Scalability

**Chat History**:
- 1,000 conversations × 100 messages = 100,000 points
- Storage: ~50-100 MB
- Performance: No degradation

**AWS Estate**:
- 10,000 resources
- Storage: ~100-200 MB (with vectors)
- Performance: <50ms search

## Error Handling

**Categories**:

1. **Initialization Errors**: Qdrant not available, collection creation fails
2. **Storage Errors**: Disk full, Qdrant down, write failures
3. **Encryption Errors**: Key not found, decryption fails
4. **Backup Errors**: S3 unavailable, snapshot creation fails

**Strategy**:
- Return `Result<T, StorageError>`
- Log errors with context
- Retry transient failures (network, throttling)
- Fail fast on permanent errors (disk full, invalid config)

## Dependencies

```toml
[dependencies]
# Qdrant
qdrant-client = "1.x"

# Encryption
aes-gcm = "0.10"
keyring = "2.0"

# AWS
aws-sdk-s3 = "1.x"

# Async
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Utilities
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
anyhow = "1"
thiserror = "1"
```

## Related Documentation

- [API Reference](api.md) - Complete Rust API
- [Configuration](configuration.md) - Config structure and options
- [Collections](collections.md) - Collection schemas and point structure
- [Encryption](encryption.md) - Encryption strategy and key management
- [Backup & Restore](backup-restore.md) - Backup workflows
- [Point Management](point-management.md) - Point ID strategies
- [Operations](operations.md) - Common operations with examples
- [Testing](testing.md) - Testing strategy