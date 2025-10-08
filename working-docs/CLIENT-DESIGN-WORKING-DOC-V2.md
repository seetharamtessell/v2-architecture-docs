# Client Architecture - Working Design Document

**Status**: Work in Progress - Design Phase
**Last Updated**: 2025-10-08
**Purpose**: Living document for client architecture design discussions

---

## Overview

Desktop application built with **React + Tauri** for conversational cloud operations management.

---

## Core Functions (The Big 3)

### 1. AWS Estate Management
- Read credentials from `~/.aws/credentials` on login
- Scan entire AWS estate (EC2, RDS, S3, Lambda, etc.)
- Store locally in Qdrant vector DB + cache
- Keep synced periodically

### 2. Chat History Management
- Store all conversations locally
- Maintain context per conversation thread
- Send relevant history to server with each request (LLM needs context)

### 3. Command Execution
- Receive commands from server
- Execute using Tauri Rust backend (AWS CLI or SDK)
- Store execution results locally

---

## Request Flow

```
USER PROMPT: "Stop pg-instance-main1"
    ↓
[STEP 1] Client searches local estate in Qdrant
    → Found: RDS instance
         - Region: us-west-2
         - Account: 123456789012
         - Status: available
         - Tags: {env: production}
    ↓
[STEP 2] Client enriches the request
    {
      "prompt": "Stop pg-instance-main1",
      "identified_resources": [{
        "type": "rds_instance",
        "id": "pg-instance-main1",
        "region": "us-west-2",
        "account": "123456789012",
        "current_state": "available",
        ...full metadata...
      }],
      "chat_history": [
        {prev conversation messages in this thread}
      ],
      "user_context": {
        "user_id": "...",
        "default_region": "us-west-2"
      }
    }
    ↓
[STEP 3] Send to Server
    ↓
[STEP 4] Server responds with ONE OF:
    A) Commands to execute
       → Client shows command + asks for approval
       → User approves
       → Execute via Rust
       → Store result

    B) Information/Explanation
       → Display to user

    C) Ask for confirmation/clarification
       → Show to user
       → Get user response
       → Loop back to STEP 2
    ↓
[STEP 5] Store interaction in chat history
```

---

## Storage Service (Critical Component)

The **Storage Service** is a **Pure Rust crate** that abstracts all local storage operations.

### Purpose
- Centralized storage layer used by Tauri Rust backend
- Handles both chat history and AWS estate data
- Uses Qdrant as the underlying vector database
- Provides clean, type-safe API for storage operations
- Implements encryption and S3 backup features

### Two Main Responsibilities

#### 1. Context & Chat History Management
```rust
// Rust API - Chat Operations
impl ChatStorage {
    async fn append(&self, context_id: &str, message: Message) -> Result<()>;
    async fn get_history(&self, context_id: &str, limit: Option<usize>) -> Result<Vec<Message>>;
    async fn delete(&self, context_id: &str) -> Result<()>;
    async fn update_context(&self, context_id: &str, metadata: Value) -> Result<()>;
    async fn search_conversations(&self, query: &str) -> Result<Vec<Conversation>>;
    async fn get_all_contexts(&self) -> Result<Vec<String>>;
}
```

#### 2. RAG with AWS Estate
```rust
// Rust API - Estate Operations
impl EstateStorage {
    async fn upsert_resource(&self, resource: AWSResource) -> Result<()>;
    async fn bulk_upsert(&self, resources: Vec<AWSResource>) -> Result<()>;
    async fn search(&self, query: &str, filters: Option<ResourceFilter>) -> Result<Vec<AWSResource>>;
    async fn get_resource(&self, resource_id: &str) -> Result<Option<AWSResource>>;
    async fn delete_resource(&self, resource_id: &str) -> Result<()>;
    async fn get_by_account(&self, account_id: &str) -> Result<Vec<AWSResource>>;
    async fn get_by_region(&self, region: &str) -> Result<Vec<AWSResource>>;
    async fn get_by_type(&self, resource_type: &str) -> Result<Vec<AWSResource>>;
    async fn count(&self) -> Result<usize>;
}
```

### Storage Architecture

```
┌──────────────────────────────────────────────────────┐
│   cloudops-storage-service (Pure Rust Crate)         │
├──────────────────────────────────────────────────────┤
│                                                       │
│  StorageService                                       │
│  ├── ChatStorage      (Qdrant collection)            │
│  ├── EstateStorage    (Qdrant collection)            │
│  └── BackupManager    (snapshot orchestration)       │
│                                                       │
│  Dependencies:                                        │
│  • qdrant-client      (Qdrant Rust client)           │
│  • aws-sdk-s3         (S3 operations)                 │
│  • tokio              (async runtime)                 │
│  • aes-gcm            (encryption)                    │
│  • keyring            (OS keychain access)            │
└──────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────┐
│   Qdrant (Local Instance)                            │
│   • Mode: Memmap (disk-backed with virtual memory)   │
│   • Collections: chat_history, aws_estate            │
│   • WAL: Enabled (Write-Ahead-Log)                   │
│   • Storage: ~/.cloudops/data/qdrant/                │
└──────────────────────────────────────────────────────┘
         ↓ (scheduled snapshots)
┌──────────────────────────────────────────────────────┐
│   Local Snapshots (Qdrant tar archives)              │
│   • Path: ~/.cloudops/snapshots/                     │
│   • Format: {collection}_{timestamp}.snapshot        │
│   • Retention: Last 7 days locally                   │
└──────────────────────────────────────────────────────┘
         ↓ (automatic upload - WE implement)
┌──────────────────────────────────────────────────────┐
│   S3 Backup (Optional, Configurable)                 │
│   • Bucket: s3://user-cloudops-backup/               │
│   • Path: {user_id}/snapshots/{collection}/          │
│   • Schedule: Every N hours (configurable)           │
│   • Retention: Last 30 days on S3                    │
│   • Encryption: AES-256 before upload                │
└──────────────────────────────────────────────────────┘
```

### What Qdrant Provides vs What We Implement

**✅ Qdrant Provides:**
- Vector storage (in-memory, memmap, or disk)
- Collections with point storage and search
- Snapshot creation (tar archives)
- S3 as snapshot destination (via config)
- Write-Ahead-Log (WAL) for crash recovery
- Excellent Rust client with async/await

**❌ Qdrant Does NOT Provide (We implement):**
- Automatic scheduled snapshots (no auto-sync)
- Application-level encryption (payloads encrypted before storage)
- Scheduled S3 uploads
- Snapshot cleanup and retention policies
- Key management (OS keychain integration)

### Storage Service API (Rust)

```rust
// Crate: cloudops-storage-service
// Cargo.toml: name = "cloudops-storage-service"

use anyhow::Result;
use serde::{Deserialize, Serialize};
use std::path::PathBuf;

/// Main configuration for storage service
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StorageConfig {
    /// Local data path (e.g., ~/.cloudops/data/)
    pub data_path: PathBuf,

    /// Qdrant configuration
    pub qdrant: QdrantConfig,

    /// Embedding configuration
    pub embedding: EmbeddingConfig,

    /// Qdrant collection configurations
    pub collections: CollectionConfigs,

    /// Encryption configuration
    pub encryption: EncryptionConfig,

    /// Optional S3 backup configuration
    pub s3_backup: Option<S3BackupConfig>,
}

/// Qdrant configuration
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QdrantConfig {
    /// Qdrant mode
    pub mode: QdrantMode,

    /// Storage path for embedded mode (e.g., ~/.cloudops/data/qdrant/)
    pub storage_path: Option<PathBuf>,

    /// URL for remote mode (e.g., http://localhost:6334)
    pub url: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum QdrantMode {
    /// Embedded mode - Qdrant runs in-process, stores data on filesystem
    Embedded,

    /// Remote mode - Connect to external Qdrant instance
    Remote,
}

/// Encryption settings
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EncryptionConfig {
    /// Enable/disable encryption
    pub enabled: bool,

    /// Where to get encryption key
    pub key_source: KeySource,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum KeySource {
    /// OS keychain (macOS Keychain, Windows Credential Manager, Linux Secret Service)
    OSKeychain { service: String, account: String },

    /// Environment variable
    EnvVar(String),

    /// File path (should be secured by filesystem permissions)
    File(PathBuf),
}

/// S3 backup configuration
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct S3BackupConfig {
    pub bucket: String,
    pub region: String,
    pub auto_sync: bool,
    pub interval_hours: u32,
    pub retention_days: u32,
}

/// Embedding configuration
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EmbeddingConfig {
    /// Embedding model provider
    pub provider: EmbeddingProvider,

    /// Model name (e.g., "text-embedding-3-small", "all-MiniLM-L6-v2")
    pub model: String,

    /// Vector dimension (must match Qdrant collection config)
    pub dimension: usize,

    /// When to generate embeddings
    pub generation_mode: EmbeddingGenerationMode,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EmbeddingProvider {
    /// OpenAI API
    OpenAI { api_key: String, base_url: Option<String> },

    /// Server API endpoint
    Server { endpoint: String, api_key: Option<String> },

    /// Local model (via HTTP endpoint like Ollama or local server)
    Local { endpoint: String },

    /// Fastembed (pure Rust embedding library)
    Fastembed { model_path: Option<PathBuf> },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EmbeddingGenerationMode {
    /// Generate during data insertion (slower write, faster search)
    OnWrite,

    /// Generate on-demand during search (faster write, slower search)
    OnSearch,

    /// Lazy: generate on first search, cache result
    Lazy,
}

/// Qdrant collection configurations
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CollectionConfigs {
    pub chat_history: CollectionConfig,
    pub aws_estate: CollectionConfig,
}

/// Configuration for a Qdrant collection
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CollectionConfig {
    /// Collection name
    pub name: String,

    /// Vector dimension
    pub vector_size: usize,

    /// Distance metric (Cosine, Euclidean, Dot)
    pub distance: DistanceMetric,

    /// Indexed payload fields (for fast filtering)
    pub indexed_fields: Vec<IndexedField>,

    /// HNSW configuration (for vector search)
    pub hnsw_config: Option<HnswConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DistanceMetric {
    Cosine,
    Euclidean,
    Dot,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IndexedField {
    /// Field name in payload
    pub name: String,

    /// Field type
    pub field_type: IndexFieldType,

    /// Create index?
    pub indexed: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum IndexFieldType {
    Keyword,
    Integer,
    Float,
    Bool,
    Geo,
    Text,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HnswConfig {
    /// Number of edges per node (default: 16)
    pub m: usize,

    /// Size of candidate list during construction (default: 100)
    pub ef_construct: usize,

    /// Full scan threshold - switch to exact search below this size
    pub full_scan_threshold: usize,
}

impl Default for HnswConfig {
    fn default() -> Self {
        Self {
            m: 16,
            ef_construct: 100,
            full_scan_threshold: 10000,
        }
    }
}

/// Main storage service struct
pub struct StorageService {
    pub chat: ChatStorage,
    pub estate: EstateStorage,
    pub backup: BackupManager,
}

impl StorageService {
    /// Initialize storage service with config
    pub async fn new(config: StorageConfig) -> Result<Self>;

    /// Health check
    pub async fn health(&self) -> Result<StorageHealth>;

    /// Shutdown gracefully
    pub async fn shutdown(&self) -> Result<()>;
}

/// Chat storage operations
pub struct ChatStorage {
    // Internal fields...
}

impl ChatStorage {
    pub async fn append(&self, context_id: &str, message: Message) -> Result<()>;
    pub async fn get_history(&self, context_id: &str, limit: Option<usize>) -> Result<Vec<Message>>;
    pub async fn delete(&self, context_id: &str) -> Result<()>;
    pub async fn update_context(&self, context_id: &str, metadata: serde_json::Value) -> Result<()>;
    pub async fn search_conversations(&self, query: &str) -> Result<Vec<Conversation>>;
    pub async fn get_all_contexts(&self) -> Result<Vec<String>>;
}

/// Estate storage operations
pub struct EstateStorage {
    // Internal fields...
}

impl EstateStorage {
    pub async fn upsert_resource(&self, resource: AWSResource) -> Result<()>;
    pub async fn bulk_upsert(&self, resources: Vec<AWSResource>) -> Result<()>;
    pub async fn search(&self, query: &str, filters: Option<ResourceFilter>) -> Result<Vec<AWSResource>>;
    pub async fn get_resource(&self, resource_id: &str) -> Result<Option<AWSResource>>;
    pub async fn delete_resource(&self, resource_id: &str) -> Result<()>;
    pub async fn get_by_account(&self, account_id: &str) -> Result<Vec<AWSResource>>;
    pub async fn get_by_region(&self, region: &str) -> Result<Vec<AWSResource>>;
    pub async fn get_by_type(&self, resource_type: &str) -> Result<Vec<AWSResource>>;
    pub async fn count(&self) -> Result<usize>;
}

/// Backup manager - handles snapshots and S3 sync
pub struct BackupManager {
    // Internal fields...
}

impl BackupManager {
    /// Create snapshot of a collection
    pub async fn create_snapshot(&self, collection: &str) -> Result<SnapshotInfo>;

    /// Upload snapshot to S3
    pub async fn upload_to_s3(&self, snapshot: &SnapshotInfo) -> Result<()>;

    /// Restore from local snapshot
    pub async fn restore_from_snapshot(&self, collection: &str, snapshot_name: &str) -> Result<()>;

    /// Restore from S3 snapshot
    pub async fn restore_from_s3(&self, collection: &str, date: &str) -> Result<()>;

    /// List available snapshots (local + S3)
    pub async fn list_snapshots(&self, collection: &str) -> Result<Vec<SnapshotInfo>>;

    /// Start automatic backup scheduler (runs in background)
    pub async fn start_auto_backup(&self) -> Result<()>;

    /// Stop automatic backup scheduler
    pub async fn stop_auto_backup(&self) -> Result<()>;

    /// Manual cleanup of old snapshots
    pub async fn cleanup_old_snapshots(&self) -> Result<()>;
}

/// Snapshot information
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnapshotInfo {
    pub name: String,
    pub collection: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub size_bytes: u64,
    pub location: SnapshotLocation,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SnapshotLocation {
    Local(PathBuf),
    S3 { bucket: String, key: String },
}

/// Storage health status
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

### Qdrant Collection Management

#### Important: Point ID Management

**Critical Design Decisions for Point IDs:**

Qdrant points are the fundamental storage unit. We must carefully design point IDs to avoid:
- ❌ ID collisions
- ❌ Orphaned data
- ❌ Duplicate entries
- ❌ Inconsistent updates

**Point ID Strategy:**

**Chat Messages:**
```rust
// Use UUID for each message (immutable, unique)
point_id: Uuid::new_v4().to_string()  // "550e8400-e29b-41d4-a716-446655440000"

// Why: Messages are immutable (never updated, only added/deleted)
// Benefits: Guaranteed unique, no collision risk
```

**AWS Resources:**
```rust
// Use deterministic ID based on resource identifier
point_id: format!("{}-{}-{}", account_id, region, resource_id)
// Example: "123456789012-us-west-2-pg-instance-main1"

// Why: Resources are updated frequently (state changes, tag updates)
// Benefits: Same resource = same point ID = upsert updates existing point
// Avoids: Creating duplicate entries for the same resource
```

**Point Lifecycle Management:**

```rust
impl EstateStorage {
    /// Upsert (Insert or Update) - idempotent
    pub async fn upsert_resource(&self, resource: AWSResource) -> Result<()> {
        // Deterministic ID ensures updates don't create duplicates
        let point_id = self.generate_resource_id(&resource);

        // If point exists, it will be updated. If not, created.
        self.qdrant.upsert_points(vec![Point {
            id: point_id,
            vector: self.embed(&resource).await?,
            payload: resource.to_payload(),
        }]).await?;

        Ok(())
    }

    /// Delete when resource no longer exists in AWS
    pub async fn delete_resource(&self, resource_id: &str) -> Result<()> {
        self.qdrant.delete_points(
            "aws_estate",
            &[resource_id.into()],
        ).await?;

        Ok(())
    }

    /// Sync strategy: Upsert current resources, delete stale ones
    pub async fn sync_from_aws(&self, current_resources: Vec<AWSResource>) -> Result<()> {
        // 1. Get all existing point IDs
        let existing_ids = self.get_all_point_ids().await?;

        // 2. Upsert all current resources
        for resource in current_resources {
            self.upsert_resource(resource).await?;
        }

        // 3. Delete resources that no longer exist in AWS
        let current_ids: HashSet<_> = current_resources
            .iter()
            .map(|r| self.generate_resource_id(r))
            .collect();

        let stale_ids: Vec<_> = existing_ids
            .into_iter()
            .filter(|id| !current_ids.contains(id))
            .collect();

        if !stale_ids.is_empty() {
            self.qdrant.delete_points("aws_estate", &stale_ids).await?;
        }

        Ok(())
    }
}
```

**Key Principles:**

1. **Immutable Data (Chat)**: Use random UUIDs
   - Never updated, only created/deleted
   - No collision risk

2. **Mutable Data (Estate)**: Use deterministic IDs
   - Same resource = same ID
   - Upsert instead of insert
   - Prevents duplicates

3. **Cleanup**: Always remove stale points
   - Resources deleted from AWS → delete from Qdrant
   - Conversations deleted → delete all messages

4. **Bulk Operations**: Use batch APIs
   ```rust
   // ✅ Good: Batch upsert
   qdrant.upsert_points(collection, vec![point1, point2, point3]).await?;

   // ❌ Bad: Individual upserts in loop
   for point in points {
       qdrant.upsert_points(collection, vec![point]).await?;
   }
   ```

5. **Idempotency**: Operations should be safe to retry
   - Upsert is idempotent (same result if called multiple times)
   - Delete is idempotent (no error if point doesn't exist)

**Point Count Monitoring:**

```rust
impl StorageService {
    /// Monitor point counts to detect issues
    pub async fn health(&self) -> Result<StorageHealth> {
        let chat_count = self.qdrant
            .count_points("chat_history")
            .await?;

        let estate_count = self.qdrant
            .count_points("aws_estate")
            .await?;

        // Alert if unexpected growth (possible duplicates)
        if estate_count > self.expected_resource_count * 2 {
            warn!("Unexpected point count in aws_estate: {}", estate_count);
        }

        Ok(StorageHealth {
            chat_points: chat_count,
            estate_points: estate_count,
            // ...
        })
    }
}
```

#### Collection Initialization

On first run, the storage service automatically creates collections with proper schema:

```rust
impl StorageService {
    pub async fn new(config: StorageConfig) -> Result<Self> {
        // Initialize Qdrant client
        let qdrant = match config.qdrant.mode {
            QdrantMode::Embedded => {
                QdrantClient::from_path(config.qdrant.storage_path.unwrap())
            }
            QdrantMode::Remote => {
                QdrantClient::from_url(config.qdrant.url.unwrap())
            }
        };

        // Initialize collections if they don't exist
        Self::init_collection_if_not_exists(
            &qdrant,
            &config.collections.chat_history,
        ).await?;

        Self::init_collection_if_not_exists(
            &qdrant,
            &config.collections.aws_estate,
        ).await?;

        Ok(Self { /* ... */ })
    }

    async fn init_collection_if_not_exists(
        qdrant: &QdrantClient,
        config: &CollectionConfig,
    ) -> Result<()> {
        // Check if collection exists
        if qdrant.collection_exists(&config.name).await? {
            return Ok(());
        }

        // Create collection
        qdrant.create_collection(CreateCollection {
            collection_name: config.name.clone(),
            vectors_config: Some(VectorsConfig {
                size: config.vector_size as u64,
                distance: match config.distance {
                    DistanceMetric::Cosine => Distance::Cosine,
                    DistanceMetric::Euclidean => Distance::Euclid,
                    DistanceMetric::Dot => Distance::Dot,
                },
            }),
            hnsw_config: config.hnsw_config.as_ref().map(|h| HnswConfigDiff {
                m: Some(h.m as u64),
                ef_construct: Some(h.ef_construct as u64),
                full_scan_threshold: Some(h.full_scan_threshold as u64),
                ..Default::default()
            }),
            ..Default::default()
        }).await?;

        // Create payload indexes
        for field in &config.indexed_fields {
            if field.indexed {
                qdrant.create_field_index(
                    &config.name,
                    &field.name,
                    match field.field_type {
                        IndexFieldType::Keyword => FieldType::Keyword,
                        IndexFieldType::Integer => FieldType::Integer,
                        IndexFieldType::Float => FieldType::Float,
                        IndexFieldType::Bool => FieldType::Bool,
                        IndexFieldType::Geo => FieldType::Geo,
                        IndexFieldType::Text => FieldType::Text,
                    },
                    None,
                    None,
                ).await?;
            }
        }

        Ok(())
    }
}
```

#### Collection Schemas

**Chat History Collection:**

**Access Pattern:** Filter by context_id only (no semantic search needed)
**Strategy:** Use minimal dummy vector (1 dimension) + indexed filters

```rust
CollectionConfig {
    name: "chat_history".to_string(),
    vector_size: 1, // Dummy vector (4 bytes overhead per message)
    distance: DistanceMetric::Cosine,
    indexed_fields: vec![
        IndexedField {
            name: "context_id".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true, // Fast filtering by conversation
        },
        IndexedField {
            name: "message_index".to_string(),
            field_type: IndexFieldType::Integer,
            indexed: true, // Order messages within conversation
        },
        IndexedField {
            name: "timestamp".to_string(),
            field_type: IndexFieldType::Integer,
            indexed: true, // Time-based queries
        },
    ],
    hnsw_config: None, // Not needed for dummy vectors
}
```

**Payload Structure (Chat):**
```json
{
  "id": "msg-uuid-12345",
  "vector": [0.0],  // Dummy vector (never used for search)
  "payload": {
    // ✅ PLAIN TEXT (for filtering)
    "context_id": "conversation-uuid",
    "message_index": 5,  // Position in conversation (0, 1, 2, ...)
    "role": "user",
    "timestamp": 1728391162,

    // ✅ ENCRYPTED (sensitive content)
    "encrypted_content": "base64-encrypted-blob",

    // After decryption, encrypted_content contains:
    // {
    //   "content": "Stop pg-instance-main1",
    //   "metadata": {
    //     "resources_mentioned": ["rds-pg-instance-main1"],
    //     "commands_executed": ["aws rds stop-db-instance ..."]
    //   }
    // }
  }
}
```

**Chat Operations (No Vector Search):**
```rust
impl ChatStorage {
    /// Append message to conversation (no embedding generation)
    pub async fn append(&self, context_id: &str, message: Message) -> Result<()> {
        // Get current message count
        let message_index = self.get_message_count(context_id).await?;

        // Encrypt content (no embedding needed)
        let encrypted = self.encryptor.encrypt(&serde_json::to_string(&json!({
            "content": message.content,
            "metadata": message.metadata,
        }))?)?;

        // Insert with dummy vector
        self.qdrant.upsert_points(vec![Point {
            id: Uuid::new_v4().to_string(),
            vector: vec![0.0],  // Dummy vector
            payload: json!({
                "context_id": context_id,
                "message_index": message_index,
                "role": message.role,
                "timestamp": Utc::now().timestamp(),
                "encrypted_content": encrypted,
            }),
        }]).await?;

        Ok(())
    }

    /// Get conversation history (ordered by message_index)
    pub async fn get_history(
        &self,
        context_id: &str,
        limit: Option<usize>,
    ) -> Result<Vec<Message>> {
        // Use scroll (not search) - pure filtering, no vector search
        let results = self.qdrant.scroll(Scroll {
            collection_name: "chat_history".to_string(),
            filter: Some(Filter {
                must: vec![
                    Condition::matches("context_id", context_id),
                ],
            }),
            order_by: Some(OrderBy {
                key: "message_index".to_string(),
                direction: Some(Direction::Asc),
            }),
            limit: limit.map(|l| l as u32),
            with_payload: Some(WithPayloadSelector::enable(true)),
        }).await?;

        // Decrypt and return
        results.result.into_iter()
            .map(|point| self.decrypt_message(point))
            .collect()
    }
}
```

**AWS Estate Collection:**

```rust
CollectionConfig {
    name: "aws_estate".to_string(),
    vector_size: 384, // or 768/1536 depending on embedding model
    distance: DistanceMetric::Cosine,
    indexed_fields: vec![
        IndexedField {
            name: "resource_type".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true, // Filter by EC2, RDS, S3, etc.
        },
        IndexedField {
            name: "account_id".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true, // Filter by AWS account
        },
        IndexedField {
            name: "region".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true, // Filter by region
        },
        IndexedField {
            name: "state".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true, // Filter by running/stopped/available
        },
        IndexedField {
            name: "last_synced".to_string(),
            field_type: IndexFieldType::Integer,
            indexed: true, // Track freshness
        },
        IndexedField {
            name: "tags.env".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true, // Filter by environment tag
        },
    ],
    hnsw_config: Some(HnswConfig::default()),
}
```

**Payload Structure (Estate):**
```json
{
  "id": "123456789012-us-west-2-rds-pg-instance-main1",
  "vector": [0.123, 0.456, ...],
  "payload": {
    // ✅ PLAIN TEXT (for filtering)
    "resource_type": "rds_instance",
    "account_id": "123456789012",
    "account_name": "production-account",
    "region": "us-west-2",
    "service": "rds",
    "state": "available",
    "last_synced": 1728391162,
    "tags": {
      "env": "production",
      "app": "main"
    },

    // ✅ ENCRYPTED (sensitive metadata + IAM permissions)
    "encrypted_data": "base64-encrypted-blob",

    // After decryption, encrypted_data contains:
    // {
    //   "identifier": "pg-instance-main1",
    //   "arn": "arn:aws:rds:us-west-2:123456789012:db:pg-instance-main1",
    //   "name": "pg-instance-main1",
    //   "engine": "postgres",
    //   "engine_version": "14.7",
    //
    //   // IAM permissions for THIS resource
    //   "iam": {
    //     "allowed_actions": [
    //       "rds:StopDBInstance",
    //       "rds:StartDBInstance",
    //       "rds:DescribeDBInstances",
    //       "rds:CreateDBSnapshot"
    //     ],
    //     "denied_actions": [
    //       "rds:DeleteDBInstance"
    //     ],
    //     "user_context": {
    //       "username": "john.doe",
    //       "role_arn": "arn:aws:iam::123456789012:role/DeveloperRole",
    //       "session_expires": 1728477562
    //     }
    //   },
    //
    //   "metadata": {
    //     "instance_class": "db.t3.medium",
    //     "allocated_storage": 100,
    //     "vpc_id": "vpc-123456",
    //     "subnet_group": "default",
    //     "publicly_accessible": false,
    //     "multi_az": true
    //   }
    // }
  }
}
```

**Why IAM is embedded in each resource:**
When user asks "stop pg-main1", we can immediately show:
- Resource details (state, region, engine, etc.)
- What permissions you have (can stop? can start? read-only?)
- No need for separate IAM lookup

#### Search with Filters

```rust
impl EstateStorage {
    /// Search with vector similarity + filters
    pub async fn search(
        &self,
        query: &str,
        filters: Option<ResourceFilter>,
    ) -> Result<Vec<AWSResource>> {
        // Generate embedding for query
        let vector = self.embedder.embed(query).await?;

        // Build filter conditions
        let filter = filters.map(|f| Filter {
            must: vec![
                f.resource_type.map(|t| Condition::matches("resource_type", t)),
                f.account_id.map(|a| Condition::matches("account_id", a)),
                f.region.map(|r| Condition::matches("region", r)),
                f.state.map(|s| Condition::matches("state", s)),
                f.tags.map(|tags| {
                    tags.into_iter().map(|(k, v)| {
                        Condition::matches(format!("tags.{}", k), v)
                    }).collect()
                }),
            ].into_iter().flatten().collect(),
            ..Default::default()
        });

        // Search with vector + filters
        let results = self.qdrant.search_points(SearchPoints {
            collection_name: "aws_estate".to_string(),
            vector,
            filter,
            limit: 10,
            with_payload: Some(WithPayloadSelector::enable(true)),
            ..Default::default()
        }).await?;

        // Decrypt and deserialize results
        let resources = results.result.into_iter()
            .map(|point| self.decrypt_resource(point))
            .collect::<Result<Vec<_>>>()?;

        Ok(resources)
    }
}

pub struct ResourceFilter {
    pub resource_type: Option<String>,
    pub account_id: Option<String>,
    pub region: Option<String>,
    pub state: Option<String>,
    pub tags: Option<HashMap<String, String>>,
}
```

#### Restore from S3

```rust
impl BackupManager {
    /// Restore collection from S3 snapshot
    pub async fn restore_from_s3(
        &self,
        collection: &str,
        snapshot_date: &str,
    ) -> Result<()> {
        // 1. List snapshots in S3
        let s3_key = format!("{}/snapshots/{}/{}",
            self.config.user_id,
            collection,
            snapshot_date
        );

        // 2. Download snapshot from S3 to local temp
        let local_path = self.config.data_path
            .join("temp")
            .join(format!("{}.snapshot", snapshot_date));

        self.s3_client
            .get_object()
            .bucket(&self.config.s3_backup.bucket)
            .key(&s3_key)
            .send()
            .await?
            .body
            .collect()
            .await?
            .write_to_file(&local_path)?;

        // 3. Delete existing collection
        self.qdrant.delete_collection(collection).await?;

        // 4. Restore from snapshot
        self.qdrant.restore_snapshot(RestoreSnapshot {
            collection_name: collection.to_string(),
            snapshot_path: local_path.to_string_lossy().to_string(),
            priority: Some(SnapshotPriority::Snapshot), // Use snapshot data
        }).await?;

        // 5. Cleanup temp file
        tokio::fs::remove_file(local_path).await?;

        Ok(())
    }

    /// List available snapshots in S3
    pub async fn list_s3_snapshots(
        &self,
        collection: &str,
    ) -> Result<Vec<SnapshotInfo>> {
        let prefix = format!("{}/snapshots/{}/",
            self.config.user_id,
            collection
        );

        let objects = self.s3_client
            .list_objects_v2()
            .bucket(&self.config.s3_backup.bucket)
            .prefix(&prefix)
            .send()
            .await?;

        let snapshots = objects.contents()
            .iter()
            .map(|obj| SnapshotInfo {
                name: obj.key().unwrap().to_string(),
                collection: collection.to_string(),
                created_at: obj.last_modified().unwrap().to_chrono_utc(),
                size_bytes: obj.size() as u64,
                location: SnapshotLocation::S3 {
                    bucket: self.config.s3_backup.bucket.clone(),
                    key: obj.key().unwrap().to_string(),
                },
            })
            .collect();

        Ok(snapshots)
    }

    /// Full restore workflow (with options)
    pub async fn restore_workflow(
        &self,
        collection: &str,
        source: RestoreSource,
        mode: RestoreMode,
    ) -> Result<()> {
        match source {
            RestoreSource::Local(snapshot_name) => {
                self.restore_from_snapshot(collection, &snapshot_name).await?;
            }
            RestoreSource::S3(date) => {
                self.restore_from_s3(collection, &date).await?;
            }
            RestoreSource::Latest => {
                // Get latest snapshot
                let snapshots = self.list_s3_snapshots(collection).await?;
                let latest = snapshots.first()
                    .ok_or_else(|| anyhow!("No snapshots found"))?;

                let date = latest.name.split('/').last().unwrap();
                self.restore_from_s3(collection, date).await?;
            }
        }

        Ok(())
    }
}

pub enum RestoreSource {
    Local(String),      // Local snapshot name
    S3(String),         // S3 snapshot date
    Latest,             // Latest available snapshot
}

pub enum RestoreMode {
    Replace,            // Delete collection and restore
    Merge,              // Merge with existing data (upsert)
}
```

#### Example Usage

```rust
// Initialize storage
let config = StorageConfig {
    data_path: PathBuf::from("~/.cloudops/data/"),
    qdrant: QdrantConfig {
        mode: QdrantMode::Embedded,
        storage_path: Some(PathBuf::from("~/.cloudops/data/qdrant/")),
        url: None,
    },
    embedding: EmbeddingConfig {
        provider: EmbeddingProvider::Server {
            endpoint: "https://api.example.com/embeddings".to_string(),
            api_key: Some("key".to_string()),
        },
        model: "text-embedding-3-small".to_string(),
        dimension: 384,
        generation_mode: EmbeddingGenerationMode::OnWrite,
    },
    collections: CollectionConfigs {
        chat_history: CollectionConfig { /* ... */ },
        aws_estate: CollectionConfig { /* ... */ },
    },
    encryption: EncryptionConfig {
        enabled: true,
        key_source: KeySource::OSKeychain {
            service: "cloudops".to_string(),
            account: "encryption_key".to_string(),
        },
    },
    s3_backup: Some(S3BackupConfig {
        bucket: "my-cloudops-backup".to_string(),
        region: "us-west-2".to_string(),
        auto_sync: true,
        interval_hours: 24,
        retention_days: 30,
    }),
};

let storage = StorageService::new(config).await?;

// Search resources
let results = storage.estate.search(
    "postgres database",
    Some(ResourceFilter {
        resource_type: Some("rds_instance".to_string()),
        region: Some("us-west-2".to_string()),
        state: Some("available".to_string()),
        ..Default::default()
    })
).await?;

// Restore from S3
storage.backup.restore_from_s3("aws_estate", "2025-10-08").await?;

// List available backups
let backups = storage.backup.list_s3_snapshots("chat_history").await?;
```

### Encryption Strategy

**Goal**: Make Qdrant's local files unreadable (application-level encryption)

**What We Encrypt:**
- ✅ Message content (chat messages)
- ✅ AWS resource metadata (ARNs, configs, sensitive data)
- ❌ Vectors (needed for semantic search)
- ❌ Filter fields (context_id, role, timestamps - needed for queries)

**Encrypted Storage Format:**
```rust
// What's stored in Qdrant
{
  id: "msg-uuid",
  vector: [0.123, 0.456, ...],  // Unencrypted (for search)
  payload: {
    // Plain text (for filtering/queries)
    context_id: "ctx-123",
    role: "user",
    timestamp: 1728391162,

    // Encrypted content ✅
    encrypted_content: "xK9mP2vL8nQ5...",  // Base64-encoded ciphertext
  }
}
```

**Encryption Implementation:**
- **Algorithm**: AES-256-GCM (authenticated encryption)
- **Key Management**: OS Keychain (automatic)
  - macOS: Keychain Access
  - Windows: Credential Manager
  - Linux: Secret Service (gnome-keyring/kwallet)
- **Key Generation**: Auto-generated on first run, stored in OS keychain
- **Rust Crates**: `aes-gcm` (encryption), `keyring` (key storage), `base64` (encoding)

**Simple Encryptor Module:**
```rust
// src/encryption.rs
use aes_gcm::{Aes256Gcm, Nonce, KeyInit, aead::Aead};

pub struct Encryptor {
    cipher: Aes256Gcm,
}

impl Encryptor {
    /// Initialize - gets/creates key from OS keychain automatically
    pub fn new() -> Result<Self> {
        let key = Self::get_or_create_key_from_keychain()?;
        Ok(Self { cipher: Aes256Gcm::new(&key) })
    }

    /// Encrypt text to base64 string
    pub fn encrypt(&self, plaintext: &str) -> Result<String> {
        let nonce = Aes256Gcm::generate_nonce(&mut OsRng);
        let ciphertext = self.cipher.encrypt(&nonce, plaintext.as_bytes())?;

        // Combine nonce + ciphertext, encode as base64
        let mut result = nonce.to_vec();
        result.extend_from_slice(&ciphertext);
        Ok(base64::encode(result))
    }

    /// Decrypt base64 string to text
    pub fn decrypt(&self, encrypted: &str) -> Result<String> {
        let decoded = base64::decode(encrypted)?;
        let (nonce_bytes, ciphertext) = decoded.split_at(12);
        let nonce = Nonce::from_slice(nonce_bytes);

        let plaintext = self.cipher.decrypt(nonce, ciphertext)?;
        Ok(String::from_utf8(plaintext)?)
    }
}
```

**Usage in Storage Service:**
```rust
// Storing
let encrypted = encryptor.encrypt(&message.content)?;
qdrant.upsert_points(vec![Point {
    vector: embedding,
    payload: json!({ "encrypted_content": encrypted }),
}]).await?;

// Retrieving
let point = qdrant.get_point(id).await?;
let decrypted = encryptor.decrypt(
    point.payload["encrypted_content"].as_str().unwrap()
)?;
```

**Key Security:**
- First run: Generate random 256-bit key → Store in OS keychain
- Subsequent runs: Load key from OS keychain
- Protected by user's OS login password
- Can't be accessed by other applications

**In Transit to S3:**
- Snapshots contain already-encrypted payloads
- Additional tar.gz compression before upload
- S3 bucket has server-side encryption enabled (SSE-S3)
- IAM policies restrict access

### Backup Strategy (We Implement Auto-Sync)

**Automatic Backup to S3 (Our Implementation):**
```rust
// BackupManager spawns a background task
impl BackupManager {
    pub async fn start_auto_backup(&self) -> Result<()> {
        let interval = Duration::from_secs(self.config.interval_hours * 3600);

        tokio::spawn(async move {
            loop {
                tokio::time::sleep(interval).await;

                // 1. Create Qdrant snapshot (local tar file)
                let snapshot = self.qdrant_client
                    .create_snapshot("chat_history")
                    .await?;

                // 2. Upload to S3
                self.upload_snapshot_to_s3(&snapshot).await?;

                // 3. Cleanup old local snapshots (keep last 7 days)
                self.cleanup_old_local_snapshots().await?;

                // 4. Cleanup old S3 snapshots (keep retention_days)
                self.cleanup_old_s3_snapshots().await?;
            }
        });
    }
}
```

**Schedule:**
- Configurable: Every N hours (default: 24 hours)
- Runs in background tokio task
- Non-blocking, doesn't affect main operations

**Retention:**
- Local snapshots: Last 7 days
- S3 snapshots: Configurable (default: 30 days)

**Manual Backup:**
- User can trigger on-demand via UI
- `backup_manager.create_snapshot("collection").await?`

**Restore:**
- From local snapshot: `restore_from_snapshot(collection, name)`
- From S3 snapshot: `restore_from_s3(collection, date)`
- Options: Replace or merge with existing data

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│              REACT FRONTEND                         │
├─────────────────────────────────────────────────────┤
│  Main Components:                                   │
│  • ChatInterface        - Main conversation UI      │
│  • ResourceExplorer     - Browse AWS estate         │
│  • ExecutionMonitor     - Show running commands     │
│  • ConversationHistory  - Past chat threads         │
│  • AccountManager       - Manage AWS credentials    │
│                                                      │
│  State Management: TBD (Redux/Zustand/Context)      │
└─────────────────────────────────────────────────────┘
              ↕ Tauri Commands (IPC)
┌─────────────────────────────────────────────────────┐
│              RUST BACKEND                           │
├─────────────────────────────────────────────────────┤
│  Core Modules:                                      │
│                                                      │
│  credentials_reader                                 │
│    → Read ~/.aws/credentials                        │
│    → Parse profiles                                 │
│    → Optional: Store in OS Keychain                 │
│                                                      │
│  estate_scanner                                     │
│    → AWS API calls (parallel per service)           │
│    → EC2, RDS, S3, Lambda, VPC, ELB, etc.          │
│    → Transform to standard format                   │
│    → Handle pagination, rate limits, errors         │
│                                                      │
│  storage_service (uses cloudops-storage-service)    │
│    → Initialize StorageService                      │
│    → ChatStorage + EstateStorage + BackupManager    │
│    → Encrypt/decrypt payloads                       │
│    → Manage Qdrant collections                      │
│                                                      │
│  request_builder                                    │
│    → Take user prompt                               │
│    → Search estate for mentioned resources          │
│    → Enrich with full metadata                      │
│    → Add chat history                               │
│    → Build final request JSON                       │
│                                                      │
│  executor                                           │
│    → Execute AWS CLI commands                       │
│    → OR use AWS SDK (Rust)                         │
│    → Stream output                                  │
│    → Handle errors                                  │
│    → Store execution logs                           │
│                                                      │
│  server_client                                      │
│    → HTTP client to server                          │
│    → Send enriched requests                         │
│    → Receive responses                              │
│    → Handle retries, timeouts                       │
│                                                      │
│  sync_orchestrator                                  │
│    → Schedule periodic syncs                        │
│    → Full sync (6h) vs incremental (15min)         │
│    → Track sync status per service                  │
└─────────────────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────────────────┐
│           LOCAL STORAGE                             │
│           (via cloudops-storage-service)            │
├─────────────────────────────────────────────────────┤
│  • Qdrant (local instance)                          │
│      - Collections: chat_history, aws_estate        │
│      - Memmap storage (disk-backed)                 │
│      - WAL enabled                                  │
│                                                      │
│  • Snapshots                                        │
│      - Local: ~/.cloudops/snapshots/                │
│      - S3: s3://bucket/snapshots/ (optional)        │
│                                                      │
│  • OS Keychain                                      │
│      - Encryption keys                              │
│      - AWS credentials (optional)                   │
└─────────────────────────────────────────────────────┘
```

---

## Data Models

### AWS Resource (in Qdrant)

```json
{
  "id": "rds-pg-instance-main1",
  "vector": [0.123, 0.456, ...],
  "payload": {
    // THIS PAYLOAD IS ENCRYPTED before storing in Qdrant
    "encrypted_data": "...base64 encrypted blob...",

    // After decryption, structure is:
    "resource_type": "rds_instance",
    "identifier": "pg-instance-main1",
    "account_id": "123456789012",
    "region": "us-west-2",
    "arn": "arn:aws:rds:us-west-2:123456789012:db:pg-instance-main1",
    "name": "pg-instance-main1",
    "state": "available",
    "engine": "postgres",
    "engine_version": "14.7",
    "tags": {
      "env": "production",
      "app": "main",
      "owner": "team-backend"
    },
    "permissions": [
      "rds:StopDBInstance",
      "rds:StartDBInstance",
      "rds:CreateDBSnapshot"
    ],
    "constraints": {
      "can_stop": true,
      "has_read_replicas": false,
      "multi_az": true
    },
    "metadata": {
      "instance_class": "db.t3.medium",
      "allocated_storage": 100,
      "vpc_id": "vpc-123456",
      "subnet_group": "default"
    },
    "last_synced": "2025-10-08T10:30:00Z"
  }
}
```

### Chat Message (in Qdrant chat_history collection)

```json
{
  "id": "msg-uuid-12345",
  "vector": [0.234, 0.567, ...],
  "payload": {
    // ENCRYPTED
    "context_id": "conversation-uuid",
    "role": "user | assistant | system",
    "content": "Stop pg-instance-main1",
    "timestamp": "2025-10-08T10:30:00Z",
    "metadata": {
      "resources_mentioned": ["rds-pg-instance-main1"],
      "commands_executed": ["aws rds stop-db-instance ..."]
    }
  }
}
```

---

## Tauri Command Interface

Commands exposed from Rust backend to React frontend:

```rust
// Credentials
#[tauri::command]
async fn read_aws_credentials() -> Result<Vec<AwsProfile>, String>

#[tauri::command]
async fn select_aws_profile(profile_name: String) -> Result<(), String>

// Estate Management
#[tauri::command]
async fn scan_aws_estate(profile: String) -> Result<ScanProgress, String>

#[tauri::command]
async fn get_scan_status() -> Result<ScanStatus, String>

#[tauri::command]
async fn search_resources(query: String) -> Result<Vec<Resource>, String>

#[tauri::command]
async fn get_resource_details(resource_id: String) -> Result<Resource, String>

// Chat
#[tauri::command]
async fn create_conversation() -> Result<String, String> // returns conversation_id

#[tauri::command]
async fn get_conversations() -> Result<Vec<Conversation>, String>

#[tauri::command]
async fn get_conversation_messages(conversation_id: String) -> Result<Vec<Message>, String>

#[tauri::command]
async fn send_message(conversation_id: String, prompt: String) -> Result<ServerResponse, String>

// Execution
#[tauri::command]
async fn execute_command(execution_id: String, approved: bool) -> Result<ExecutionResult, String>

#[tauri::command]
async fn get_execution_logs(conversation_id: String) -> Result<Vec<Execution>, String>

// Sync & Backup
#[tauri::command]
async fn trigger_sync(sync_type: String) -> Result<(), String> // 'full' | 'incremental'

#[tauri::command]
async fn get_sync_status() -> Result<SyncStatus, String>

#[tauri::command]
async fn create_backup() -> Result<SnapshotInfo, String>

#[tauri::command]
async fn list_backups() -> Result<Vec<SnapshotInfo>, String>

#[tauri::command]
async fn restore_backup(snapshot_name: String) -> Result<(), String>
```

---

## Design Decisions To Make

### 1. Estate Scanning

**Questions:**
- Which AWS services to support initially?
  - Start with: EC2, RDS, S3, Lambda, VPC, ELB?
  - Add more later: ECS, EKS, DynamoDB, CloudFront, etc.?
- How to handle large estates (10k+ resources)?
  - Pagination strategy?
  - Memory management?
- Multi-account support:
  - Scan all profiles from ~/.aws/credentials automatically?
  - Or let user select which accounts to scan?
- Error handling:
  - What if scan fails for one service?
  - Continue with others or abort?

**Decisions:**
- [ ] TBD

---

### 2. Chat History Context

**Questions:**
- How much history to send to server?
  - Last 10 messages?
  - Last N tokens (to fit LLM context window)?
  - Smart selection (only relevant messages)?
- Conversation management:
  - Single continuous conversation?
  - Multiple separate threads?
  - Can user organize/label conversations?
- Context retention:
  - Remember resource mentions across messages?
  - Remember user preferences (default region, approval settings)?

**Decisions:**
- [ ] TBD

---

### 3. Execution Model

**Questions:**
- Execution method:
  - Use AWS CLI (simpler, but requires CLI installed)?
  - Use AWS SDK for Rust (more control, but more code)?
  - Support both?
- Approval workflow:
  - Always require approval?
  - Support "auto-approve" for low-risk operations?
  - Risk levels: low/medium/high?
- Output display:
  - Stream output in real-time?
  - Show after completion?
  - Store output for later review?
- Multi-step operations:
  - Execute sequentially with approval between steps?
  - Show all steps upfront and approve once?

**Decisions:**
- [ ] TBD

---

### 4. State Management (React)

**Questions:**
- Which library?
  - Redux (full-featured, verbose)
  - Zustand (lightweight, simple)
  - Jotai (atomic state)
  - React Context (built-in, good enough?)
- What state needs to be global?
  - Current conversation
  - AWS resources cache (or always query Rust?)
  - Execution status
  - User preferences

**Decisions:**
- [ ] TBD

---

### 5. Qdrant Embeddings

**Questions:**
- How to generate embeddings?
  - Local model (e.g., sentence-transformers via Python bridge)?
  - Call server API for embeddings?
  - Use OpenAI API?
- When to generate?
  - During sync (slower sync, faster search)?
  - On-demand during search (faster sync, slower search)?
- What to embed?
  - Resource name + tags + description?
  - Just name?
  - Include ARN, ID?

**Decisions:**
- [ ] TBD

---

### 6. Security & Credentials

**Questions:**
- Credential storage:
  - Keep using ~/.aws/credentials (simpler)?
  - Copy to OS Keychain (more secure)?
  - Encrypt locally?
- Session management:
  - How long to keep credentials in memory?
  - Re-authenticate periodically?
- MFA handling:
  - Support MFA?
  - How to prompt user for MFA token?
- Audit logging:
  - What to log locally?
  - All operations? Only executions?
  - Retention policy?

**Decisions:**
- [ ] TBD

---

## Technical Stack

### Frontend
- **Framework**: React 18+
- **UI Library**: TBD (Material-UI, Ant Design, Shadcn, custom?)
- **State Management**: TBD
- **Routing**: React Router (if multi-page) or single-page?
- **Code Editor**: Monaco Editor (for showing/editing commands)
- **Markdown**: For rendering rich responses from server

### Backend (Rust)
- **Framework**: Tauri 2.x
- **HTTP Client**: reqwest
- **AWS SDK**: aws-sdk-rust (official AWS SDK for Rust)
- **Storage Service**: cloudops-storage-service (our Rust crate)
- **Vector DB**: qdrant-client (Rust client for Qdrant)
- **Encryption**: aes-gcm (AES-256-GCM)
- **Key Storage**: keyring (OS keychain access)
- **JSON**: serde_json
- **Async Runtime**: tokio

### Local Services
- **Qdrant**: Run as local service (Docker or binary)
- **Location**: ~/.cloudops/data/qdrant/

---

## Open Questions & Notes

### Performance
- How to handle slow AWS API calls during initial scan?
  - Show progress bar?
  - Background scan with notification?
- Search performance in Qdrant with 10k+ resources?
- React performance with long chat history?
- Encryption overhead for payloads?

### UX Considerations
- First-time user experience:
  1. Install app
  2. Grant permission to read ~/.aws/credentials
  3. Select AWS profile(s)
  4. Initial scan (show progress)
  5. Ready to chat
- Dark mode support?
- Keyboard shortcuts?
- Copy/paste commands?
- Export conversation history?

### Error Handling
- Network errors (server unreachable)?
- AWS API errors (throttling, permissions)?
- Qdrant errors?
- Invalid user input?
- Encryption key missing/corrupted?

### Testing Strategy
- Unit tests for Rust modules
- Integration tests for Tauri commands
- E2E tests for React UI
- Mock AWS APIs for testing?
- Test encryption/decryption
- Test snapshot/restore

---

## Next Steps

1. **Finalize core decisions** (execution model, state management, embeddings)
2. **Design cloudops-storage-service crate** in detail
3. **Create Rust module designs** for Tauri backend
4. **Design React component hierarchy**
5. **Define Qdrant collection schemas** (vectors, payloads)
6. **Create sequence diagrams** for key flows
7. **Prototype critical paths** (scan → search → execute)

---

## Notes from Discussion

- Connection unstable (working from airplane) - created offline doc
- **Pure Rust crate** for storage service (not npm package)
- Qdrant provides: storage, snapshots, S3 destination
- Qdrant does NOT provide: auto-sync, encryption, key management
- We implement: encryption, auto-backup scheduler, retention policies
- Storage backend configurable for future flexibility

---

## References

- Main architecture doc: [architecture.md](architecture.md)
- Client overview: [docs/02-client/overview.md](docs/02-client/overview.md)
- Server overview: [docs/03-server/overview.md](docs/03-server/overview.md)
- Qdrant docs: https://qdrant.tech/documentation/
- Qdrant Rust client: https://docs.rs/qdrant-client