# Storage Service API Reference

Complete Rust API reference for the `cloudops-storage-service` crate.

## Overview

The Storage Service provides a clean, type-safe Rust API for all local storage operations. It abstracts Qdrant vector database operations, encryption, and backup management into a single, cohesive interface.

**Crate**: `cloudops-storage-service`

---

> **📦 Common Types**: This module uses shared types from [`cloudops-common`](../common/). Key types like `AWSResource`, `IAMPermissions`, `UserContext`, and `ResourceConstraints` are defined in the common crate and referenced here.
>
> See: [Common Types Documentation](../common/README.md)

---

## Main Entry Point

### StorageService

The main storage service struct that provides access to all storage operations.

```rust
pub struct StorageService {
    pub chat: ChatStorage,
    pub estate: EstateStorage,
    pub backup: BackupManager,
}
```

#### Methods

```rust
impl StorageService {
    /// Initialize storage service with configuration
    ///
    /// This will:
    /// - Initialize Qdrant client (embedded or remote mode)
    /// - Create collections if they don't exist
    /// - Set up encryption
    /// - Initialize backup manager
    pub async fn new(config: StorageConfig) -> Result<Self>;

    /// Health check for all storage components
    ///
    /// Returns status of:
    /// - Qdrant connection
    /// - S3 configuration (if enabled)
    /// - Disk space
    /// - Collection health (point counts, status)
    pub async fn health(&self) -> Result<StorageHealth>;

    /// Shutdown gracefully
    ///
    /// This will:
    /// - Stop background tasks (backup scheduler)
    /// - Flush pending operations
    /// - Close Qdrant connection
    pub async fn shutdown(&self) -> Result<()>;
}
```

#### Example

```rust
use cloudops_storage_service::{StorageService, StorageConfig, QdrantConfig, QdrantMode};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
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

    // Use storage service
    let health = storage.health().await?;
    println!("Storage health: {:?}", health);

    // Shutdown when done
    storage.shutdown().await?;
    Ok(())
}
```

## Chat Storage API

### ChatStorage

Manages conversation history with encryption and fast retrieval.

```rust
pub struct ChatStorage {
    // Internal fields...
}
```

#### Methods

```rust
impl ChatStorage {
    /// Append a message to a conversation
    ///
    /// Messages are immutable once created. Each message gets:
    /// - Unique UUID as point ID
    /// - Encrypted content
    /// - Indexed metadata (context_id, role, timestamp)
    /// - Sequential message_index for ordering
    pub async fn append(
        &self,
        context_id: &str,
        message: Message
    ) -> Result<()>;

    /// Get conversation history, ordered by message_index
    ///
    /// Uses scroll API (not vector search) for pure filtering.
    /// Limit can be used to get recent N messages.
    pub async fn get_history(
        &self,
        context_id: &str,
        limit: Option<usize>
    ) -> Result<Vec<Message>>;

    /// Delete entire conversation
    ///
    /// Removes all messages with matching context_id.
    pub async fn delete(&self, context_id: &str) -> Result<()>;

    /// Update conversation metadata
    ///
    /// Note: This updates conversation-level metadata,
    /// not individual messages (which are immutable).
    pub async fn update_context(
        &self,
        context_id: &str,
        metadata: serde_json::Value
    ) -> Result<()>;

    /// Search across all conversations
    ///
    /// Searches encrypted message content after decryption.
    /// Returns matching conversations with snippets.
    pub async fn search_conversations(
        &self,
        query: &str
    ) -> Result<Vec<Conversation>>;

    /// List all conversation IDs
    ///
    /// Useful for building conversation list UI.
    pub async fn get_all_contexts(&self) -> Result<Vec<String>>;
}
```

#### Data Types

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Message {
    pub role: MessageRole,
    pub content: String,
    pub metadata: Option<serde_json::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum MessageRole {
    User,
    Assistant,
    System,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Conversation {
    pub context_id: String,
    pub messages: Vec<Message>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub metadata: Option<serde_json::Value>,
}
```

#### Example

```rust
use cloudops_storage_service::{Message, MessageRole};

// Create a new conversation
let context_id = uuid::Uuid::new_v4().to_string();

// Add user message
storage.chat.append(&context_id, Message {
    role: MessageRole::User,
    content: "Stop pg-instance-main1".to_string(),
    metadata: None,
}).await?;

// Add assistant response
storage.chat.append(&context_id, Message {
    role: MessageRole::Assistant,
    content: "I'll stop the RDS instance...".to_string(),
    metadata: Some(json!({
        "resources_mentioned": ["rds-pg-instance-main1"]
    })),
}).await?;

// Retrieve history
let history = storage.chat.get_history(&context_id, Some(10)).await?;
for msg in history {
    println!("{:?}: {}", msg.role, msg.content);
}
```

## Estate Storage API

### EstateStorage

Manages AWS resource data with vector search and filtering capabilities.

```rust
pub struct EstateStorage {
    // Internal fields...
}
```

#### Methods

```rust
impl EstateStorage {
    /// Upsert (insert or update) a single resource
    ///
    /// Uses deterministic point ID based on resource identifier.
    /// If resource exists, it will be updated. If not, created.
    /// This prevents duplicate entries for the same resource.
    pub async fn upsert_resource(
        &self,
        resource: AWSResource
    ) -> Result<()>;

    /// Bulk upsert multiple resources
    ///
    /// More efficient than individual upserts.
    /// Use this for syncing large batches of resources.
    pub async fn bulk_upsert(
        &self,
        resources: Vec<AWSResource>
    ) -> Result<()>;

    /// Search resources with vector similarity + filters
    ///
    /// Query is embedded and used for semantic search.
    /// Filters narrow down results by metadata.
    pub async fn search(
        &self,
        query: &str,
        filters: Option<ResourceFilter>
    ) -> Result<Vec<AWSResource>>;

    /// Get specific resource by ID
    ///
    /// Returns None if resource doesn't exist.
    pub async fn get_resource(
        &self,
        resource_id: &str
    ) -> Result<Option<AWSResource>>;

    /// Delete resource
    ///
    /// Idempotent - no error if resource doesn't exist.
    pub async fn delete_resource(
        &self,
        resource_id: &str
    ) -> Result<()>;

    /// Get all resources in an AWS account
    pub async fn get_by_account(
        &self,
        account_id: &str
    ) -> Result<Vec<AWSResource>>;

    /// Get all resources in a region
    pub async fn get_by_region(
        &self,
        region: &str
    ) -> Result<Vec<AWSResource>>;

    /// Get all resources of a specific type
    pub async fn get_by_type(
        &self,
        resource_type: &str
    ) -> Result<Vec<AWSResource>>;

    /// Count total resources
    pub async fn count(&self) -> Result<usize>;
}
```

#### Data Types

> **📦 Note**: The following types are defined in [`cloudops-common`](../common/README.md) and re-exported here for convenience. See the common types documentation for complete details.

```rust
use cloudops_common::aws::{AWSResource, IAMPermissions, UserContext, ResourceConstraints};

// AWSResource - Complete AWS resource representation
// See: https://docs/04-services/common/README.md#awsresource

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AWSResource {
    pub resource_type: String,
    pub identifier: String,
    pub account_id: String,
    pub account_name: String,
    pub region: String,
    pub service: String,
    pub arn: String,
    pub name: String,
    pub state: String,
    pub tags: HashMap<String, String>,
    pub iam: IAMPermissions,
    pub constraints: ResourceConstraints,
    pub metadata: serde_json::Value,
    pub last_synced: DateTime<Utc>,
}

// IAMPermissions - Per-resource IAM permissions
// See: https://docs/04-services/common/README.md#iampermissions

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IAMPermissions {
    /// Actions allowed on this specific resource
    pub allowed_actions: Vec<String>,

    /// Actions explicitly denied on this resource
    pub denied_actions: Vec<String>,

    /// User/role context for these permissions
    pub user_context: UserContext,
}

// UserContext - AWS user/role context
// See: https://docs/04-services/common/README.md#usercontext

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserContext {
    /// IAM username
    pub username: String,

    /// Role ARN if using assumed role
    pub role_arn: Option<String>,

    /// Session expiration (for temporary credentials)
    pub session_expires: Option<DateTime<Utc>>,
}

// ResourceConstraints - Operational constraints
// See: https://docs/04-services/common/README.md#resourceconstraints

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceConstraints {
    pub can_stop: bool,
    pub can_delete: bool,
    pub has_dependencies: bool,
    pub metadata: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Clone, Default)]
pub struct ResourceFilter {
    pub resource_type: Option<String>,
    pub account_id: Option<String>,
    pub region: Option<String>,
    pub state: Option<String>,
    pub tags: Option<HashMap<String, String>>,
}
```

#### Example

```rust
use cloudops_storage_service::{AWSResource, ResourceFilter};

// Upsert a resource with IAM permissions
let resource = AWSResource {
    resource_type: "rds_instance".to_string(),
    identifier: "pg-instance-main1".to_string(),
    account_id: "123456789012".to_string(),
    account_name: "production-account".to_string(),
    region: "us-west-2".to_string(),
    service: "rds".to_string(),
    arn: "arn:aws:rds:us-west-2:123456789012:db:pg-instance-main1".to_string(),
    name: "pg-instance-main1".to_string(),
    state: "available".to_string(),
    tags: [("env".to_string(), "production".to_string())].into(),
    iam: IAMPermissions {
        allowed_actions: vec![
            "rds:StopDBInstance".to_string(),
            "rds:StartDBInstance".to_string(),
            "rds:DescribeDBInstances".to_string(),
        ],
        denied_actions: vec!["rds:DeleteDBInstance".to_string()],
        user_context: UserContext {
            username: "john.doe".to_string(),
            role_arn: Some("arn:aws:iam::123456789012:role/DeveloperRole".to_string()),
            session_expires: None,
        },
    },
    constraints: ResourceConstraints {
        can_stop: true,
        can_delete: false,
        has_dependencies: false,
        metadata: HashMap::new(),
    },
    metadata: json!({
        "instance_class": "db.t3.medium",
        "engine": "postgres",
    }),
    last_synced: Utc::now(),
};

storage.estate.upsert_resource(resource).await?;

// Search with filters
let results = storage.estate.search(
    "postgres database",
    Some(ResourceFilter {
        resource_type: Some("rds_instance".to_string()),
        region: Some("us-west-2".to_string()),
        state: Some("available".to_string()),
        ..Default::default()
    })
).await?;

for resource in results {
    println!("Found: {} ({})", resource.name, resource.state);
    println!("  Permissions: {:?}", resource.iam.allowed_actions);
    println!("  Can stop: {}", resource.constraints.can_stop);
}
```

## Backup Manager API

### BackupManager

Handles snapshots and S3 synchronization.

```rust
pub struct BackupManager {
    // Internal fields...
}
```

#### Methods

```rust
impl BackupManager {
    /// Create snapshot of a collection
    ///
    /// Returns snapshot info with local path.
    pub async fn create_snapshot(
        &self,
        collection: &str
    ) -> Result<SnapshotInfo>;

    /// Upload snapshot to S3
    ///
    /// Snapshot must exist locally.
    pub async fn upload_to_s3(
        &self,
        snapshot: &SnapshotInfo
    ) -> Result<()>;

    /// Restore from local snapshot
    ///
    /// This will:
    /// - Delete existing collection
    /// - Restore from snapshot file
    pub async fn restore_from_snapshot(
        &self,
        collection: &str,
        snapshot_name: &str
    ) -> Result<()>;

    /// Restore from S3 snapshot
    ///
    /// This will:
    /// - Download snapshot from S3
    /// - Delete existing collection
    /// - Restore from downloaded snapshot
    /// - Clean up temp file
    pub async fn restore_from_s3(
        &self,
        collection: &str,
        date: &str
    ) -> Result<()>;

    /// List available snapshots (local + S3)
    pub async fn list_snapshots(
        &self,
        collection: &str
    ) -> Result<Vec<SnapshotInfo>>;

    /// Start automatic backup scheduler
    ///
    /// Spawns a background task that:
    /// - Creates snapshots periodically
    /// - Uploads to S3 (if configured)
    /// - Cleans up old snapshots
    pub async fn start_auto_backup(&self) -> Result<()>;

    /// Stop automatic backup scheduler
    pub async fn stop_auto_backup(&self) -> Result<()>;

    /// Manual cleanup of old snapshots
    ///
    /// Removes:
    /// - Local snapshots older than 7 days
    /// - S3 snapshots older than retention_days
    pub async fn cleanup_old_snapshots(&self) -> Result<()>;
}
```

#### Data Types

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnapshotInfo {
    pub name: String,
    pub collection: String,
    pub created_at: DateTime<Utc>,
    pub size_bytes: u64,
    pub location: SnapshotLocation,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SnapshotLocation {
    Local(PathBuf),
    S3 { bucket: String, key: String },
}
```

#### Example

```rust
// Create snapshot
let snapshot = storage.backup.create_snapshot("cloud_estate").await?;
println!("Created snapshot: {}", snapshot.name);

// Upload to S3 (if configured)
storage.backup.upload_to_s3(&snapshot).await?;

// List available snapshots
let snapshots = storage.backup.list_snapshots("cloud_estate").await?;
for snap in snapshots {
    println!("{}: {} bytes", snap.name, snap.size_bytes);
}

// Restore from S3
storage.backup.restore_from_s3("cloud_estate", "2025-10-08").await?;

// Start automatic backups
storage.backup.start_auto_backup().await?;
```

## Health Status API

### StorageHealth

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct StorageHealth {
    pub qdrant_connected: bool,
    pub s3_configured: bool,
    pub disk_space_mb: u64,
    pub collections: Vec<CollectionHealth>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct CollectionHealth {
    pub name: String,
    pub points_count: usize,
    pub status: String,
}
```

#### Example

```rust
let health = storage.health().await?;

if !health.qdrant_connected {
    eprintln!("Qdrant is not connected!");
}

for collection in health.collections {
    println!("{}: {} points", collection.name, collection.points_count);
}

if health.disk_space_mb < 1000 {
    eprintln!("Low disk space: {} MB", health.disk_space_mb);
}
```

## Error Handling

All methods return `Result<T, anyhow::Error>` or `Result<T, E>` where `E` implements `std::error::Error`.

Common error scenarios:

- **QdrantError**: Connection failures, collection not found, invalid operations
- **EncryptionError**: Encryption/decryption failures, missing key
- **S3Error**: Network issues, permission errors, bucket not found
- **SerializationError**: Invalid data format

Example error handling:

```rust
use anyhow::Context;

match storage.estate.get_resource("resource-id").await {
    Ok(Some(resource)) => {
        println!("Found resource: {:?}", resource);
    }
    Ok(None) => {
        println!("Resource not found");
    }
    Err(e) => {
        eprintln!("Error retrieving resource: {}", e);
        // Chain context for better error messages
        return Err(e).context("Failed to get resource from estate storage");
    }
}
```

## Thread Safety

All storage components are designed to be used from multiple async tasks:

- `StorageService`, `ChatStorage`, `EstateStorage`, and `BackupManager` are all `Send + Sync`
- Internal locking is handled automatically
- Safe to clone and use across tasks

```rust
let storage = Arc::new(storage);

let storage_clone = storage.clone();
tokio::spawn(async move {
    storage_clone.estate.search("query", None).await.unwrap();
});

let storage_clone = storage.clone();
tokio::spawn(async move {
    storage_clone.chat.append("ctx", message).await.unwrap();
});
```

## See Also

- [Configuration](configuration.md) - Complete configuration reference
- [Collections](collections.md) - Collection schemas and point structures
- [Encryption](encryption.md) - Encryption implementation details
- [Backup & Restore](backup-restore.md) - Backup workflows and strategies
- [Operations](operations.md) - Common operation patterns
- [Testing](testing.md) - Testing strategies and examples