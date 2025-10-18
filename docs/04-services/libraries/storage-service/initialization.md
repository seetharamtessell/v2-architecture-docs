# Storage Service Initialization

Detailed documentation for Storage Service startup, shutdown, and recovery scenarios.

## Overview

The Storage Service is an **independent Rust module** that provides RAG (Retrieval-Augmented Generation) capabilities through Qdrant vector database. It can be used by:
- **Tauri frontend** (embedded Qdrant on user's laptop)
- **Python server** (remote Qdrant instance)
- **Playbook Service** (local RAG for playbooks)
- **Any module** that needs vector search and storage

**Key Scenarios Covered**:
- **Cold Start** - First-time initialization (no existing data)
- **Warm Start** - Normal startup with existing data
- **Crash Recovery** - Recovery after unexpected shutdown
- **Collection Management** - CRUD operations for collections

## Module Architecture

```
┌─────────────────────────────────────────────┐
│         Storage Service Module              │
│  (Independent Rust crate: storage-service)  │
├─────────────────────────────────────────────┤
│  - Collection CRUD                          │
│  - Point CRUD                               │
│  - Vector search                            │
│  - Encryption                               │
│  - Backup/Restore                           │
└─────────────────────────────────────────────┘
           │                    │
           ▼                    ▼
    ┌─────────────┐      ┌─────────────┐
    │   Embedded  │      │   Remote    │
    │   Qdrant    │      │   Qdrant    │
    │  (laptop)   │      │  (server)   │
    └─────────────┘      └─────────────┘
```

## Initialization Flow

### High-Level Sequence

```rust
pub async fn new(config: StorageConfig) -> Result<Self> {
    // 1. Validate configuration
    validate_config(&config)?;

    // 2. Initialize encryption (if enabled)
    let encryption = init_encryption(&config.encryption).await?;

    // 3. Initialize Qdrant client (embedded OR remote)
    let qdrant = init_qdrant(&config.qdrant).await?;

    // 4. Ensure collections exist (create if missing)
    ensure_collections(&qdrant, &config.collections).await?;

    // 5. Initialize embedding service
    let embedder = init_embedder(&config.embedding).await?;

    // 6. Run health check
    let health = check_health(&qdrant).await?;
    log::info!("Storage service initialized: {:?}", health);

    Ok(StorageService {
        qdrant,
        encryption,
        embedder,
        config,
    })
}
```

### 1. Configuration Validation

```rust
fn validate_config(config: &StorageConfig) -> Result<()> {
    // Validate Qdrant mode
    match &config.qdrant.mode {
        QdrantMode::Embedded => {
            let storage_path = config.qdrant.storage_path.as_ref()
                .ok_or_else(|| anyhow!("Embedded mode requires storage_path"))?;

            // Create directory if it doesn't exist
            if !storage_path.exists() {
                fs::create_dir_all(storage_path)
                    .context("Failed to create Qdrant storage directory")?;
            }
        }
        QdrantMode::Remote => {
            config.qdrant.url.as_ref()
                .ok_or_else(|| anyhow!("Remote mode requires url"))?;
        }
    }

    // Validate embedding config
    if config.embedding.dimension < 1 || config.embedding.dimension > 4096 {
        return Err(anyhow!("Invalid embedding dimension: {}", config.embedding.dimension));
    }

    Ok(())
}
```

### 2. Initialize Encryption

```rust
async fn init_encryption(config: &EncryptionConfig) -> Result<Option<EncryptionManager>> {
    if !config.enabled {
        log::warn!("Encryption is DISABLED");
        return Ok(None);
    }

    let key = match &config.key_source {
        KeySource::OSKeychain { service, account } => {
            // Try to retrieve existing key
            match keyring::Entry::new(service, account)?.get_password() {
                Ok(key) => {
                    log::info!("Retrieved encryption key from OS keychain");
                    decode_key(&key)?
                }
                Err(_) => {
                    // Generate new key
                    log::info!("Generating new encryption key");
                    let key = generate_encryption_key();

                    // Store in keychain
                    keyring::Entry::new(service, account)?
                        .set_password(&encode_key(&key))?;

                    key
                }
            }
        }
        KeySource::EnvVar(var_name) => {
            let encoded = env::var(var_name)
                .context("Encryption key not found in environment")?;
            decode_key(&encoded)?
        }
        KeySource::File(path) => {
            if !path.exists() {
                let key = generate_encryption_key();
                fs::write(path, encode_key(&key))?;
                key
            } else {
                let encoded = fs::read_to_string(path)?;
                decode_key(&encoded)?
            }
        }
    };

    Ok(Some(EncryptionManager::new(key)))
}
```

### 3. Initialize Qdrant Client

```rust
async fn init_qdrant(config: &QdrantConfig) -> Result<QdrantClient> {
    let client = match &config.mode {
        QdrantMode::Embedded => {
            let storage_path = config.storage_path.as_ref().unwrap();

            log::info!("Initializing embedded Qdrant: {:?}", storage_path);

            QdrantClient::new_embedded(storage_path.to_str().unwrap()).await?
        }
        QdrantMode::Remote => {
            let url = config.url.as_ref().unwrap();

            log::info!("Connecting to remote Qdrant: {}", url);

            QdrantClient::new(url).await?
        }
    };

    // Verify connection
    client.health_check().await
        .context("Qdrant health check failed")?;

    log::info!("Qdrant connection established");
    Ok(client)
}
```

### 4. Collection CRUD Operations

#### Create Collection

```rust
pub async fn create_collection(
    &self,
    name: &str,
    config: &CollectionConfig,
) -> Result<()> {
    log::info!("Creating collection: {}", name);

    // Create collection with vector config
    self.qdrant.create_collection(
        name,
        VectorsConfig {
            size: config.vector_size,
            distance: config.distance,
        },
        config.hnsw_config.clone(),
    ).await?;

    // Create indexes for indexed fields
    for field in &config.indexed_fields {
        self.qdrant.create_field_index(
            name,
            &field.name,
            field.field_type,
        ).await?;
    }

    log::info!("Collection {} created successfully", name);
    Ok(())
}
```

#### Read Collection Info

```rust
pub async fn get_collection_info(&self, name: &str) -> Result<CollectionInfo> {
    let info = self.qdrant.get_collection_info(name).await?;

    Ok(CollectionInfo {
        name: info.name,
        points_count: info.points_count,
        vector_size: info.config.params.vectors.size,
        distance: info.config.params.vectors.distance,
        status: info.status,
        indexed_fields_count: info.payload_schema.len(),
    })
}
```

#### List All Collections

```rust
pub async fn list_collections(&self) -> Result<Vec<String>> {
    let collections = self.qdrant.list_collections().await?;
    Ok(collections.iter().map(|c| c.name.clone()).collect())
}
```

#### Update Collection (Re-index)

```rust
pub async fn update_collection_indexes(
    &self,
    name: &str,
    fields: Vec<IndexedField>,
) -> Result<()> {
    log::info!("Updating indexes for collection: {}", name);

    for field in fields {
        // Qdrant will create index if it doesn't exist
        self.qdrant.create_field_index(
            name,
            &field.name,
            field.field_type,
        ).await?;
    }

    log::info!("Indexes updated for {}", name);
    Ok(())
}
```

#### Delete Collection

```rust
pub async fn delete_collection(&self, name: &str) -> Result<()> {
    log::warn!("Deleting collection: {}", name);

    self.qdrant.delete_collection(name).await?;

    log::info!("Collection {} deleted", name);
    Ok(())
}
```

### 5. Ensure Collections Exist

```rust
async fn ensure_collections(
    qdrant: &QdrantClient,
    configs: &CollectionConfigs,
) -> Result<()> {
    log::info!("Ensuring collections exist...");

    // Get existing collections
    let existing = qdrant.list_collections().await?;
    let existing_names: HashSet<_> = existing.iter()
        .map(|c| c.name.as_str())
        .collect();

    // Chat history collection
    if !existing_names.contains("chat_history") {
        log::info!("Creating chat_history collection");
        create_collection(qdrant, "chat_history", &configs.chat_history).await?;
    } else {
        log::info!("chat_history collection exists");
    }

    // Cloud estate collection
    if !existing_names.contains("cloud_estate") {
        log::info!("Creating cloud_estate collection");
        create_collection(qdrant, "cloud_estate", &configs.cloud_estate).await?;
    } else {
        log::info!("cloud_estate collection exists");
    }

    // User playbooks collection
    if !existing_names.contains("user_playbooks") {
        log::info!("Creating user_playbooks collection");
        create_collection(qdrant, "user_playbooks", &configs.user_playbooks).await?;
    } else {
        log::info!("user_playbooks collection exists");
    }

    log::info!("All collections ready");
    Ok(())
}

async fn create_collection(
    qdrant: &QdrantClient,
    name: &str,
    config: &CollectionConfig,
) -> Result<()> {
    // Create collection
    qdrant.create_collection(
        name,
        VectorsConfig {
            size: config.vector_size,
            distance: config.distance,
        },
        config.hnsw_config.clone(),
    ).await?;

    // Create indexes
    for field in &config.indexed_fields {
        qdrant.create_field_index(name, &field.name, field.field_type).await?;
    }

    Ok(())
}
```

## Startup Scenarios

### 1. Cold Start (First-Time Initialization)

**Scenario**: Module is initialized for the first time.

**What Happens**:
1. Qdrant storage directory doesn't exist → Create directory
2. Initialize empty Qdrant instance
3. Collections don't exist → Create all configured collections
4. Encryption key doesn't exist → Generate and store new key
5. Ready to accept operations

**Example**:
```rust
// First run
let config = StorageConfig {
    qdrant: QdrantConfig {
        mode: QdrantMode::Embedded,
        storage_path: Some(PathBuf::from("~/.escher/qdrant/")),
        url: None,
    },
    collections: CollectionConfigs::default(),
    encryption: EncryptionConfig {
        enabled: true,
        key_source: KeySource::OSKeychain {
            service: "escher".to_string(),
            account: "storage_key".to_string(),
        },
    },
    embedding: EmbeddingConfig {
        provider: EmbeddingProvider::Server {
            endpoint: "https://api.escher.com/embeddings".to_string(),
            api_key: Some(env::var("ESCHER_API_KEY")?),
        },
        model: "text-embedding-3-small".to_string(),
        dimension: 384,
    },
};

let storage = StorageService::new(config).await?;

// Result:
// - ~/.escher/qdrant/ created
// - 3 collections created (0 points each)
// - Encryption key generated and stored
```

**Validation**:
```rust
let collections = storage.list_collections().await?;
assert_eq!(collections.len(), 3);
assert!(collections.contains(&"chat_history".to_string()));
assert!(collections.contains(&"cloud_estate".to_string()));
assert!(collections.contains(&"user_playbooks".to_string()));

let info = storage.get_collection_info("chat_history").await?;
assert_eq!(info.points_count, 0);
```

**Performance**:
- Embedded mode: 2-5 seconds (create collections, initialize Qdrant)
- Remote mode: 500ms-1s (network latency + collection creation)

### 2. Warm Start (Normal Startup)

**Scenario**: Module is initialized with existing data.

**What Happens**:
1. Qdrant storage directory exists → Load existing database
2. Collections exist → Skip creation
3. Encryption key exists → Load from keychain
4. Data is accessible immediately
5. Ready to accept operations

**Example**:
```rust
// Subsequent runs
let config = StorageConfig::default();
let storage = StorageService::new(config).await?;

// Existing data is immediately available
let chat_info = storage.get_collection_info("chat_history").await?;
println!("Loaded chat_history with {} messages", chat_info.points_count);

let estate_info = storage.get_collection_info("cloud_estate").await?;
println!("Loaded cloud_estate with {} resources", estate_info.points_count);
```

**Performance**:
- Embedded mode (10k points): 200-500ms
- Embedded mode (100k points): 500ms-1s
- Remote mode: 100-300ms

### 3. Crash Recovery

**Scenario**: Module crashed or was terminated unexpectedly.

**Detection**:
```rust
async fn detect_crash(storage_path: &Path) -> Result<bool> {
    // Check for Qdrant lock files
    let lock_file = storage_path.join(".lock");

    if lock_file.exists() {
        log::warn!("Detected crash: lock file exists");
        // Qdrant will clean this up on startup
        return Ok(true);
    }

    Ok(false)
}
```

**Recovery**:
```rust
async fn init_with_recovery(config: StorageConfig) -> Result<StorageService> {
    if let Some(storage_path) = &config.qdrant.storage_path {
        if detect_crash(storage_path).await? {
            log::info!("Performing crash recovery...");
            // Qdrant's embedded mode handles recovery automatically
        }
    }

    // Normal initialization
    let storage = StorageService::new(config).await?;

    // Verify collection integrity
    for collection in &["chat_history", "cloud_estate", "user_playbooks"] {
        let info = storage.get_collection_info(collection).await?;
        log::info!("Collection {} recovered: {} points", collection, info.points_count);
    }

    Ok(storage)
}
```

**Recovery Guarantees**:
- **Qdrant handles crash recovery automatically** - WAL (Write-Ahead Log) ensures durability
- **Committed operations are never lost** - ACID guarantees
- **Pending operations are rolled back** - No partial writes
- **Collections remain consistent** - Indexes are rebuilt if needed

**Example**:
```rust
// After crash, next startup:
let storage = StorageService::new(config).await?;

// All committed data is available
let info = storage.get_collection_info("chat_history").await?;
println!("Recovered {} messages", info.points_count);
```

## Point CRUD Operations

### Create (Upsert) Points

```rust
pub async fn upsert_point(
    &self,
    collection: &str,
    point_id: String,
    vector: Vec<f32>,
    payload: serde_json::Value,
) -> Result<()> {
    // Encrypt payload if encryption enabled
    let encrypted_payload = if let Some(enc) = &self.encryption {
        enc.encrypt_payload(&payload)?
    } else {
        payload
    };

    // Upsert point
    self.qdrant.upsert_points(
        collection,
        vec![PointStruct {
            id: point_id.into(),
            vector: vector.into(),
            payload: encrypted_payload.into(),
        }],
    ).await?;

    Ok(())
}

pub async fn bulk_upsert(
    &self,
    collection: &str,
    points: Vec<(String, Vec<f32>, serde_json::Value)>,
) -> Result<()> {
    let mut point_structs = Vec::new();

    for (id, vector, payload) in points {
        let encrypted_payload = if let Some(enc) = &self.encryption {
            enc.encrypt_payload(&payload)?
        } else {
            payload
        };

        point_structs.push(PointStruct {
            id: id.into(),
            vector: vector.into(),
            payload: encrypted_payload.into(),
        });
    }

    self.qdrant.upsert_points(collection, point_structs).await?;

    Ok(())
}
```

### Read Points

```rust
pub async fn get_point(
    &self,
    collection: &str,
    point_id: &str,
) -> Result<Option<Point>> {
    let points = self.qdrant.get_points(
        collection,
        &[point_id.into()],
        Some(true), // with_payload
        Some(true), // with_vector
    ).await?;

    if let Some(point) = points.first() {
        // Decrypt payload if encryption enabled
        let payload = if let Some(enc) = &self.encryption {
            enc.decrypt_payload(&point.payload)?
        } else {
            point.payload.clone()
        };

        Ok(Some(Point {
            id: point.id.clone(),
            vector: point.vector.clone(),
            payload,
        }))
    } else {
        Ok(None)
    }
}

pub async fn search_points(
    &self,
    collection: &str,
    query_vector: Vec<f32>,
    filter: Option<Filter>,
    limit: usize,
) -> Result<Vec<ScoredPoint>> {
    let results = self.qdrant.search_points(SearchPoints {
        collection_name: collection.to_string(),
        vector: query_vector,
        filter,
        limit: limit as u64,
        with_payload: Some(WithPayloadSelector::enable(true)),
        with_vector: Some(WithVectorSelector::enable(true)),
        ..Default::default()
    }).await?;

    // Decrypt payloads
    let mut decrypted_results = Vec::new();
    for result in results {
        let payload = if let Some(enc) = &self.encryption {
            enc.decrypt_payload(&result.payload)?
        } else {
            result.payload
        };

        decrypted_results.push(ScoredPoint {
            id: result.id,
            score: result.score,
            vector: result.vector,
            payload,
        });
    }

    Ok(decrypted_results)
}

pub async fn scroll_points(
    &self,
    collection: &str,
    filter: Option<Filter>,
    limit: Option<usize>,
) -> Result<Vec<Point>> {
    let results = self.qdrant.scroll(Scroll {
        collection_name: collection.to_string(),
        filter,
        limit: limit.map(|l| l as u32),
        with_payload: Some(WithPayloadSelector::enable(true)),
        with_vector: Some(WithVectorSelector::enable(true)),
        ..Default::default()
    }).await?;

    // Decrypt payloads
    let mut decrypted_points = Vec::new();
    for point in results.points {
        let payload = if let Some(enc) = &self.encryption {
            enc.decrypt_payload(&point.payload)?
        } else {
            point.payload
        };

        decrypted_points.push(Point {
            id: point.id,
            vector: point.vector,
            payload,
        });
    }

    Ok(decrypted_points)
}
```

### Delete Points

```rust
pub async fn delete_point(
    &self,
    collection: &str,
    point_id: &str,
) -> Result<()> {
    self.qdrant.delete_points(
        collection,
        vec![point_id.into()],
    ).await?;

    Ok(())
}

pub async fn delete_points_by_filter(
    &self,
    collection: &str,
    filter: Filter,
) -> Result<u64> {
    let result = self.qdrant.delete_points_by_filter(
        collection,
        filter,
    ).await?;

    Ok(result.deleted_count)
}
```

## Usage Examples

### Example 1: Tauri Frontend (Embedded Mode)

```rust
// In Tauri app initialization
use storage_service::{StorageService, StorageConfig, QdrantMode};

#[tauri::command]
async fn init_storage() -> Result<(), String> {
    let config = StorageConfig {
        qdrant: QdrantConfig {
            mode: QdrantMode::Embedded,
            storage_path: Some(PathBuf::from("~/.escher/qdrant/")),
            url: None,
        },
        // ... rest of config
    };

    let storage = StorageService::new(config).await
        .map_err(|e| e.to_string())?;

    // Store in Tauri state
    app.manage(storage);

    Ok(())
}

#[tauri::command]
async fn search_resources(
    storage: tauri::State<'_, StorageService>,
    query: String,
) -> Result<Vec<Resource>, String> {
    let vector = storage.embedder.embed(&query).await
        .map_err(|e| e.to_string())?;

    let results = storage.search_points("cloud_estate", vector, None, 10).await
        .map_err(|e| e.to_string())?;

    Ok(results.into_iter().map(|p| parse_resource(p.payload)).collect())
}
```

### Example 2: Python Server (Remote Mode)

```rust
// In Rust service used by Python server
use storage_service::{StorageService, StorageConfig, QdrantMode};

pub async fn init_server_storage() -> Result<StorageService> {
    let config = StorageConfig {
        qdrant: QdrantConfig {
            mode: QdrantMode::Remote,
            storage_path: None,
            url: Some("http://qdrant-server:6333".to_string()),
        },
        // ... rest of config
    };

    StorageService::new(config).await
}
```

### Example 3: Playbook Service (Local RAG)

```rust
use storage_service::{StorageService, StorageConfig};

pub struct PlaybookService {
    storage: Arc<StorageService>,
}

impl PlaybookService {
    pub async fn new() -> Result<Self> {
        let config = StorageConfig {
            qdrant: QdrantConfig {
                mode: QdrantMode::Embedded,
                storage_path: Some(PathBuf::from("~/.escher/playbooks/qdrant/")),
                url: None,
            },
            // ... playbook-specific config
        };

        let storage = Arc::new(StorageService::new(config).await?);

        Ok(Self { storage })
    }

    pub async fn search_playbooks(&self, query: &str) -> Result<Vec<Playbook>> {
        let vector = self.storage.embedder.embed(query).await?;

        let results = self.storage.search_points(
            "user_playbooks",
            vector,
            None,
            10,
        ).await?;

        Ok(results.into_iter().map(|p| parse_playbook(p.payload)).collect())
    }
}
```

## Shutdown

```rust
pub async fn shutdown(&self) -> Result<()> {
    log::info!("Shutting down storage service...");

    // Flush any pending operations
    self.qdrant.flush_all().await?;

    // Close connection (for remote mode)
    self.qdrant.close().await?;

    log::info!("Storage service shutdown complete");
    Ok(())
}
```

## Best Practices

1. **Always check health after initialization**
```rust
let storage = StorageService::new(config).await?;
let info = storage.get_collection_info("chat_history").await?;
log::info!("Initialized with {} existing messages", info.points_count);
```

2. **Use bulk operations for performance**
```rust
// ✅ Good
storage.bulk_upsert("cloud_estate", resources).await?;

// ❌ Bad
for resource in resources {
    storage.upsert_point("cloud_estate", id, vector, payload).await?;
}
```

3. **Handle encryption key carefully**
```rust
// Prefer OS keychain for automatic management
KeySource::OSKeychain {
    service: "escher".to_string(),
    account: "storage_key".to_string(),
}
```

4. **Graceful shutdown on app exit**
```rust
storage.shutdown().await?;
```

## See Also

- [API Reference](api.md) - Complete API documentation
- [Configuration](configuration.md) - Configuration options
- [Collections](collections.md) - Collection schemas
- [Operations](operations.md) - Common operation patterns
- [Backup & Restore](backup-restore.md) - Backup strategies
