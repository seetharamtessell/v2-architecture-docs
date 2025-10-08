# Backup & Restore

Complete backup and restore workflows for the Storage Service.

## Overview

The Storage Service provides automatic and manual backup capabilities with S3 integration. Backups are created using Qdrant's snapshot feature, with our own implementation for scheduling, uploading, and retention management.

### What Qdrant Provides

✅ **Snapshot Creation**: Creates tar archives of collections
✅ **Restore from Snapshot**: Restores collection from tar archive
✅ **Local Snapshot Storage**: Saves snapshots to filesystem

### What We Implement

✅ **Automatic Scheduling**: Periodic snapshot creation
✅ **S3 Upload**: Upload snapshots to S3 bucket
✅ **Retention Policies**: Clean up old snapshots (local and S3)
✅ **Encryption**: Snapshots contain already-encrypted data
✅ **Restoration Workflows**: Restore from local or S3

## Backup Architecture

```
┌─────────────────────────────────────────────────┐
│  BackupManager (Background Task)                │
│  • Creates snapshots every N hours              │
│  • Uploads to S3 (if configured)                │
│  • Cleans up old snapshots                      │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  Qdrant Snapshot (tar archive)                  │
│  • Path: ~/.cloudops/snapshots/                 │
│  • Format: {collection}_{timestamp}.snapshot    │
│  • Content: Collection data + metadata          │
└─────────────────────────────────────────────────┘
                    ↓ (if S3 configured)
┌─────────────────────────────────────────────────┐
│  S3 Bucket                                      │
│  • Path: {user_id}/snapshots/{collection}/      │
│  • Encrypted: SSE-S3                            │
│  • Retention: Configurable                      │
└─────────────────────────────────────────────────┘
```

## Backup Configuration

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

    /// Retention period in days (S3 only)
    pub retention_days: u32,
}
```

### Example Configuration

```rust
let config = StorageConfig {
    // ... other config
    s3_backup: Some(S3BackupConfig {
        bucket: "my-cloudops-backup".to_string(),
        region: "us-west-2".to_string(),
        auto_sync: true,
        interval_hours: 24,        // Daily backups
        retention_days: 30,        // Keep for 30 days
    }),
};
```

### Local Snapshot Retention

**Fixed Retention**: 7 days (not configurable)

Local snapshots are automatically cleaned up after 7 days to conserve disk space. S3 retention is configurable.

## Automatic Backup

### Starting Auto-Backup

```rust
// Start automatic backup scheduler
storage.backup.start_auto_backup().await?;
```

### Background Task Implementation

```rust
impl BackupManager {
    pub async fn start_auto_backup(&self) -> Result<()> {
        if !self.config.s3_backup.as_ref().map(|c| c.auto_sync).unwrap_or(false) {
            return Err(anyhow!("Auto-backup not configured"));
        }

        let config = self.config.clone();
        let qdrant = self.qdrant.clone();
        let s3_client = self.s3_client.clone();

        // Spawn background task
        let handle = tokio::spawn(async move {
            let interval = Duration::from_secs(
                config.s3_backup.unwrap().interval_hours as u64 * 3600
            );

            loop {
                // Wait for interval
                tokio::time::sleep(interval).await;

                // Backup both collections
                for collection in &["chat_history", "aws_estate"] {
                    match Self::backup_collection(
                        collection,
                        &qdrant,
                        &s3_client,
                        &config,
                    ).await {
                        Ok(snapshot) => {
                            info!("Backup created: {}", snapshot.name);
                        }
                        Err(e) => {
                            error!("Backup failed for {}: {}", collection, e);
                        }
                    }
                }

                // Cleanup old snapshots
                if let Err(e) = Self::cleanup_old_snapshots(&config).await {
                    error!("Cleanup failed: {}", e);
                }
            }
        });

        // Store handle for later cancellation
        *self.backup_handle.lock().await = Some(handle);

        Ok(())
    }

    pub async fn stop_auto_backup(&self) -> Result<()> {
        if let Some(handle) = self.backup_handle.lock().await.take() {
            handle.abort();
        }
        Ok(())
    }

    async fn backup_collection(
        collection: &str,
        qdrant: &QdrantClient,
        s3_client: &S3Client,
        config: &StorageConfig,
    ) -> Result<SnapshotInfo> {
        // 1. Create snapshot
        let snapshot = qdrant.create_snapshot(collection).await?;

        // 2. Upload to S3 (if configured)
        if let Some(s3_config) = &config.s3_backup {
            Self::upload_snapshot_to_s3(
                &snapshot,
                s3_client,
                s3_config,
                config.user_id,
            ).await?;
        }

        Ok(snapshot)
    }
}
```

### What Happens During Auto-Backup

1. **Wait** for configured interval
2. **Create** snapshots for both collections (chat_history, aws_estate)
3. **Upload** to S3 (if configured)
4. **Cleanup** old local snapshots (>7 days)
5. **Cleanup** old S3 snapshots (>retention_days)
6. **Repeat**

## Manual Backup

### Creating Snapshots

```rust
// Create snapshot for specific collection
let snapshot = storage.backup.create_snapshot("aws_estate").await?;
println!("Snapshot created: {}", snapshot.name);
println!("Size: {} bytes", snapshot.size_bytes);
println!("Location: {:?}", snapshot.location);
```

### Uploading to S3

```rust
// Upload existing snapshot
storage.backup.upload_to_s3(&snapshot).await?;
```

### Complete Manual Backup

```rust
// Create and upload in one step
async fn manual_backup(storage: &StorageService) -> Result<()> {
    // Backup chat history
    let chat_snapshot = storage.backup.create_snapshot("chat_history").await?;
    storage.backup.upload_to_s3(&chat_snapshot).await?;

    // Backup AWS estate
    let estate_snapshot = storage.backup.create_snapshot("aws_estate").await?;
    storage.backup.upload_to_s3(&estate_snapshot).await?;

    println!("Manual backup completed");
    Ok(())
}
```

## Snapshot Structure

### Snapshot Info

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SnapshotInfo {
    /// Snapshot name (e.g., "aws_estate_2025-10-08T10-30-00.snapshot")
    pub name: String,

    /// Collection name
    pub collection: String,

    /// Creation timestamp
    pub created_at: DateTime<Utc>,

    /// Size in bytes
    pub size_bytes: u64,

    /// Storage location
    pub location: SnapshotLocation,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SnapshotLocation {
    Local(PathBuf),
    S3 { bucket: String, key: String },
}
```

### Snapshot Naming Convention

**Format**: `{collection}_{timestamp}.snapshot`

**Examples:**
- `chat_history_2025-10-08T10-30-00.snapshot`
- `aws_estate_2025-10-09T14-45-30.snapshot`

**Timestamp Format**: ISO 8601 (`YYYY-MM-DDTHH-MM-SS`)

### S3 Path Structure

```
s3://{bucket}/{user_id}/snapshots/{collection}/{date}.snapshot

Example:
s3://my-cloudops-backup/user-123/snapshots/aws_estate/2025-10-08T10-30-00.snapshot
```

## Listing Snapshots

### List All Snapshots

```rust
// List snapshots for a collection (local + S3)
let snapshots = storage.backup.list_snapshots("aws_estate").await?;

for snapshot in snapshots {
    println!("{}: {} bytes ({})",
        snapshot.name,
        snapshot.size_bytes,
        match snapshot.location {
            SnapshotLocation::Local(_) => "local",
            SnapshotLocation::S3 { .. } => "S3",
        }
    );
}
```

### Implementation

```rust
impl BackupManager {
    pub async fn list_snapshots(&self, collection: &str) -> Result<Vec<SnapshotInfo>> {
        let mut snapshots = Vec::new();

        // 1. List local snapshots
        let local_path = self.config.data_path.join("snapshots");
        let pattern = format!("{}_*.snapshot", collection);

        for entry in glob::glob(&local_path.join(pattern).to_string_lossy())? {
            let path = entry?;
            let metadata = tokio::fs::metadata(&path).await?;

            snapshots.push(SnapshotInfo {
                name: path.file_name().unwrap().to_string_lossy().to_string(),
                collection: collection.to_string(),
                created_at: metadata.modified()?.into(),
                size_bytes: metadata.len(),
                location: SnapshotLocation::Local(path),
            });
        }

        // 2. List S3 snapshots (if configured)
        if let Some(s3_config) = &self.config.s3_backup {
            let s3_snapshots = self.list_s3_snapshots(collection).await?;
            snapshots.extend(s3_snapshots);
        }

        // Sort by creation time (newest first)
        snapshots.sort_by(|a, b| b.created_at.cmp(&a.created_at));

        Ok(snapshots)
    }

    async fn list_s3_snapshots(&self, collection: &str) -> Result<Vec<SnapshotInfo>> {
        let s3_config = self.config.s3_backup.as_ref()
            .ok_or_else(|| anyhow!("S3 not configured"))?;

        let prefix = format!("{}/snapshots/{}/",
            self.config.user_id,
            collection
        );

        let objects = self.s3_client
            .list_objects_v2()
            .bucket(&s3_config.bucket)
            .prefix(&prefix)
            .send()
            .await?;

        let snapshots = objects.contents()
            .iter()
            .map(|obj| SnapshotInfo {
                name: obj.key().unwrap()
                    .split('/')
                    .last()
                    .unwrap()
                    .to_string(),
                collection: collection.to_string(),
                created_at: obj.last_modified()
                    .unwrap()
                    .to_chrono_utc(),
                size_bytes: obj.size() as u64,
                location: SnapshotLocation::S3 {
                    bucket: s3_config.bucket.clone(),
                    key: obj.key().unwrap().to_string(),
                },
            })
            .collect();

        Ok(snapshots)
    }
}
```

## Restore Workflows

### Restore from Local Snapshot

```rust
// Restore collection from local snapshot
storage.backup.restore_from_snapshot(
    "aws_estate",
    "aws_estate_2025-10-08T10-30-00.snapshot"
).await?;
```

### Restore from S3

```rust
// Restore from S3 snapshot (by date)
storage.backup.restore_from_s3(
    "aws_estate",
    "2025-10-08T10-30-00"
).await?;
```

### Restore Latest

```rust
// Restore from latest available snapshot
let snapshots = storage.backup.list_snapshots("aws_estate").await?;
let latest = snapshots.first()
    .ok_or_else(|| anyhow!("No snapshots found"))?;

match &latest.location {
    SnapshotLocation::Local(_) => {
        storage.backup.restore_from_snapshot(
            &latest.collection,
            &latest.name
        ).await?;
    }
    SnapshotLocation::S3 { .. } => {
        let date = latest.name.strip_suffix(".snapshot").unwrap();
        storage.backup.restore_from_s3(
            &latest.collection,
            date
        ).await?;
    }
}
```

## Restore Implementation

### Restore from Local

```rust
impl BackupManager {
    pub async fn restore_from_snapshot(
        &self,
        collection: &str,
        snapshot_name: &str,
    ) -> Result<()> {
        let snapshot_path = self.config.data_path
            .join("snapshots")
            .join(snapshot_name);

        if !snapshot_path.exists() {
            return Err(anyhow!("Snapshot not found: {}", snapshot_name));
        }

        // 1. Delete existing collection
        self.qdrant.delete_collection(collection).await?;

        // 2. Restore from snapshot
        self.qdrant.restore_snapshot(RestoreSnapshot {
            collection_name: collection.to_string(),
            snapshot_path: snapshot_path.to_string_lossy().to_string(),
            priority: Some(SnapshotPriority::Snapshot),
        }).await?;

        // 3. Wait for collection to be ready
        self.wait_for_collection(collection).await?;

        info!("Restored {} from {}", collection, snapshot_name);
        Ok(())
    }

    async fn wait_for_collection(&self, collection: &str) -> Result<()> {
        for _ in 0..30 {
            match self.qdrant.collection_info(collection).await {
                Ok(info) if info.status == CollectionStatus::Green => {
                    return Ok(());
                }
                _ => {
                    tokio::time::sleep(Duration::from_secs(1)).await;
                }
            }
        }
        Err(anyhow!("Collection {} not ready after restore", collection))
    }
}
```

### Restore from S3

```rust
impl BackupManager {
    pub async fn restore_from_s3(
        &self,
        collection: &str,
        date: &str,
    ) -> Result<()> {
        let s3_config = self.config.s3_backup.as_ref()
            .ok_or_else(|| anyhow!("S3 not configured"))?;

        // 1. Construct S3 key
        let snapshot_name = format!("{}.snapshot", date);
        let s3_key = format!("{}/snapshots/{}/{}",
            self.config.user_id,
            collection,
            snapshot_name
        );

        // 2. Download snapshot to temp directory
        let temp_path = self.config.data_path
            .join("temp")
            .join(&snapshot_name);

        tokio::fs::create_dir_all(temp_path.parent().unwrap()).await?;

        let response = self.s3_client
            .get_object()
            .bucket(&s3_config.bucket)
            .key(&s3_key)
            .send()
            .await?;

        let data = response.body.collect().await?.into_bytes();
        tokio::fs::write(&temp_path, data).await?;

        info!("Downloaded snapshot from S3: {} bytes", temp_path.metadata()?.len());

        // 3. Restore from downloaded snapshot
        self.restore_from_snapshot(collection, &snapshot_name).await?;

        // 4. Cleanup temp file
        tokio::fs::remove_file(temp_path).await?;

        Ok(())
    }
}
```

## Cleanup Strategies

### Automatic Cleanup

Cleanup runs as part of the auto-backup cycle:

```rust
impl BackupManager {
    async fn cleanup_old_snapshots(&self) -> Result<()> {
        // 1. Cleanup local snapshots (>7 days)
        self.cleanup_local_snapshots().await?;

        // 2. Cleanup S3 snapshots (>retention_days)
        if self.config.s3_backup.is_some() {
            self.cleanup_s3_snapshots().await?;
        }

        Ok(())
    }

    async fn cleanup_local_snapshots(&self) -> Result<()> {
        let cutoff = Utc::now() - Duration::from_secs(7 * 24 * 60 * 60);
        let snapshots_path = self.config.data_path.join("snapshots");

        for entry in glob::glob(&snapshots_path.join("*.snapshot").to_string_lossy())? {
            let path = entry?;
            let metadata = tokio::fs::metadata(&path).await?;
            let modified: DateTime<Utc> = metadata.modified()?.into();

            if modified < cutoff {
                tokio::fs::remove_file(&path).await?;
                info!("Deleted old local snapshot: {:?}", path);
            }
        }

        Ok(())
    }

    async fn cleanup_s3_snapshots(&self) -> Result<()> {
        let s3_config = self.config.s3_backup.as_ref().unwrap();
        let cutoff = Utc::now() - Duration::from_secs(
            s3_config.retention_days as u64 * 24 * 60 * 60
        );

        for collection in &["chat_history", "aws_estate"] {
            let snapshots = self.list_s3_snapshots(collection).await?;

            for snapshot in snapshots {
                if snapshot.created_at < cutoff {
                    if let SnapshotLocation::S3 { bucket, key } = snapshot.location {
                        self.s3_client
                            .delete_object()
                            .bucket(&bucket)
                            .key(&key)
                            .send()
                            .await?;

                        info!("Deleted old S3 snapshot: {}", key);
                    }
                }
            }
        }

        Ok(())
    }
}
```

### Manual Cleanup

```rust
// Trigger manual cleanup
storage.backup.cleanup_old_snapshots().await?;
```

## Disaster Recovery

### Complete Data Loss

If local data is completely lost:

```rust
async fn disaster_recovery(storage: &mut StorageService) -> Result<()> {
    println!("Starting disaster recovery from S3...");

    // 1. List available snapshots
    let chat_snapshots = storage.backup.list_snapshots("chat_history").await?;
    let estate_snapshots = storage.backup.list_snapshots("aws_estate").await?;

    // 2. Get latest snapshots
    let latest_chat = chat_snapshots.first()
        .ok_or_else(|| anyhow!("No chat_history snapshots"))?;
    let latest_estate = estate_snapshots.first()
        .ok_or_else(|| anyhow!("No aws_estate snapshots"))?;

    println!("Latest chat snapshot: {} ({})",
        latest_chat.name,
        latest_chat.created_at
    );
    println!("Latest estate snapshot: {} ({})",
        latest_estate.name,
        latest_estate.created_at
    );

    // 3. Restore from S3
    let chat_date = latest_chat.name.strip_suffix(".snapshot").unwrap();
    storage.backup.restore_from_s3("chat_history", chat_date).await?;
    println!("✓ Chat history restored");

    let estate_date = latest_estate.name.strip_suffix(".snapshot").unwrap();
    storage.backup.restore_from_s3("aws_estate", estate_date).await?;
    println!("✓ AWS estate restored");

    // 4. Verify restoration
    let health = storage.health().await?;
    for collection in health.collections {
        println!("{}: {} points", collection.name, collection.points_count);
    }

    println!("Disaster recovery completed successfully");
    Ok(())
}
```

### Partial Data Corruption

If only one collection is corrupted:

```rust
// Restore only affected collection
storage.backup.restore_from_s3("chat_history", "latest").await?;
```

## Best Practices

### Backup Frequency

**Recommendations:**
- **Chat History**: Daily (low volume, high value)
- **AWS Estate**: Every 6-12 hours (changes frequently)
- **Manual**: Before major operations (bulk delete, migration)

### Testing Restores

Regularly test restore process:

```rust
#[cfg(test)]
mod tests {
    #[tokio::test]
    async fn test_backup_restore_cycle() -> Result<()> {
        let storage = setup_test_storage().await?;

        // 1. Create test data
        create_test_data(&storage).await?;
        let original_count = storage.estate.count().await?;

        // 2. Create backup
        let snapshot = storage.backup.create_snapshot("aws_estate").await?;

        // 3. Modify data
        storage.estate.delete_all().await?;
        assert_eq!(storage.estate.count().await?, 0);

        // 4. Restore
        storage.backup.restore_from_snapshot("aws_estate", &snapshot.name).await?;

        // 5. Verify
        let restored_count = storage.estate.count().await?;
        assert_eq!(restored_count, original_count);

        Ok(())
    }
}
```

### Monitoring

Monitor backup health:

```rust
async fn check_backup_health(storage: &StorageService) -> Result<BackupHealth> {
    let mut health = BackupHealth::default();

    // Check for recent backups
    for collection in &["chat_history", "aws_estate"] {
        let snapshots = storage.backup.list_snapshots(collection).await?;

        if let Some(latest) = snapshots.first() {
            let age = Utc::now() - latest.created_at;

            health.last_backup.insert(
                collection.to_string(),
                latest.created_at,
            );

            // Alert if backup is >48 hours old
            if age.num_hours() > 48 {
                health.warnings.push(format!(
                    "No recent backup for {}: {} hours old",
                    collection,
                    age.num_hours()
                ));
            }
        } else {
            health.warnings.push(format!("No backups found for {}", collection));
        }
    }

    Ok(health)
}

#[derive(Debug, Default)]
struct BackupHealth {
    last_backup: HashMap<String, DateTime<Utc>>,
    warnings: Vec<String>,
}
```

## Troubleshooting

### Backup Failures

**Common Issues:**

1. **Disk Space**: Not enough space for snapshot
   ```rust
   // Check available space before backup
   let available = fs4::available_space(&self.config.data_path)?;
   let required = self.estimate_snapshot_size(collection).await?;

   if available < required * 2 {
       return Err(anyhow!("Insufficient disk space"));
   }
   ```

2. **S3 Permissions**: Cannot upload to bucket
   ```rust
   // Test S3 permissions
   self.s3_client.head_bucket()
       .bucket(&s3_config.bucket)
       .send()
       .await?;
   ```

3. **Network Issues**: Upload interrupted
   ```rust
   // Retry with exponential backoff
   let mut attempts = 0;
   loop {
       match self.upload_to_s3(&snapshot).await {
           Ok(_) => break,
           Err(e) if attempts < 3 => {
               attempts += 1;
               tokio::time::sleep(Duration::from_secs(2u64.pow(attempts))).await;
           }
           Err(e) => return Err(e),
       }
   }
   ```

### Restore Failures

**Common Issues:**

1. **Snapshot Not Found**: Check snapshot name and location
2. **Corrupted Snapshot**: Download again from S3
3. **Collection Not Ready**: Increase wait timeout

## See Also

- [Configuration](configuration.md) - S3BackupConfig settings
- [API Reference](api.md) - BackupManager API
- [Operations](operations.md) - Common backup operations
- [Collections](collections.md) - What gets backed up