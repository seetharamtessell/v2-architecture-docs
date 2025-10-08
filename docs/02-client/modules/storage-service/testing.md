# Testing Strategy

Comprehensive testing approaches for the Storage Service.

## Overview

Testing the Storage Service involves:
- Unit tests for individual components
- Integration tests for Qdrant operations
- End-to-end tests for complete workflows
- Performance tests for scalability
- Mock tests for external dependencies

## Test Structure

```
cloudops-storage-service/
├── src/
│   ├── lib.rs
│   ├── chat.rs
│   ├── estate.rs
│   ├── backup.rs
│   ├── encryption.rs
│   └── ...
└── tests/
    ├── unit/
    │   ├── encryption_tests.rs
    │   ├── point_id_tests.rs
    │   └── config_tests.rs
    ├── integration/
    │   ├── chat_tests.rs
    │   ├── estate_tests.rs
    │   ├── backup_tests.rs
    │   └── sync_tests.rs
    └── e2e/
        ├── full_workflow_tests.rs
        └── disaster_recovery_tests.rs
```

## Unit Tests

### Encryption Tests

```rust
#[cfg(test)]
mod encryption_tests {
    use super::*;

    #[test]
    fn test_encrypt_decrypt_roundtrip() {
        let encryptor = Encryptor::new_with_key(generate_test_key());

        let plaintext = "sensitive data";
        let encrypted = encryptor.encrypt(plaintext).unwrap();
        let decrypted = encryptor.decrypt(&encrypted).unwrap();

        assert_eq!(plaintext, decrypted);
    }

    #[test]
    fn test_encryption_produces_different_ciphertext() {
        let encryptor = Encryptor::new_with_key(generate_test_key());

        let plaintext = "test data";
        let encrypted1 = encryptor.encrypt(plaintext).unwrap();
        let encrypted2 = encryptor.encrypt(plaintext).unwrap();

        // Different nonces = different ciphertexts
        assert_ne!(encrypted1, encrypted2);

        // But both decrypt to same plaintext
        assert_eq!(encryptor.decrypt(&encrypted1).unwrap(), plaintext);
        assert_eq!(encryptor.decrypt(&encrypted2).unwrap(), plaintext);
    }

    #[test]
    fn test_decrypt_invalid_data() {
        let encryptor = Encryptor::new_with_key(generate_test_key());

        let result = encryptor.decrypt("invalid-base64-!@#$");
        assert!(result.is_err());
    }

    #[test]
    fn test_decrypt_with_wrong_key() {
        let encryptor1 = Encryptor::new_with_key(generate_test_key());
        let encryptor2 = Encryptor::new_with_key(generate_test_key());

        let encrypted = encryptor1.encrypt("test").unwrap();
        let result = encryptor2.decrypt(&encrypted);

        assert!(result.is_err());
    }

    fn generate_test_key() -> Key<Aes256Gcm> {
        let mut key_bytes = [0u8; 32];
        OsRng.fill_bytes(&mut key_bytes);
        Key::<Aes256Gcm>::from_slice(&key_bytes).clone()
    }
}
```

### Point ID Tests

```rust
#[cfg(test)]
mod point_id_tests {
    use super::*;

    #[test]
    fn test_generate_resource_id() {
        let resource = AWSResource {
            account_id: "123456789012".to_string(),
            region: "us-west-2".to_string(),
            identifier: "pg-instance-main1".to_string(),
            ..Default::default()
        };

        let id = generate_resource_id(&resource);

        assert_eq!(id, "123456789012-us-west-2-pg-instance-main1");
    }

    #[test]
    fn test_generate_resource_id_deterministic() {
        let resource = create_test_resource();

        let id1 = generate_resource_id(&resource);
        let id2 = generate_resource_id(&resource);

        assert_eq!(id1, id2);
    }

    #[test]
    fn test_different_resources_different_ids() {
        let resource1 = AWSResource {
            identifier: "instance-1".to_string(),
            ..create_test_resource()
        };

        let resource2 = AWSResource {
            identifier: "instance-2".to_string(),
            ..create_test_resource()
        };

        let id1 = generate_resource_id(&resource1);
        let id2 = generate_resource_id(&resource2);

        assert_ne!(id1, id2);
    }
}
```

### Configuration Tests

```rust
#[cfg(test)]
mod config_tests {
    use super::*;

    #[test]
    fn test_default_config() {
        let config = StorageConfig::default();

        assert!(config.encryption.enabled);
        assert_eq!(config.collections.chat_history.vector_size, 1);
        assert!(config.collections.aws_estate.vector_size > 1);
    }

    #[test]
    fn test_config_validation() {
        let mut config = StorageConfig::default();
        config.embedding.dimension = 384;
        config.collections.aws_estate.vector_size = 768;

        let result = config.validate();
        assert!(result.is_err());
    }

    #[test]
    fn test_config_serialization() {
        let config = StorageConfig::default();
        let json = serde_json::to_string(&config).unwrap();
        let deserialized: StorageConfig = serde_json::from_str(&json).unwrap();

        assert_eq!(config.encryption.enabled, deserialized.encryption.enabled);
    }
}
```

## Integration Tests

### Chat Storage Tests

```rust
#[cfg(test)]
mod chat_integration_tests {
    use super::*;

    #[tokio::test]
    async fn test_append_and_retrieve_message() {
        let storage = setup_test_storage().await;
        let context_id = Uuid::new_v4().to_string();

        // Append message
        storage.chat.append(&context_id, Message {
            role: MessageRole::User,
            content: "Hello".to_string(),
            metadata: None,
        }).await.unwrap();

        // Retrieve
        let messages = storage.chat.get_history(&context_id, None).await.unwrap();

        assert_eq!(messages.len(), 1);
        assert_eq!(messages[0].content, "Hello");
    }

    #[tokio::test]
    async fn test_message_ordering() {
        let storage = setup_test_storage().await;
        let context_id = Uuid::new_v4().to_string();

        // Append multiple messages
        for i in 0..5 {
            storage.chat.append(&context_id, Message {
                role: MessageRole::User,
                content: format!("Message {}", i),
                metadata: None,
            }).await.unwrap();
        }

        // Retrieve and verify order
        let messages = storage.chat.get_history(&context_id, None).await.unwrap();

        assert_eq!(messages.len(), 5);
        for (i, msg) in messages.iter().enumerate() {
            assert_eq!(msg.content, format!("Message {}", i));
        }
    }

    #[tokio::test]
    async fn test_conversation_isolation() {
        let storage = setup_test_storage().await;
        let ctx1 = Uuid::new_v4().to_string();
        let ctx2 = Uuid::new_v4().to_string();

        // Add messages to both contexts
        storage.chat.append(&ctx1, create_test_message("Context 1")).await.unwrap();
        storage.chat.append(&ctx2, create_test_message("Context 2")).await.unwrap();

        // Verify isolation
        let messages1 = storage.chat.get_history(&ctx1, None).await.unwrap();
        let messages2 = storage.chat.get_history(&ctx2, None).await.unwrap();

        assert_eq!(messages1.len(), 1);
        assert_eq!(messages2.len(), 1);
        assert_eq!(messages1[0].content, "Context 1");
        assert_eq!(messages2[0].content, "Context 2");
    }

    #[tokio::test]
    async fn test_delete_conversation() {
        let storage = setup_test_storage().await;
        let context_id = Uuid::new_v4().to_string();

        // Add messages
        for i in 0..3 {
            storage.chat.append(&context_id, create_test_message(&format!("Msg {}", i)))
                .await.unwrap();
        }

        // Delete conversation
        storage.chat.delete(&context_id).await.unwrap();

        // Verify deleted
        let messages = storage.chat.get_history(&context_id, None).await.unwrap();
        assert_eq!(messages.len(), 0);
    }
}
```

### Estate Storage Tests

```rust
#[cfg(test)]
mod estate_integration_tests {
    use super::*;

    #[tokio::test]
    async fn test_upsert_resource() {
        let storage = setup_test_storage().await;
        let resource = create_test_resource();

        // Upsert
        storage.estate.upsert_resource(resource.clone()).await.unwrap();

        // Retrieve
        let resource_id = format!("{}-{}-{}",
            resource.account_id,
            resource.region,
            resource.identifier
        );
        let retrieved = storage.estate.get_resource(&resource_id).await.unwrap();

        assert!(retrieved.is_some());
        assert_eq!(retrieved.unwrap().name, resource.name);
    }

    #[tokio::test]
    async fn test_upsert_updates_existing() {
        let storage = setup_test_storage().await;
        let mut resource = create_test_resource();

        // First upsert
        resource.state = "running".to_string();
        storage.estate.upsert_resource(resource.clone()).await.unwrap();

        // Update state
        resource.state = "stopped".to_string();
        storage.estate.upsert_resource(resource.clone()).await.unwrap();

        // Verify only one point exists with updated state
        let resource_id = format!("{}-{}-{}",
            resource.account_id,
            resource.region,
            resource.identifier
        );
        let retrieved = storage.estate.get_resource(&resource_id).await.unwrap().unwrap();

        assert_eq!(retrieved.state, "stopped");

        // Verify count (should be 1, not 2)
        let count = storage.estate.count().await.unwrap();
        assert_eq!(count, 1);
    }

    #[tokio::test]
    async fn test_search_with_filters() {
        let storage = setup_test_storage().await;

        // Add resources
        let resource1 = AWSResource {
            resource_type: "ec2_instance".to_string(),
            region: "us-west-2".to_string(),
            state: "running".to_string(),
            ..create_test_resource()
        };

        let resource2 = AWSResource {
            resource_type: "rds_instance".to_string(),
            region: "us-east-1".to_string(),
            state: "available".to_string(),
            ..create_test_resource()
        };

        storage.estate.bulk_upsert(vec![resource1, resource2]).await.unwrap();

        // Search with filter
        let results = storage.estate.search(
            "instance",
            Some(ResourceFilter {
                resource_type: Some("ec2_instance".to_string()),
                region: Some("us-west-2".to_string()),
                ..Default::default()
            })
        ).await.unwrap();

        assert_eq!(results.len(), 1);
        assert_eq!(results[0].resource_type, "ec2_instance");
    }

    #[tokio::test]
    async fn test_bulk_upsert() {
        let storage = setup_test_storage().await;

        // Create 100 resources
        let resources: Vec<_> = (0..100)
            .map(|i| AWSResource {
                identifier: format!("instance-{}", i),
                ..create_test_resource()
            })
            .collect();

        // Bulk upsert
        storage.estate.bulk_upsert(resources).await.unwrap();

        // Verify count
        let count = storage.estate.count().await.unwrap();
        assert_eq!(count, 100);
    }

    #[tokio::test]
    async fn test_delete_resource() {
        let storage = setup_test_storage().await;
        let resource = create_test_resource();
        let resource_id = format!("{}-{}-{}",
            resource.account_id,
            resource.region,
            resource.identifier
        );

        // Add resource
        storage.estate.upsert_resource(resource).await.unwrap();

        // Verify exists
        assert!(storage.estate.get_resource(&resource_id).await.unwrap().is_some());

        // Delete
        storage.estate.delete_resource(&resource_id).await.unwrap();

        // Verify deleted
        assert!(storage.estate.get_resource(&resource_id).await.unwrap().is_none());
    }
}
```

### Backup Tests

```rust
#[cfg(test)]
mod backup_integration_tests {
    use super::*;

    #[tokio::test]
    async fn test_create_snapshot() {
        let storage = setup_test_storage().await;

        // Add some data
        add_test_data(&storage).await;

        // Create snapshot
        let snapshot = storage.backup.create_snapshot("aws_estate").await.unwrap();

        assert!(snapshot.size_bytes > 0);
        assert_eq!(snapshot.collection, "aws_estate");
    }

    #[tokio::test]
    async fn test_backup_restore_cycle() {
        let storage = setup_test_storage().await;

        // Add data
        let resources = create_test_resources(10);
        storage.estate.bulk_upsert(resources.clone()).await.unwrap();
        let original_count = storage.estate.count().await.unwrap();

        // Create snapshot
        let snapshot = storage.backup.create_snapshot("aws_estate").await.unwrap();

        // Delete all data
        for resource in &resources {
            let id = format!("{}-{}-{}",
                resource.account_id,
                resource.region,
                resource.identifier
            );
            storage.estate.delete_resource(&id).await.unwrap();
        }
        assert_eq!(storage.estate.count().await.unwrap(), 0);

        // Restore
        storage.backup.restore_from_snapshot("aws_estate", &snapshot.name).await.unwrap();

        // Verify restored
        let restored_count = storage.estate.count().await.unwrap();
        assert_eq!(restored_count, original_count);
    }

    #[tokio::test]
    async fn test_list_snapshots() {
        let storage = setup_test_storage().await;

        // Create multiple snapshots
        for _ in 0..3 {
            storage.backup.create_snapshot("chat_history").await.unwrap();
            tokio::time::sleep(Duration::from_secs(1)).await;
        }

        // List snapshots
        let snapshots = storage.backup.list_snapshots("chat_history").await.unwrap();

        assert_eq!(snapshots.len(), 3);

        // Verify sorted by creation time (newest first)
        for i in 1..snapshots.len() {
            assert!(snapshots[i-1].created_at >= snapshots[i].created_at);
        }
    }
}
```

## End-to-End Tests

### Complete Workflow Test

```rust
#[cfg(test)]
mod e2e_tests {
    use super::*;

    #[tokio::test]
    async fn test_complete_conversation_workflow() {
        let storage = setup_test_storage().await;

        // 1. Create conversation
        let context_id = Uuid::new_v4().to_string();

        // 2. Add messages
        storage.chat.append(&context_id, Message {
            role: MessageRole::User,
            content: "Stop pg-instance-main1".to_string(),
            metadata: None,
        }).await.unwrap();

        // 3. Search for resource
        let resources = storage.estate.search("postgres", None).await.unwrap();
        assert!(resources.len() > 0);

        // 4. Add assistant response
        storage.chat.append(&context_id, Message {
            role: MessageRole::Assistant,
            content: "Stopping instance...".to_string(),
            metadata: Some(json!({
                "resources": [resources[0].identifier.clone()],
            })),
        }).await.unwrap();

        // 5. Update resource state
        let mut resource = resources[0].clone();
        resource.state = "stopped".to_string();
        storage.estate.upsert_resource(resource).await.unwrap();

        // 6. Verify final state
        let messages = storage.chat.get_history(&context_id, None).await.unwrap();
        assert_eq!(messages.len(), 2);

        let updated = storage.estate.search("postgres", Some(ResourceFilter {
            state: Some("stopped".to_string()),
            ..Default::default()
        })).await.unwrap();
        assert_eq!(updated.len(), 1);
    }

    #[tokio::test]
    async fn test_sync_workflow() {
        let storage = setup_test_storage().await;

        // 1. Initial sync (create resources)
        let resources = create_test_resources(50);
        sync_from_aws(&storage, resources.clone()).await.unwrap();
        assert_eq!(storage.estate.count().await.unwrap(), 50);

        // 2. Update sync (modify some resources)
        let mut updated_resources = resources.clone();
        for resource in &mut updated_resources[0..10] {
            resource.state = "stopped".to_string();
        }
        sync_from_aws(&storage, updated_resources.clone()).await.unwrap();
        assert_eq!(storage.estate.count().await.unwrap(), 50);  // Same count

        // 3. Delete sync (remove some resources)
        let remaining_resources: Vec<_> = updated_resources.into_iter()
            .skip(10)
            .collect();
        sync_from_aws(&storage, remaining_resources).await.unwrap();
        assert_eq!(storage.estate.count().await.unwrap(), 40);  // 10 deleted
    }

    #[tokio::test]
    async fn test_disaster_recovery() {
        let storage = setup_test_storage().await;

        // 1. Setup data
        add_test_data(&storage).await;
        let original_count = storage.estate.count().await.unwrap();

        // 2. Create backup
        let snapshot = storage.backup.create_snapshot("aws_estate").await.unwrap();

        // 3. Simulate data loss (delete collection)
        delete_all_data(&storage).await;
        assert_eq!(storage.estate.count().await.unwrap(), 0);

        // 4. Restore from backup
        storage.backup.restore_from_snapshot("aws_estate", &snapshot.name).await.unwrap();

        // 5. Verify recovery
        let recovered_count = storage.estate.count().await.unwrap();
        assert_eq!(recovered_count, original_count);
    }
}
```

## Performance Tests

### Scalability Tests

```rust
#[cfg(test)]
mod performance_tests {
    use super::*;

    #[tokio::test]
    async fn test_bulk_insert_performance() {
        let storage = setup_test_storage().await;
        let resource_counts = vec![100, 1000, 10000];

        for count in resource_counts {
            let resources = create_test_resources(count);
            let start = Instant::now();

            storage.estate.bulk_upsert(resources).await.unwrap();

            let duration = start.elapsed();
            println!("Inserted {} resources in {:?}", count, duration);

            // Should be under 5 seconds for 10k resources
            if count == 10000 {
                assert!(duration.as_secs() < 5);
            }
        }
    }

    #[tokio::test]
    async fn test_search_performance() {
        let storage = setup_test_storage().await;

        // Setup: 10k resources
        let resources = create_test_resources(10000);
        storage.estate.bulk_upsert(resources).await.unwrap();

        // Test search performance
        let queries = vec![
            "instance",
            "postgres database",
            "production ec2",
        ];

        for query in queries {
            let start = Instant::now();
            let results = storage.estate.search(query, None).await.unwrap();
            let duration = start.elapsed();

            println!("Search '{}': {} results in {:?}", query, results.len(), duration);

            // Should be under 100ms
            assert!(duration.as_millis() < 100);
        }
    }

    #[tokio::test]
    async fn test_concurrent_operations() {
        let storage = Arc::new(setup_test_storage().await);
        let mut handles = vec![];

        // Spawn 10 concurrent tasks
        for i in 0..10 {
            let storage_clone = storage.clone();
            let handle = tokio::spawn(async move {
                let resources = create_test_resources(100);
                storage_clone.estate.bulk_upsert(resources).await.unwrap();
            });
            handles.push(handle);
        }

        // Wait for all tasks
        for handle in handles {
            handle.await.unwrap();
        }

        // Verify all inserted
        let count = storage.estate.count().await.unwrap();
        assert_eq!(count, 1000);
    }
}
```

## Mock Tests

### Mock Qdrant Client

```rust
#[cfg(test)]
mod mock_tests {
    use mockall::predicate::*;
    use mockall::mock;

    mock! {
        QdrantClient {
            async fn upsert_points(&self, points: Vec<Point>) -> Result<()>;
            async fn search_points(&self, request: SearchPoints) -> Result<Vec<Point>>;
            async fn get_point(&self, id: &str) -> Result<Option<Point>>;
            async fn delete_points(&self, ids: &[String]) -> Result<()>;
        }
    }

    #[tokio::test]
    async fn test_with_mock_qdrant() {
        let mut mock_qdrant = MockQdrantClient::new();

        // Setup expectations
        mock_qdrant
            .expect_upsert_points()
            .times(1)
            .returning(|_| Ok(()));

        mock_qdrant
            .expect_search_points()
            .times(1)
            .returning(|_| Ok(vec![]));

        // Use mock in tests
        // ...
    }
}
```

## Test Utilities

### Setup Functions

```rust
#[cfg(test)]
mod test_utils {
    use super::*;

    pub async fn setup_test_storage() -> StorageService {
        let config = create_test_config();
        StorageService::new(config).await.unwrap()
    }

    pub fn create_test_config() -> StorageConfig {
        StorageConfig {
            data_path: PathBuf::from("/tmp/cloudops-test/"),
            qdrant: QdrantConfig {
                mode: QdrantMode::Embedded,
                storage_path: Some(PathBuf::from("/tmp/cloudops-test/qdrant/")),
                url: None,
            },
            embedding: EmbeddingConfig {
                provider: EmbeddingProvider::Fastembed {
                    model_path: None,
                },
                model: "all-MiniLM-L6-v2".to_string(),
                dimension: 384,
                generation_mode: EmbeddingGenerationMode::OnWrite,
            },
            collections: CollectionConfigs::default(),
            encryption: EncryptionConfig {
                enabled: false,  // Disable for tests
                key_source: KeySource::EnvVar("TEST_KEY".to_string()),
            },
            s3_backup: None,  // No S3 in tests
        }
    }

    pub fn create_test_resource() -> AWSResource {
        AWSResource {
            resource_type: "ec2_instance".to_string(),
            identifier: "i-1234567890abcdef0".to_string(),
            account_id: "123456789012".to_string(),
            region: "us-west-2".to_string(),
            arn: "arn:aws:ec2:us-west-2:123456789012:instance/i-1234567890abcdef0".to_string(),
            name: "test-instance".to_string(),
            state: "running".to_string(),
            tags: [("env".to_string(), "test".to_string())].into(),
            permissions: vec!["ec2:StopInstances".to_string()],
            constraints: ResourceConstraints::default(),
            metadata: json!({}),
            last_synced: Utc::now(),
        }
    }

    pub fn create_test_resources(count: usize) -> Vec<AWSResource> {
        (0..count)
            .map(|i| AWSResource {
                identifier: format!("instance-{}", i),
                name: format!("test-instance-{}", i),
                ..create_test_resource()
            })
            .collect()
    }

    pub fn create_test_message(content: &str) -> Message {
        Message {
            role: MessageRole::User,
            content: content.to_string(),
            metadata: None,
        }
    }

    pub async fn add_test_data(storage: &StorageService) {
        let resources = create_test_resources(10);
        storage.estate.bulk_upsert(resources).await.unwrap();
    }

    pub async fn cleanup_test_data() {
        tokio::fs::remove_dir_all("/tmp/cloudops-test/").await.ok();
    }
}
```

## Running Tests

### All Tests

```bash
cargo test
```

### Specific Test Suite

```bash
# Unit tests only
cargo test --lib

# Integration tests only
cargo test --test '*'

# Specific module
cargo test chat_integration_tests

# Specific test
cargo test test_backup_restore_cycle
```

### With Output

```bash
cargo test -- --nocapture
```

### Performance Tests

```bash
# Release mode for accurate performance
cargo test --release performance_tests
```

### Coverage

```bash
# Install tarpaulin
cargo install cargo-tarpaulin

# Run with coverage
cargo tarpaulin --out Html
```

## Continuous Integration

### GitHub Actions

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Setup Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable

      - name: Run tests
        run: cargo test --all-features

      - name: Run integration tests
        run: cargo test --test '*' --all-features

      - name: Check code coverage
        run: |
          cargo install cargo-tarpaulin
          cargo tarpaulin --out Xml

      - name: Upload coverage
        uses: codecov/codecov-action@v2
```

## Best Practices

### 1. Test Isolation

```rust
#[tokio::test]
async fn test_with_isolation() {
    // Each test gets its own storage
    let storage = setup_test_storage().await;

    // Test operations
    // ...

    // Cleanup happens automatically when storage drops
}
```

### 2. Deterministic Tests

```rust
// ✅ Good: Deterministic
#[test]
fn test_deterministic() {
    let resource = create_test_resource();
    let id1 = generate_resource_id(&resource);
    let id2 = generate_resource_id(&resource);
    assert_eq!(id1, id2);
}

// ❌ Bad: Non-deterministic
#[test]
fn test_non_deterministic() {
    let id = Uuid::new_v4().to_string();
    assert_eq!(id, "expected-uuid");  // Will fail!
}
```

### 3. Meaningful Assertions

```rust
// ✅ Good: Clear assertion
assert_eq!(
    messages.len(),
    3,
    "Expected 3 messages, got {}",
    messages.len()
);

// ❌ Bad: Unclear
assert!(messages.len() == 3);
```

### 4. Test Data Builders

```rust
pub struct ResourceBuilder {
    resource: AWSResource,
}

impl ResourceBuilder {
    pub fn new() -> Self {
        Self {
            resource: create_test_resource(),
        }
    }

    pub fn with_type(mut self, resource_type: &str) -> Self {
        self.resource.resource_type = resource_type.to_string();
        self
    }

    pub fn with_state(mut self, state: &str) -> Self {
        self.resource.state = state.to_string();
        self
    }

    pub fn build(self) -> AWSResource {
        self.resource
    }
}

// Usage
let resource = ResourceBuilder::new()
    .with_type("rds_instance")
    .with_state("available")
    .build();
```

## See Also

- [API Reference](api.md) - API to test
- [Operations](operations.md) - Operations to test
- [Configuration](configuration.md) - Test configuration
- [Collections](collections.md) - Collection behavior to test