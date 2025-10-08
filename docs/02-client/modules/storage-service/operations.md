# Common Operations

Practical examples and patterns for using the Storage Service.

## Overview

This guide provides code examples for common operations:
- Initializing the storage service
- Managing conversations
- Working with AWS resources
- Searching and filtering
- Backup and restore
- Error handling

## Initialization

### Basic Setup

```rust
use cloudops_storage_service::*;
use std::path::PathBuf;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Create configuration
    let config = StorageConfig {
        data_path: PathBuf::from("~/.cloudops/data/"),
        qdrant: QdrantConfig {
            mode: QdrantMode::Embedded,
            storage_path: Some(PathBuf::from("~/.cloudops/data/qdrant/")),
            url: None,
        },
        embedding: EmbeddingConfig {
            provider: EmbeddingProvider::Server {
                endpoint: "https://api.cloudops.example.com/embeddings".to_string(),
                api_key: Some(env::var("CLOUDOPS_API_KEY")?),
            },
            model: "text-embedding-3-small".to_string(),
            dimension: 384,
            generation_mode: EmbeddingGenerationMode::OnWrite,
        },
        collections: CollectionConfigs::default(),
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

    // Initialize storage service
    let storage = StorageService::new(config).await?;

    // Check health
    let health = storage.health().await?;
    println!("Storage initialized: {:?}", health);

    // Start automatic backups
    storage.backup.start_auto_backup().await?;

    Ok(())
}
```

### With Configuration File

```rust
use serde_json;
use std::fs;

async fn init_from_config() -> anyhow::Result<StorageService> {
    // Load config from file
    let config_path = PathBuf::from("~/.cloudops/config.json");
    let config_str = fs::read_to_string(config_path)?;
    let config: StorageConfig = serde_json::from_str(&config_str)?;

    // Initialize
    let storage = StorageService::new(config).await?;

    Ok(storage)
}
```

### With Default Configuration

```rust
async fn init_with_defaults() -> anyhow::Result<StorageService> {
    let mut config = StorageConfig::default();

    // Override specific settings
    config.s3_backup = Some(S3BackupConfig {
        bucket: env::var("S3_BACKUP_BUCKET")?,
        region: env::var("AWS_REGION")?,
        auto_sync: true,
        interval_hours: 12,
        retention_days: 30,
    });

    let storage = StorageService::new(config).await?;

    Ok(storage)
}
```

## Chat Operations

### Creating a Conversation

```rust
use uuid::Uuid;

async fn create_conversation(storage: &StorageService) -> anyhow::Result<String> {
    // Generate conversation ID
    let context_id = Uuid::new_v4().to_string();

    // Add initial system message
    storage.chat.append(&context_id, Message {
        role: MessageRole::System,
        content: "You are a helpful AWS operations assistant.".to_string(),
        metadata: None,
    }).await?;

    println!("Created conversation: {}", context_id);
    Ok(context_id)
}
```

### Adding Messages

```rust
async fn send_message(
    storage: &StorageService,
    context_id: &str,
    content: &str,
) -> anyhow::Result<()> {
    // Add user message
    storage.chat.append(context_id, Message {
        role: MessageRole::User,
        content: content.to_string(),
        metadata: Some(json!({
            "timestamp": Utc::now().to_rfc3339(),
        })),
    }).await?;

    // Simulate assistant response
    storage.chat.append(context_id, Message {
        role: MessageRole::Assistant,
        content: "I'll help you with that.".to_string(),
        metadata: Some(json!({
            "timestamp": Utc::now().to_rfc3339(),
        })),
    }).await?;

    Ok(())
}
```

### Retrieving History

```rust
async fn get_conversation(
    storage: &StorageService,
    context_id: &str,
) -> anyhow::Result<()> {
    // Get all messages
    let messages = storage.chat.get_history(context_id, None).await?;

    println!("Conversation: {}", context_id);
    for (i, msg) in messages.iter().enumerate() {
        println!("  [{}] {:?}: {}", i, msg.role, msg.content);
    }

    Ok(())
}
```

### Getting Recent Messages

```rust
async fn get_recent_messages(
    storage: &StorageService,
    context_id: &str,
    limit: usize,
) -> anyhow::Result<Vec<Message>> {
    // Get last N messages
    let messages = storage.chat.get_history(context_id, Some(limit)).await?;
    Ok(messages)
}
```

### Listing Conversations

```rust
async fn list_conversations(storage: &StorageService) -> anyhow::Result<()> {
    let context_ids = storage.chat.get_all_contexts().await?;

    println!("Active conversations: {}", context_ids.len());
    for id in context_ids {
        let messages = storage.chat.get_history(&id, Some(1)).await?;
        if let Some(first) = messages.first() {
            println!("  {}: {}", id, first.content.chars().take(50).collect::<String>());
        }
    }

    Ok(())
}
```

### Deleting a Conversation

```rust
async fn delete_conversation(
    storage: &StorageService,
    context_id: &str,
) -> anyhow::Result<()> {
    storage.chat.delete(context_id).await?;
    println!("Deleted conversation: {}", context_id);
    Ok(())
}
```

## AWS Resource Operations

### Adding a Resource

```rust
async fn add_resource(storage: &StorageService) -> anyhow::Result<()> {
    let resource = AWSResource {
        resource_type: "rds_instance".to_string(),
        identifier: "pg-instance-main1".to_string(),
        account_id: "123456789012".to_string(),
        region: "us-west-2".to_string(),
        arn: "arn:aws:rds:us-west-2:123456789012:db:pg-instance-main1".to_string(),
        name: "pg-instance-main1".to_string(),
        state: "available".to_string(),
        tags: [
            ("env".to_string(), "production".to_string()),
            ("app".to_string(), "main".to_string()),
        ].into(),
        permissions: vec![
            "rds:StopDBInstance".to_string(),
            "rds:StartDBInstance".to_string(),
        ],
        constraints: ResourceConstraints {
            can_stop: true,
            can_delete: false,
            has_dependencies: false,
            metadata: HashMap::new(),
        },
        metadata: json!({
            "engine": "postgres",
            "engine_version": "14.7",
            "instance_class": "db.t3.medium",
        }),
        last_synced: Utc::now(),
    };

    storage.estate.upsert_resource(resource).await?;
    println!("Added resource");
    Ok(())
}
```

### Bulk Import

```rust
async fn import_resources(
    storage: &StorageService,
    resources: Vec<AWSResource>,
) -> anyhow::Result<()> {
    let start = Instant::now();

    // Use bulk upsert for efficiency
    storage.estate.bulk_upsert(resources.clone()).await?;

    let duration = start.elapsed();
    println!("Imported {} resources in {:?}", resources.len(), duration);

    Ok(())
}
```

### Searching Resources

```rust
async fn search_resources(
    storage: &StorageService,
    query: &str,
) -> anyhow::Result<Vec<AWSResource>> {
    // Simple semantic search
    let results = storage.estate.search(query, None).await?;

    println!("Found {} resources for query: '{}'", results.len(), query);
    for resource in &results {
        println!("  - {} ({}, {})",
            resource.name,
            resource.resource_type,
            resource.state
        );
    }

    Ok(results)
}
```

### Searching with Filters

```rust
async fn search_with_filters(
    storage: &StorageService,
    query: &str,
) -> anyhow::Result<Vec<AWSResource>> {
    // Search with filters
    let results = storage.estate.search(
        query,
        Some(ResourceFilter {
            resource_type: Some("rds_instance".to_string()),
            region: Some("us-west-2".to_string()),
            state: Some("available".to_string()),
            tags: Some([
                ("env".to_string(), "production".to_string()),
            ].into()),
            ..Default::default()
        })
    ).await?;

    println!("Found {} matching resources", results.len());
    Ok(results)
}
```

### Getting Specific Resource

```rust
async fn get_resource(
    storage: &StorageService,
    resource_id: &str,
) -> anyhow::Result<Option<AWSResource>> {
    let resource = storage.estate.get_resource(resource_id).await?;

    match resource {
        Some(r) => {
            println!("Found: {} ({})", r.name, r.state);
            Ok(Some(r))
        }
        None => {
            println!("Resource not found: {}", resource_id);
            Ok(None)
        }
    }
}
```

### Listing by Filters

```rust
async fn list_by_account(
    storage: &StorageService,
    account_id: &str,
) -> anyhow::Result<Vec<AWSResource>> {
    let resources = storage.estate.get_by_account(account_id).await?;
    println!("Account {} has {} resources", account_id, resources.len());
    Ok(resources)
}

async fn list_by_region(
    storage: &StorageService,
    region: &str,
) -> anyhow::Result<Vec<AWSResource>> {
    let resources = storage.estate.get_by_region(region).await?;
    println!("Region {} has {} resources", region, resources.len());
    Ok(resources)
}

async fn list_by_type(
    storage: &StorageService,
    resource_type: &str,
) -> anyhow::Result<Vec<AWSResource>> {
    let resources = storage.estate.get_by_type(resource_type).await?;
    println!("Found {} resources of type {}", resources.len(), resource_type);
    Ok(resources)
}
```

### Updating Resource State

```rust
async fn update_resource_state(
    storage: &StorageService,
    resource_id: &str,
    new_state: &str,
) -> anyhow::Result<()> {
    // Get existing resource
    let mut resource = storage.estate.get_resource(resource_id).await?
        .ok_or_else(|| anyhow!("Resource not found"))?;

    // Update state
    resource.state = new_state.to_string();
    resource.last_synced = Utc::now();

    // Upsert (updates existing point)
    storage.estate.upsert_resource(resource).await?;

    println!("Updated resource {} state to {}", resource_id, new_state);
    Ok(())
}
```

### Deleting Resource

```rust
async fn delete_resource(
    storage: &StorageService,
    resource_id: &str,
) -> anyhow::Result<()> {
    storage.estate.delete_resource(resource_id).await?;
    println!("Deleted resource: {}", resource_id);
    Ok(())
}
```

## Sync Operations

### Full Sync

```rust
async fn full_sync(
    storage: &StorageService,
    current_resources: Vec<AWSResource>,
) -> anyhow::Result<()> {
    println!("Starting full sync: {} resources", current_resources.len());
    let start = Instant::now();

    // Get existing point IDs
    let existing_ids = storage.estate.get_all_point_ids().await?;
    println!("Existing points: {}", existing_ids.len());

    // Upsert all current resources
    storage.estate.bulk_upsert(current_resources.clone()).await?;
    println!("Upserted {} resources", current_resources.len());

    // Identify and delete stale resources
    let current_ids: HashSet<_> = current_resources
        .iter()
        .map(|r| format!("{}-{}-{}", r.account_id, r.region, r.identifier))
        .collect();

    let stale_ids: Vec<_> = existing_ids
        .into_iter()
        .filter(|id| !current_ids.contains(id))
        .collect();

    if !stale_ids.is_empty() {
        for id in &stale_ids {
            storage.estate.delete_resource(id).await?;
        }
        println!("Deleted {} stale resources", stale_ids.len());
    }

    let duration = start.elapsed();
    println!("Full sync completed in {:?}", duration);

    Ok(())
}
```

### Incremental Sync

```rust
async fn incremental_sync(
    storage: &StorageService,
    changed_resources: Vec<AWSResource>,
) -> anyhow::Result<()> {
    println!("Starting incremental sync: {} changed resources", changed_resources.len());

    // Upsert only changed resources
    storage.estate.bulk_upsert(changed_resources).await?;

    println!("Incremental sync completed");
    Ok(())
}
```

## Backup Operations

### Manual Backup

```rust
async fn create_backup(storage: &StorageService) -> anyhow::Result<()> {
    println!("Creating backup...");

    // Backup both collections
    let chat_snapshot = storage.backup.create_snapshot("chat_history").await?;
    println!("Chat history: {} ({} bytes)",
        chat_snapshot.name,
        chat_snapshot.size_bytes
    );

    let estate_snapshot = storage.backup.create_snapshot("aws_estate").await?;
    println!("AWS estate: {} ({} bytes)",
        estate_snapshot.name,
        estate_snapshot.size_bytes
    );

    // Upload to S3
    storage.backup.upload_to_s3(&chat_snapshot).await?;
    storage.backup.upload_to_s3(&estate_snapshot).await?;

    println!("Backup completed and uploaded to S3");
    Ok(())
}
```

### List Backups

```rust
async fn list_backups(storage: &StorageService) -> anyhow::Result<()> {
    println!("Available backups:");

    for collection in &["chat_history", "aws_estate"] {
        let snapshots = storage.backup.list_snapshots(collection).await?;

        println!("\n{}:", collection);
        for snapshot in snapshots {
            let location = match snapshot.location {
                SnapshotLocation::Local(_) => "Local",
                SnapshotLocation::S3 { .. } => "S3",
            };

            println!("  {} - {} ({} bytes) [{}]",
                snapshot.created_at.format("%Y-%m-%d %H:%M:%S"),
                snapshot.name,
                snapshot.size_bytes,
                location
            );
        }
    }

    Ok(())
}
```

### Restore from Backup

```rust
async fn restore_from_backup(
    storage: &StorageService,
    collection: &str,
    date: &str,
) -> anyhow::Result<()> {
    println!("Restoring {} from {}...", collection, date);

    // Restore from S3
    storage.backup.restore_from_s3(collection, date).await?;

    // Verify
    let health = storage.health().await?;
    for coll in health.collections {
        if coll.name == collection {
            println!("Restored: {} points", coll.points_count);
        }
    }

    Ok(())
}
```

## Error Handling

### Basic Error Handling

```rust
async fn operation_with_error_handling(
    storage: &StorageService,
) -> anyhow::Result<()> {
    match storage.estate.search("query", None).await {
        Ok(results) => {
            println!("Found {} results", results.len());
        }
        Err(e) => {
            eprintln!("Search failed: {}", e);
            // Handle error appropriately
            return Err(e);
        }
    }

    Ok(())
}
```

### Retry Logic

```rust
use std::time::Duration;

async fn operation_with_retry(
    storage: &StorageService,
    resource: AWSResource,
) -> anyhow::Result<()> {
    let mut attempts = 0;
    let max_attempts = 3;

    loop {
        match storage.estate.upsert_resource(resource.clone()).await {
            Ok(_) => {
                println!("Operation succeeded");
                return Ok(());
            }
            Err(e) => {
                attempts += 1;
                if attempts >= max_attempts {
                    return Err(anyhow!("Operation failed after {} attempts: {}", attempts, e));
                }

                eprintln!("Attempt {} failed: {}", attempts, e);
                tokio::time::sleep(Duration::from_secs(2u64.pow(attempts))).await;
            }
        }
    }
}
```

### Context-Rich Errors

```rust
use anyhow::Context;

async fn operation_with_context(
    storage: &StorageService,
    resource_id: &str,
) -> anyhow::Result<()> {
    let resource = storage.estate.get_resource(resource_id).await
        .context("Failed to retrieve resource")?
        .ok_or_else(|| anyhow!("Resource not found: {}", resource_id))
        .context("Resource lookup failed")?;

    println!("Found resource: {}", resource.name);
    Ok(())
}
```

## Health Monitoring

### Check Health

```rust
async fn check_health(storage: &StorageService) -> anyhow::Result<()> {
    let health = storage.health().await?;

    println!("Storage Health:");
    println!("  Qdrant: {}", if health.qdrant_connected { "✓" } else { "✗" });
    println!("  S3: {}", if health.s3_configured { "✓" } else { "✗" });
    println!("  Disk Space: {} MB", health.disk_space_mb);

    println!("\nCollections:");
    for collection in health.collections {
        println!("  {}: {} points ({})",
            collection.name,
            collection.points_count,
            collection.status
        );
    }

    Ok(())
}
```

### Monitor Performance

```rust
use std::time::Instant;

async fn monitor_operation_performance(
    storage: &StorageService,
    query: &str,
) -> anyhow::Result<()> {
    let start = Instant::now();

    let results = storage.estate.search(query, None).await?;

    let duration = start.elapsed();

    println!("Search completed:");
    println!("  Results: {}", results.len());
    println!("  Duration: {:?}", duration);

    if duration.as_millis() > 100 {
        eprintln!("Warning: Slow search (>{} ms)", duration.as_millis());
    }

    Ok(())
}
```

## Cleanup Operations

### Delete Old Conversations

```rust
async fn cleanup_old_conversations(
    storage: &StorageService,
    days: i64,
) -> anyhow::Result<()> {
    let cutoff = Utc::now() - chrono::Duration::days(days);
    println!("Deleting conversations older than {}", cutoff);

    let context_ids = storage.chat.get_all_contexts().await?;

    for context_id in context_ids {
        // Get first message timestamp
        let messages = storage.chat.get_history(&context_id, Some(1)).await?;

        if let Some(first) = messages.first() {
            // Check timestamp from metadata
            if let Some(timestamp_str) = first.metadata
                .as_ref()
                .and_then(|m| m["timestamp"].as_str())
            {
                if let Ok(timestamp) = DateTime::parse_from_rfc3339(timestamp_str) {
                    if timestamp.with_timezone(&Utc) < cutoff {
                        storage.chat.delete(&context_id).await?;
                        println!("Deleted conversation: {}", context_id);
                    }
                }
            }
        }
    }

    Ok(())
}
```

### Vacuum Operation

```rust
async fn vacuum_storage(storage: &StorageService) -> anyhow::Result<()> {
    println!("Starting storage vacuum...");

    // 1. Cleanup duplicates
    let duplicates = storage.estate.cleanup_duplicates().await?;
    println!("Removed {} duplicate resources", duplicates);

    // 2. Cleanup old snapshots
    storage.backup.cleanup_old_snapshots().await?;
    println!("Cleaned up old snapshots");

    // 3. Check health
    let health = storage.health().await?;
    println!("Storage health: {:?}", health);

    println!("Vacuum completed");
    Ok(())
}
```

## Graceful Shutdown

### Shutdown Sequence

```rust
async fn shutdown_storage(storage: StorageService) -> anyhow::Result<()> {
    println!("Shutting down storage...");

    // 1. Stop auto-backup
    storage.backup.stop_auto_backup().await?;
    println!("✓ Stopped auto-backup");

    // 2. Create final snapshot
    let snapshot = storage.backup.create_snapshot("chat_history").await?;
    println!("✓ Created final snapshot: {}", snapshot.name);

    // 3. Shutdown storage service
    storage.shutdown().await?;
    println!("✓ Storage shutdown complete");

    Ok(())
}
```

## See Also

- [API Reference](api.md) - Complete API documentation
- [Configuration](configuration.md) - Configuration options
- [Collections](collections.md) - Collection schemas
- [Point Management](point-management.md) - Point ID strategies
- [Testing](testing.md) - Testing examples