# Encryption Strategy

Application-level encryption for sensitive data in Qdrant storage.

## Overview

The Storage Service implements **application-level encryption** to protect sensitive data at rest. Data is encrypted before being stored in Qdrant and decrypted when retrieved.

### Goal

Make Qdrant's local files unreadable by:
- Encrypting message content
- Encrypting AWS resource metadata
- Keeping encryption keys secure in OS keychain

### What We Encrypt vs What We Don't

| Data Type | Encrypted | Reason |
|-----------|-----------|--------|
| Message content | ✅ Yes | Contains sensitive conversation data |
| Resource metadata | ✅ Yes | Contains ARNs, configurations, etc. |
| Vectors | ❌ No | Needed for semantic search |
| Filter fields | ❌ No | Needed for queries (context_id, region, etc.) |
| Point IDs | ❌ No | Public identifiers |

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Application Layer (Storage Service)            │
│                                                  │
│  Data → Encrypt → Store in Qdrant               │
│  Data ← Decrypt ← Retrieve from Qdrant          │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  Qdrant (stores encrypted payloads)             │
│  • Vectors: Unencrypted (needed for search)     │
│  • Filter fields: Unencrypted (needed for SQL)  │
│  • Content: Encrypted (base64 blob)             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  OS Keychain (stores encryption key)            │
│  • macOS: Keychain Access                       │
│  • Windows: Credential Manager                  │
│  • Linux: Secret Service                        │
└─────────────────────────────────────────────────┘
```

## Encryption Algorithm

### AES-256-GCM

**Algorithm**: AES-256-GCM (Advanced Encryption Standard, 256-bit key, Galois/Counter Mode)

**Benefits:**
- **Confidentiality**: Data is encrypted
- **Integrity**: Tampering is detected
- **Authentication**: Ensures data hasn't been modified
- **Performance**: Fast (hardware-accelerated on most CPUs)

**Key Properties:**
- **Key Size**: 256 bits (32 bytes)
- **Nonce Size**: 96 bits (12 bytes)
- **Tag Size**: 128 bits (16 bytes)

### Rust Implementation

```rust
use aes_gcm::{
    Aes256Gcm, Nonce, Key, KeyInit,
    aead::{Aead, OsRng, rand_core::RngCore},
};
use base64::{Engine as _, engine::general_purpose::STANDARD as BASE64};

pub struct Encryptor {
    cipher: Aes256Gcm,
}

impl Encryptor {
    /// Initialize from OS keychain
    pub fn new() -> Result<Self> {
        let key = Self::get_or_create_key_from_keychain()?;
        let cipher = Aes256Gcm::new(&key);
        Ok(Self { cipher })
    }

    /// Encrypt plaintext to base64 string
    pub fn encrypt(&self, plaintext: &str) -> Result<String> {
        // Generate random nonce
        let mut nonce_bytes = [0u8; 12];
        OsRng.fill_bytes(&mut nonce_bytes);
        let nonce = Nonce::from_slice(&nonce_bytes);

        // Encrypt
        let ciphertext = self.cipher
            .encrypt(nonce, plaintext.as_bytes())
            .map_err(|e| anyhow!("Encryption failed: {}", e))?;

        // Combine nonce + ciphertext, encode as base64
        let mut result = nonce_bytes.to_vec();
        result.extend_from_slice(&ciphertext);

        Ok(BASE64.encode(&result))
    }

    /// Decrypt base64 string to plaintext
    pub fn decrypt(&self, encrypted: &str) -> Result<String> {
        // Decode base64
        let decoded = BASE64.decode(encrypted)
            .map_err(|e| anyhow!("Base64 decode failed: {}", e))?;

        // Split nonce + ciphertext
        if decoded.len() < 12 {
            return Err(anyhow!("Invalid encrypted data: too short"));
        }

        let (nonce_bytes, ciphertext) = decoded.split_at(12);
        let nonce = Nonce::from_slice(nonce_bytes);

        // Decrypt
        let plaintext = self.cipher
            .decrypt(nonce, ciphertext)
            .map_err(|e| anyhow!("Decryption failed: {}", e))?;

        String::from_utf8(plaintext)
            .map_err(|e| anyhow!("UTF-8 decode failed: {}", e))
    }
}
```

### Encrypted Data Format

```
┌─────────────────────────────────────────────────────┐
│  Base64 String (stored in Qdrant)                   │
│  "xK9mP2vL8nQ5tY7rU2wE4hJ6kL3..."                   │
└─────────────────────────────────────────────────────┘
                    ↓ Base64 Decode
┌─────────────────────────────────────────────────────┐
│  Binary Data                                         │
│  [nonce: 12 bytes][ciphertext: variable][tag: 16 bytes]│
└─────────────────────────────────────────────────────┘
```

**Structure:**
1. **Nonce** (12 bytes): Random value, unique per encryption
2. **Ciphertext** (variable): Encrypted data
3. **Tag** (16 bytes): Authentication tag (included in ciphertext by AES-GCM)

## Key Management

### OS Keychain

The encryption key is stored in the operating system's secure keychain:

- **macOS**: Keychain Access (`security` command)
- **Windows**: Credential Manager (Windows Credential Vault)
- **Linux**: Secret Service (gnome-keyring, kwallet)

### Key Lifecycle

```rust
impl Encryptor {
    fn get_or_create_key_from_keychain() -> Result<Key<Aes256Gcm>> {
        let keyring = keyring::Entry::new("cloudops", "encryption_key")?;

        // Try to load existing key
        match keyring.get_password() {
            Ok(key_hex) => {
                // Key exists, decode and return
                let key_bytes = hex::decode(key_hex)?;
                Ok(Key::<Aes256Gcm>::from_slice(&key_bytes).clone())
            }
            Err(_) => {
                // Key doesn't exist, generate new one
                let mut key_bytes = [0u8; 32];
                OsRng.fill_bytes(&mut key_bytes);

                // Store in keychain
                let key_hex = hex::encode(&key_bytes);
                keyring.set_password(&key_hex)?;

                Ok(Key::<Aes256Gcm>::from_slice(&key_bytes).clone())
            }
        }
    }
}
```

**First Run:**
1. Check keychain for existing key
2. If not found, generate random 256-bit key
3. Store in keychain (hex-encoded)
4. Return key

**Subsequent Runs:**
1. Load key from keychain
2. Decode from hex
3. Return key

### Key Security

**Protected By:**
- User's OS login password
- OS-level encryption (FileVault, BitLocker, LUKS)
- Hardware security modules (on supported systems)

**Access Control:**
- Only accessible by the CloudOps application
- Cannot be accessed by other applications
- Protected by OS permissions

**Key Rotation:**

```rust
impl Encryptor {
    /// Rotate encryption key (re-encrypt all data)
    pub async fn rotate_key(
        &mut self,
        storage: &StorageService,
    ) -> Result<()> {
        // 1. Generate new key
        let mut new_key_bytes = [0u8; 32];
        OsRng.fill_bytes(&mut new_key_bytes);
        let new_key = Key::<Aes256Gcm>::from_slice(&new_key_bytes);
        let new_cipher = Aes256Gcm::new(new_key);

        // 2. Re-encrypt all data
        // (This is expensive - only do when necessary)
        let all_points = storage.get_all_points().await?;

        for point in all_points {
            // Decrypt with old key
            let plaintext = self.decrypt(&point.encrypted_data)?;

            // Encrypt with new key
            let new_encrypted = self.encrypt_with_cipher(&new_cipher, &plaintext)?;

            // Update point
            storage.update_point(point.id, new_encrypted).await?;
        }

        // 3. Store new key in keychain
        let keyring = keyring::Entry::new("cloudops", "encryption_key")?;
        keyring.set_password(&hex::encode(new_key_bytes))?;

        // 4. Update cipher
        self.cipher = new_cipher;

        Ok(())
    }
}
```

## Usage in Storage Service

### Chat Storage

```rust
impl ChatStorage {
    pub async fn append(&self, context_id: &str, message: Message) -> Result<()> {
        // Serialize message content
        let content_json = serde_json::to_string(&json!({
            "content": message.content,
            "metadata": message.metadata,
        }))?;

        // Encrypt
        let encrypted = self.encryptor.encrypt(&content_json)?;

        // Store in Qdrant
        self.qdrant.upsert_points(vec![Point {
            id: Uuid::new_v4().to_string(),
            vector: vec![0.0], // Dummy vector
            payload: json!({
                // Plain text (for filtering)
                "context_id": context_id,
                "message_index": message_index,
                "role": message.role.to_string(),
                "timestamp": Utc::now().timestamp(),

                // Encrypted content
                "encrypted_content": encrypted,
            }),
        }]).await?;

        Ok(())
    }

    pub async fn get_history(
        &self,
        context_id: &str,
        limit: Option<usize>,
    ) -> Result<Vec<Message>> {
        let results = self.qdrant.scroll(/* ... */).await?;

        // Decrypt all messages
        let messages = results.result.into_iter()
            .map(|point| {
                // Get encrypted content
                let encrypted = point.payload["encrypted_content"]
                    .as_str()
                    .ok_or_else(|| anyhow!("Missing encrypted_content"))?;

                // Decrypt
                let content_json = self.encryptor.decrypt(encrypted)?;
                let content: serde_json::Value = serde_json::from_str(&content_json)?;

                // Build message
                Ok(Message {
                    role: MessageRole::from_str(
                        point.payload["role"].as_str().unwrap()
                    )?,
                    content: content["content"].as_str().unwrap().to_string(),
                    metadata: content["metadata"].clone(),
                })
            })
            .collect::<Result<Vec<_>>>()?;

        Ok(messages)
    }
}
```

### Estate Storage

```rust
impl EstateStorage {
    pub async fn upsert_resource(&self, resource: AWSResource) -> Result<()> {
        // Separate encrypted vs plain fields
        let encrypted_data = json!({
            "identifier": resource.identifier,
            "arn": resource.arn,
            "name": resource.name,
            "engine": resource.metadata.get("engine"),
            "engine_version": resource.metadata.get("engine_version"),
            "permissions": resource.permissions,
            "constraints": resource.constraints,
            "metadata": resource.metadata,
        });

        // Encrypt sensitive data
        let encrypted = self.encryptor.encrypt(&encrypted_data.to_string())?;

        // Generate embedding from unencrypted searchable fields
        let embedding_text = format!(
            "{} {} {}",
            resource.resource_type,
            resource.name,
            resource.tags.iter()
                .map(|(k, v)| format!("{}: {}", k, v))
                .collect::<Vec<_>>()
                .join(" ")
        );
        let vector = self.embedder.embed(&embedding_text).await?;

        // Store in Qdrant
        let point_id = format!("{}-{}-{}",
            resource.account_id,
            resource.region,
            resource.identifier
        );

        self.qdrant.upsert_points(vec![Point {
            id: point_id,
            vector,
            payload: json!({
                // Plain text (for filtering)
                "resource_type": resource.resource_type,
                "account_id": resource.account_id,
                "region": resource.region,
                "state": resource.state,
                "last_synced": Utc::now().timestamp(),
                "tags": resource.tags,

                // Encrypted data
                "encrypted_data": encrypted,
            }),
        }]).await?;

        Ok(())
    }

    pub async fn get_resource(&self, resource_id: &str) -> Result<Option<AWSResource>> {
        let point = self.qdrant.get_point("aws_estate", resource_id).await?;

        if let Some(point) = point {
            // Decrypt
            let encrypted = point.payload["encrypted_data"]
                .as_str()
                .ok_or_else(|| anyhow!("Missing encrypted_data"))?;
            let decrypted = self.encryptor.decrypt(encrypted)?;
            let encrypted_data: serde_json::Value = serde_json::from_str(&decrypted)?;

            // Build resource
            let resource = AWSResource {
                resource_type: point.payload["resource_type"].as_str().unwrap().to_string(),
                account_id: point.payload["account_id"].as_str().unwrap().to_string(),
                region: point.payload["region"].as_str().unwrap().to_string(),
                state: point.payload["state"].as_str().unwrap().to_string(),

                // From encrypted data
                identifier: encrypted_data["identifier"].as_str().unwrap().to_string(),
                arn: encrypted_data["arn"].as_str().unwrap().to_string(),
                name: encrypted_data["name"].as_str().unwrap().to_string(),
                permissions: serde_json::from_value(encrypted_data["permissions"].clone())?,
                constraints: serde_json::from_value(encrypted_data["constraints"].clone())?,
                metadata: encrypted_data["metadata"].clone(),

                // Other fields...
            };

            Ok(Some(resource))
        } else {
            Ok(None)
        }
    }
}
```

## Performance Considerations

### Encryption Overhead

**AES-256-GCM is fast:**
- Encryption: ~1-2 GB/s (hardware-accelerated)
- Per-message overhead: <0.1ms for typical messages
- Negligible impact on overall performance

**Benchmarks:**
```rust
// Message: 1 KB
// Encrypt: ~0.05ms
// Decrypt: ~0.05ms

// Resource: 5 KB
// Encrypt: ~0.1ms
// Decrypt: ~0.1ms
```

### Optimization Tips

1. **Batch Encryption**: Encrypt multiple items before storing
2. **Lazy Decryption**: Only decrypt when displaying to user
3. **Cache Decrypted Data**: If showing same message multiple times

## Security Best Practices

### Do's

✅ **Use OS Keychain** for key storage
✅ **Generate random nonces** for each encryption
✅ **Verify authentication tags** during decryption (automatic with AES-GCM)
✅ **Rotate keys periodically** (e.g., annually)
✅ **Monitor failed decryption attempts**

### Don'ts

❌ **Never hardcode keys** in source code
❌ **Never log encryption keys** or plaintext sensitive data
❌ **Never reuse nonces** with the same key
❌ **Never store keys in environment variables** (production)
❌ **Never skip encryption** based on configuration (use encryption: true/false only for development)

## Backup Encryption

### Snapshots

**Qdrant snapshots contain:**
- Already-encrypted payloads (application-level)
- Unencrypted vectors and filters

**S3 Upload:**
1. Snapshot created (contains encrypted data)
2. Tar/gzip compression
3. Upload to S3
4. S3 server-side encryption (SSE-S3) applied

**Result: Double Encryption**
- Application-level: AES-256-GCM (our encryption)
- Storage-level: AES-256 (S3 SSE)

### Restore Process

```rust
impl BackupManager {
    pub async fn restore_from_s3(
        &self,
        collection: &str,
        date: &str,
    ) -> Result<()> {
        // 1. Download snapshot (encrypted by S3)
        let snapshot = self.download_from_s3(collection, date).await?;

        // 2. Restore to Qdrant (still contains our encrypted payloads)
        self.qdrant.restore_snapshot(collection, &snapshot).await?;

        // 3. Data is accessible (our encryption key from keychain)
        // No additional decryption step needed - happens on read

        Ok(())
    }
}
```

## Compliance & Audit

### Data at Rest

- **Qdrant files**: Application-level encryption (AES-256-GCM)
- **Snapshots**: Application-level + S3 SSE
- **Key storage**: OS keychain (protected by user login)

### Data in Transit

- **S3 uploads**: HTTPS (TLS 1.3)
- **Server API**: HTTPS (TLS 1.3)

### Audit Logging

```rust
impl Encryptor {
    pub fn encrypt(&self, plaintext: &str) -> Result<String> {
        let start = Instant::now();
        let result = self.encrypt_internal(plaintext);
        let duration = start.elapsed();

        // Log encryption event
        audit_log!(
            "encryption",
            success = result.is_ok(),
            duration_ms = duration.as_millis(),
            data_size_bytes = plaintext.len(),
        );

        result
    }

    pub fn decrypt(&self, encrypted: &str) -> Result<String> {
        let start = Instant::now();
        let result = self.decrypt_internal(encrypted);
        let duration = start.elapsed();

        // Log decryption event (including failures)
        audit_log!(
            "decryption",
            success = result.is_ok(),
            duration_ms = duration.as_millis(),
        );

        // Alert on repeated failures (possible tampering)
        if result.is_err() {
            security_alert!("Decryption failed: possible data corruption or tampering");
        }

        result
    }
}
```

## Troubleshooting

### Decryption Failures

**Common Causes:**
1. Key changed/corrupted in keychain
2. Data corrupted in Qdrant
3. Base64 decode failure
4. Wrong key used

**Solution:**
```rust
// Check key exists
let keyring = keyring::Entry::new("cloudops", "encryption_key")?;
match keyring.get_password() {
    Ok(_) => println!("Key exists"),
    Err(_) => println!("Key missing - will regenerate"),
}

// Validate encryption/decryption roundtrip
let test = "test data";
let encrypted = encryptor.encrypt(test)?;
let decrypted = encryptor.decrypt(&encrypted)?;
assert_eq!(test, decrypted);
```

### Key Recovery

If key is lost, data cannot be recovered (by design).

**Prevention:**
- Regular backups (snapshots include encrypted data)
- Export key for safekeeping (manual process):
  ```rust
  let keyring = keyring::Entry::new("cloudops", "encryption_key")?;
  let key_hex = keyring.get_password()?;
  // Save key_hex to secure location (password manager, etc.)
  ```

## See Also

- [Configuration](configuration.md) - EncryptionConfig settings
- [Collections](collections.md) - What gets encrypted in each collection
- [Backup & Restore](backup-restore.md) - Encrypted backup workflows
- [API Reference](api.md) - Encryptor API