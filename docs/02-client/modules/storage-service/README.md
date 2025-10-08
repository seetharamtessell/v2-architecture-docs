# Storage Service Module

**Crate Name**: `cloudops-storage-service`
**Status**: Design Complete ✅ | Ready for Implementation

## Overview

The Storage Service is a Pure Rust crate that handles all local storage operations for the CloudOps client. It uses a single Qdrant instance (embedded mode) for both chat history and AWS estate data, with application-level encryption and automated S3 backup.

## Responsibilities

### 1. Estate Storage with RAG + IAM
- Store AWS resources in Qdrant with vector embeddings
- Semantic search over resources ("postgres database" → RDS instances)
- **IAM permissions embedded** with each resource (know what actions you can perform)
- Filter by account, region, service, type, tags
- Deterministic point IDs prevent duplicates

### 2. Context & Chat Storage
- Store conversation history with encryption
- Filter-based access (no vector search needed)
- Dummy vectors (1D) for minimal overhead
- Fast retrieval by context_id
- Sequential message ordering

### 3. Backup Management
- Create Qdrant snapshots (local + S3)
- Automated background scheduler
- Retention policies (7 days local, 30 days S3)
- Restore from local or S3 snapshots
- Complete disaster recovery workflow

## Key Features

✅ **Single Qdrant Instance** - Embedded mode, ~20-30 MB
✅ **Dual Collection Strategy** - Chat (dummy vectors) + Estate (real embeddings)
✅ **Application-Level Encryption** - AES-256-GCM + OS Keychain
✅ **IAM Integration** - Permissions per resource
✅ **Point ID Management** - Prevent duplicates, enable idempotent sync
✅ **Auto S3 Backup** - Background task with configurable schedule

## Documentation

### Architecture & Design
- **[architecture.md](architecture.md)** - Overall design, components, data flow, storage layout
- **[collections.md](collections.md)** - Collection schemas, point structures, indexed fields, IAM embedding

### Implementation Reference
- **[api.md](api.md)** - Complete Rust API with signatures and examples
- **[configuration.md](configuration.md)** - StorageConfig, QdrantConfig, EmbeddingConfig, etc.
- **[operations.md](operations.md)** - Common operations with working code examples

### Technical Details
- **[encryption.md](encryption.md)** - AES-256-GCM implementation, OS Keychain integration
- **[backup-restore.md](backup-restore.md)** - Backup workflows, S3 sync, disaster recovery
- **[point-management.md](point-management.md)** - Point ID strategies, sync patterns, avoiding duplicates

### Development
- **[testing.md](testing.md)** - Unit, integration, and performance testing strategies

## Quick Examples

### Initialize Storage
```rust
use cloudops_storage_service::{StorageService, StorageConfig};

let config = StorageConfig {
    data_path: PathBuf::from("~/.cloudops/data/"),
    qdrant: QdrantConfig {
        mode: QdrantMode::Embedded,
        storage_path: Some(PathBuf::from("~/.cloudops/data/qdrant/")),
        url: None,
    },
    // ... other config
};

let storage = StorageService::new(config).await?;
```

### Store & Retrieve Chat
```rust
// Append message
storage.chat.append(context_id, Message {
    role: MessageRole::User,
    content: "Stop pg-instance-main1".to_string(),
    metadata: None,
}).await?;

// Get history
let history = storage.chat.get_history(context_id, Some(10)).await?;
```

### Search AWS Resources
```rust
// Semantic search with filters + get IAM permissions
let results = storage.estate.search(
    "postgres database",
    Some(ResourceFilter {
        resource_type: Some("rds_instance".to_string()),
        region: Some("us-west-2".to_string()),
        ..Default::default()
    })
).await?;

for resource in results {
    println!("Found: {}", resource.name);
    println!("  Allowed actions: {:?}", resource.iam.allowed_actions);
    println!("  Can stop: {}", resource.constraints.can_stop);
}
```

### Backup to S3
```rust
// Start auto-backup (runs in background)
storage.backup.start_auto_backup().await?;

// Manual snapshot
let snapshot = storage.backup.create_snapshot("aws_estate").await?;
storage.backup.upload_to_s3(&snapshot).await?;

// Restore from S3
storage.backup.restore_from_s3("aws_estate", "2025-10-08").await?;
```

## Design Highlights

### Chat Collection
- **Vector**: 1 dimension (dummy, 4 bytes)
- **Access**: Scroll API with filters (no vector search)
- **Point ID**: Random UUID (immutable messages)
- **Indexed**: context_id, message_index, timestamp

### Estate Collection
- **Vector**: 384 dimensions (real embeddings)
- **Access**: Vector search + filters
- **Point ID**: `{account}-{region}-{resource_id}` (deterministic)
- **Indexed**: resource_type, account_id, region, service, state, tags
- **IAM**: Embedded with each resource

## Implementation Status

| Component | Status |
|-----------|--------|
| Documentation | ✅ Complete (9 files, ~170 KB) |
| API Design | ✅ Complete |
| IAM Integration | ✅ Designed |
| Implementation | ⏳ Pending |
| Testing | ⏳ Pending |

## Next Steps

1. Implement Rust crate based on design docs
2. Unit tests (encryption, point IDs, sync logic)
3. Integration tests (Qdrant operations)
4. Performance tests (10k resources, concurrent access)
5. Estate Scanner integration (IAM permission discovery)

## Related Modules

- [Execution Engine](../execution-engine/README.md) - Uses IAM permissions to validate operations
- [Estate Scanner](../estate-scanner/README.md) - Populates estate storage with resources + IAM
- [Request Builder](../request-builder/README.md) - Queries storage for context enrichment