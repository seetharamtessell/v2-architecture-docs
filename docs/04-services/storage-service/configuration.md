# Storage Service Configuration

Complete configuration reference for the Storage Service.

## Overview

The Storage Service is configured through a single `StorageConfig` struct that defines:
- Local data paths
- Qdrant settings (embedded vs remote)
- Embedding generation
- Collection schemas
- Encryption settings
- S3 backup configuration

## StorageConfig

Main configuration struct for the storage service.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StorageConfig {
    /// Local data path (e.g., ~/.cloudops/data/)
    pub data_path: PathBuf,

    /// Qdrant configuration
    pub qdrant: QdrantConfig,

    /// Embedding configuration
    pub embedding: EmbeddingConfig,

    /// Collection configurations
    pub collections: CollectionConfigs,

    /// Encryption configuration
    pub encryption: EncryptionConfig,

    /// Optional S3 backup configuration
    pub s3_backup: Option<S3BackupConfig>,
}
```

### Example: Complete Configuration

```rust
use cloudops_storage_service::*;
use std::path::PathBuf;

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
        chat_history: CollectionConfig {
            name: "chat_history".to_string(),
            vector_size: 1, // Dummy vector
            distance: DistanceMetric::Cosine,
            indexed_fields: vec![
                IndexedField {
                    name: "context_id".to_string(),
                    field_type: IndexFieldType::Keyword,
                    indexed: true,
                },
                IndexedField {
                    name: "message_index".to_string(),
                    field_type: IndexFieldType::Integer,
                    indexed: true,
                },
            ],
            hnsw_config: None,
        },
        cloud_estate: CollectionConfig {
            name: "cloud_estate".to_string(),
            vector_size: 384,
            distance: DistanceMetric::Cosine,
            indexed_fields: vec![
                IndexedField {
                    name: "resource_type".to_string(),
                    field_type: IndexFieldType::Keyword,
                    indexed: true,
                },
                IndexedField {
                    name: "account_id".to_string(),
                    field_type: IndexFieldType::Keyword,
                    indexed: true,
                },
                IndexedField {
                    name: "region".to_string(),
                    field_type: IndexFieldType::Keyword,
                    indexed: true,
                },
            ],
            hnsw_config: Some(HnswConfig::default()),
        },
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
```

## Qdrant Configuration

### QdrantConfig

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QdrantConfig {
    /// Qdrant mode (Embedded or Remote)
    pub mode: QdrantMode,

    /// Storage path for embedded mode (e.g., ~/.cloudops/data/qdrant/)
    pub storage_path: Option<PathBuf>,

    /// URL for remote mode (e.g., http://localhost:6334)
    pub url: Option<String>,
}
```

### QdrantMode

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum QdrantMode {
    /// Embedded mode - Qdrant runs in-process
    /// Data stored on filesystem
    Embedded,

    /// Remote mode - Connect to external Qdrant instance
    Remote,
}
```

### Examples

#### Embedded Mode (Recommended)

```rust
QdrantConfig {
    mode: QdrantMode::Embedded,
    storage_path: Some(PathBuf::from("~/.cloudops/data/qdrant/")),
    url: None,
}
```

**Benefits:**
- No external dependencies
- Simpler setup
- Better performance (no network overhead)
- Automatic lifecycle management

**Use When:**
- Desktop application
- Single-user use case
- Simplicity preferred

#### Remote Mode

```rust
QdrantConfig {
    mode: QdrantMode::Remote,
    storage_path: None,
    url: Some("http://localhost:6334".to_string()),
}
```

**Benefits:**
- Can connect to existing Qdrant instance
- Shared storage across multiple clients
- Easier to inspect data (Qdrant UI)

**Use When:**
- Testing against specific Qdrant version
- Debugging with Qdrant UI
- Sharing data across processes

## Embedding Configuration

### EmbeddingConfig

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EmbeddingConfig {
    /// Embedding model provider
    pub provider: EmbeddingProvider,

    /// Model name
    pub model: String,

    /// Vector dimension (must match collection config)
    pub dimension: usize,

    /// When to generate embeddings
    pub generation_mode: EmbeddingGenerationMode,
}
```

### EmbeddingProvider

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EmbeddingProvider {
    /// OpenAI API
    OpenAI {
        api_key: String,
        base_url: Option<String>,
    },

    /// Server API endpoint
    Server {
        endpoint: String,
        api_key: Option<String>,
    },

    /// Local model (via HTTP endpoint like Ollama)
    Local {
        endpoint: String,
    },

    /// Fastembed (pure Rust embedding library)
    Fastembed {
        model_path: Option<PathBuf>,
    },
}
```

### EmbeddingGenerationMode

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EmbeddingGenerationMode {
    /// Generate during data insertion (slower write, faster search)
    OnWrite,

    /// Generate on-demand during search (faster write, slower search)
    OnSearch,

    /// Lazy: generate on first search, cache result
    Lazy,
}
```

### Examples

#### Server API (Recommended)

```rust
EmbeddingConfig {
    provider: EmbeddingProvider::Server {
        endpoint: "https://api.cloudops.example.com/embeddings".to_string(),
        api_key: Some("api-key-123".to_string()),
    },
    model: "text-embedding-3-small".to_string(),
    dimension: 384,
    generation_mode: EmbeddingGenerationMode::OnWrite,
}
```

**Benefits:**
- Centralized model updates
- No local model management
- Consistent embeddings across clients

#### OpenAI API

```rust
EmbeddingConfig {
    provider: EmbeddingProvider::OpenAI {
        api_key: "sk-...".to_string(),
        base_url: None, // Use default OpenAI API
    },
    model: "text-embedding-3-small".to_string(),
    dimension: 1536, // text-embedding-3-small dimension
    generation_mode: EmbeddingGenerationMode::OnWrite,
}
```

**Supported Models:**
- `text-embedding-3-small` (1536 dimensions)
- `text-embedding-3-large` (3072 dimensions)
- `text-embedding-ada-002` (1536 dimensions)

#### Local Model (Ollama)

```rust
EmbeddingConfig {
    provider: EmbeddingProvider::Local {
        endpoint: "http://localhost:11434/api/embeddings".to_string(),
    },
    model: "all-minilm".to_string(),
    dimension: 384,
    generation_mode: EmbeddingGenerationMode::OnWrite,
}
```

**Benefits:**
- No external API dependencies
- Privacy (data doesn't leave machine)
- No usage costs

**Requirements:**
- Ollama installed and running
- Model pulled (`ollama pull all-minilm`)

#### Fastembed (Pure Rust)

```rust
EmbeddingConfig {
    provider: EmbeddingProvider::Fastembed {
        model_path: Some(PathBuf::from("~/.cloudops/models/")),
    },
    model: "all-MiniLM-L6-v2".to_string(),
    dimension: 384,
    generation_mode: EmbeddingGenerationMode::OnWrite,
}
```

**Benefits:**
- No external process required
- Fast (GPU acceleration support)
- Pure Rust (no Python dependency)

**Models:**
- `all-MiniLM-L6-v2` (384 dimensions)
- `all-MiniLM-L12-v2` (384 dimensions)

## Collection Configuration

### CollectionConfigs

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CollectionConfigs {
    pub chat_history: CollectionConfig,
    pub cloud_estate: CollectionConfig,
}
```

### CollectionConfig

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CollectionConfig {
    /// Collection name
    pub name: String,

    /// Vector dimension
    pub vector_size: usize,

    /// Distance metric
    pub distance: DistanceMetric,

    /// Indexed payload fields
    pub indexed_fields: Vec<IndexedField>,

    /// HNSW configuration (for vector search)
    pub hnsw_config: Option<HnswConfig>,
}
```

### DistanceMetric

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DistanceMetric {
    /// Cosine similarity (most common for text embeddings)
    Cosine,

    /// Euclidean distance
    Euclidean,

    /// Dot product
    Dot,
}
```

**Recommendations:**
- **Text embeddings**: Use `Cosine` (normalized embeddings)
- **Image embeddings**: Use `Euclidean` or `Cosine`
- **Raw features**: Use `Euclidean`

### IndexedField

```rust
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
    Keyword,    // Exact string matching
    Integer,    // Numeric values
    Float,      // Floating-point numbers
    Bool,       // Boolean values
    Geo,        // Geographic coordinates
    Text,       // Full-text search
}
```

### HnswConfig

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HnswConfig {
    /// Number of edges per node (default: 16)
    pub m: usize,

    /// Size of candidate list during construction (default: 100)
    pub ef_construct: usize,

    /// Full scan threshold (default: 10000)
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
```

**HNSW Parameters:**
- **m**: Higher = better recall, more memory
- **ef_construct**: Higher = better index quality, slower build
- **full_scan_threshold**: Switch to exact search below this size

### Example: Chat History Collection

```rust
CollectionConfig {
    name: "chat_history".to_string(),
    vector_size: 1, // Dummy vector (no semantic search needed)
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
            indexed: true, // Order messages
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

### Example: Cloud Estate Collection

```rust
CollectionConfig {
    name: "cloud_estate".to_string(),
    vector_size: 384, // Match embedding dimension
    distance: DistanceMetric::Cosine,
    indexed_fields: vec![
        IndexedField {
            name: "resource_type".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "account_id".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "region".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "state".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "tags.env".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true, // Nested field indexing
        },
    ],
    hnsw_config: Some(HnswConfig::default()),
}
```

## Encryption Configuration

### EncryptionConfig

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EncryptionConfig {
    /// Enable/disable encryption
    pub enabled: bool,

    /// Where to get encryption key
    pub key_source: KeySource,
}
```

### KeySource

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum KeySource {
    /// OS keychain
    /// - macOS: Keychain Access
    /// - Windows: Credential Manager
    /// - Linux: Secret Service (gnome-keyring/kwallet)
    OSKeychain {
        service: String,
        account: String,
    },

    /// Environment variable
    EnvVar(String),

    /// File path (should be secured by filesystem permissions)
    File(PathBuf),
}
```

### Examples

#### OS Keychain (Recommended)

```rust
EncryptionConfig {
    enabled: true,
    key_source: KeySource::OSKeychain {
        service: "cloudops".to_string(),
        account: "encryption_key".to_string(),
    },
}
```

**Benefits:**
- Automatic OS-level protection
- No key management in code
- Protected by user's OS login

#### Environment Variable

```rust
EncryptionConfig {
    enabled: true,
    key_source: KeySource::EnvVar("CLOUDOPS_ENCRYPTION_KEY".to_string()),
}
```

**Use When:**
- Testing/development
- Container deployments
- CI/CD environments

**Security Note:** Ensure environment variable is set securely.

#### File-Based Key

```rust
EncryptionConfig {
    enabled: true,
    key_source: KeySource::File(
        PathBuf::from("~/.cloudops/encryption.key")
    ),
}
```

**Use When:**
- Custom key management
- Shared keys across machines

**Security Note:** File should have restrictive permissions (0600).

## S3 Backup Configuration

### S3BackupConfig

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct S3BackupConfig {
    /// S3 bucket name
    pub bucket: String,

    /// AWS region
    pub region: String,

    /// Enable automatic sync
    pub auto_sync: bool,

    /// Backup interval in hours
    pub interval_hours: u32,

    /// Retention period in days
    pub retention_days: u32,
}
```

### Example

```rust
Some(S3BackupConfig {
    bucket: "my-cloudops-backup".to_string(),
    region: "us-west-2".to_string(),
    auto_sync: true,
    interval_hours: 24, // Daily backups
    retention_days: 30, // Keep for 30 days
})
```

### Retention Policies

**Local Snapshots:**
- Kept for 7 days automatically
- Cleaned up by `cleanup_old_snapshots()`

**S3 Snapshots:**
- Kept for `retention_days`
- Cleaned up during auto-backup cycle

### Disabling Backups

```rust
s3_backup: None // No S3 backups
```

## Configuration Loading

### From File

```rust
use std::fs;

let config_str = fs::read_to_string("~/.cloudops/config.json")?;
let config: StorageConfig = serde_json::from_str(&config_str)?;

let storage = StorageService::new(config).await?;
```

### From Environment

```rust
use std::env;

let config = StorageConfig {
    data_path: PathBuf::from(
        env::var("CLOUDOPS_DATA_PATH")
            .unwrap_or_else(|_| "~/.cloudops/data".to_string())
    ),
    // ... rest of config
};
```

### Default Configuration

```rust
impl Default for StorageConfig {
    fn default() -> Self {
        Self {
            data_path: PathBuf::from("~/.cloudops/data/"),
            qdrant: QdrantConfig {
                mode: QdrantMode::Embedded,
                storage_path: Some(PathBuf::from("~/.cloudops/data/qdrant/")),
                url: None,
            },
            embedding: EmbeddingConfig {
                provider: EmbeddingProvider::Server {
                    endpoint: "https://api.cloudops.example.com/embeddings".to_string(),
                    api_key: None,
                },
                model: "text-embedding-3-small".to_string(),
                dimension: 384,
                generation_mode: EmbeddingGenerationMode::OnWrite,
            },
            // ... default collections, encryption, no S3
        }
    }
}
```

## Validation

Configuration is validated on initialization:

```rust
impl StorageConfig {
    /// Validate configuration
    pub fn validate(&self) -> Result<(), ConfigError> {
        // Check paths exist
        if !self.data_path.exists() {
            return Err(ConfigError::InvalidPath(self.data_path.clone()));
        }

        // Check embedding dimension matches collection config
        if self.embedding.dimension != self.collections.cloud_estate.vector_size {
            return Err(ConfigError::DimensionMismatch);
        }

        // Validate Qdrant config
        match &self.qdrant.mode {
            QdrantMode::Embedded => {
                if self.qdrant.storage_path.is_none() {
                    return Err(ConfigError::MissingStoragePath);
                }
            }
            QdrantMode::Remote => {
                if self.qdrant.url.is_none() {
                    return Err(ConfigError::MissingUrl);
                }
            }
        }

        Ok(())
    }
}
```

## See Also

- [API Reference](api.md) - Complete API documentation
- [Collections](collections.md) - Collection schemas
- [Encryption](encryption.md) - Encryption details
- [Backup & Restore](backup-restore.md) - Backup strategies