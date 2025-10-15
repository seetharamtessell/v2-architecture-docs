# Playbook Service

**Crate Name**: `escher-playbook-service`
**Status**: Design Phase 🚧
**Location**: Runs on client (laptop or user's cloud runtime)

---

## Overview

The Playbook Service is a Rust-based service that manages playbook storage, synchronization, and execution on the client side. It handles:

1. **Local playbook storage** (user's custom playbooks)
2. **Metadata publishing** to server tenant space
3. **Script management** (download, cache, lifecycle)
4. **Local RAG** (6th collection: `user_playbooks`)

---

## Responsibilities

### 1. Local Playbook Storage
- Store user's custom playbooks in `~/.escher/playbooks/`
- Organize by playbook ID and version
- Support 4 execution formats: shell, CloudFormation/ARM, Terraform, Python
- Encrypt sensitive data at rest

### 2. Metadata Publishing
- Extract metadata from local playbooks
- Publish to server's tenant-specific RAG collection
- Incremental sync (only changed playbooks)
- Background sync every 5 minutes
- Immediate sync on create/update/delete

### 3. Script Management
- Download scripts from server (Escher Library or Tenant Storage)
- Cache downloaded scripts locally (`~/.escher/cache/scripts/`)
- LRU eviction (500 MB max cache)
- TTL: 7 days
- Verify integrity (checksums)

### 4. Local RAG (user_playbooks Collection)
- Store full playbook metadata + scripts in local Qdrant
- Vector search over user's playbooks
- Coordinate with server agent (provide search results)
- Support offline operation

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ PLAYBOOK SERVICE (Rust)                                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Playbook Manager                                       │ │
│  │  • Create/Read/Update/Delete playbooks                 │ │
│  │  • Validate playbook metadata                          │ │
│  │  • Version management                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Metadata Publisher                                     │ │
│  │  • Extract metadata from playbooks                     │ │
│  │  • Sync to server tenant space                         │ │
│  │  • Handle conflicts (server vs local)                  │ │
│  │  • Background sync scheduler                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Script Manager                                         │ │
│  │  • Download scripts from server                        │ │
│  │  • Cache management (LRU, TTL)                         │ │
│  │  • Resolve script paths (local vs cached vs download) │ │
│  │  • Verify checksums                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Local RAG Manager                                      │ │
│  │  • Manage user_playbooks collection in Qdrant         │ │
│  │  • Generate embeddings (local model)                   │ │
│  │  • Search user's playbooks                             │ │
│  │  • Sync with filesystem                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         ↕                    ↕                    ↕
   Filesystem          Storage Service       Server API
  ~/.escher/           (Qdrant)              (WebSocket)
```

---

## File Structure

```
~/.escher/
├── playbooks/                          # Local playbook storage
│   ├── local/                          # Never uploaded (local_only)
│   │   ├── user-backup-prod/
│   │   │   ├── v1.0.0/
│   │   │   │   ├── metadata.json
│   │   │   │   ├── shell/backup.sh
│   │   │   │   ├── terraform/main.tf
│   │   │   │   └── python/backup.py
│   │   │   └── v1.1.0/
│   │   └── user-migration-tool/
│   │
│   ├── synced/                         # Uploaded to server
│   │   ├── user-weekend-shutdown/
│   │   │   ├── v1.0.0/
│   │   │   │   ├── metadata.json
│   │   │   │   ├── shell/stop.sh
│   │   │   │   └── python/stop.py
│   │   │   └── v1.2.0/
│   │   └── ai-generated-cleanup/
│   │
│   └── ai-generated/                   # Pending user decision
│       └── temp-cleanup-logs-v1.0.0/
│
├── cache/                              # Downloaded from server
│   ├── scripts/
│   │   ├── escher-aws-rds-stop-v1.0.0/
│   │   │   ├── shell/stop.sh
│   │   │   └── python/stop_rds.py
│   │   └── escher-aws-ec2-start-v1.0.0/
│   └── metadata/
│       └── last_sync.json
│
└── qdrant/                             # Local vector database
    └── collections/
        └── user_playbooks/             # 6th collection
```

---

## Data Flow

### Flow 1: User Creates Playbook

```
User creates playbook via UI
    ↓
Frontend calls Tauri command: create_playbook()
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Rust)                            │
├────────────────────────────────────────────────────┤
│  1. Validate metadata                              │
│  2. Generate playbook ID                           │
│  3. Save to filesystem:                            │
│     • ~/.escher/playbooks/local/my-playbook/v1.0.0/│
│     • Write metadata.json                          │
│     • Write scripts (shell, TF, etc.)              │
│                                                     │
│  4. Update local RAG:                              │
│     • Generate embedding                           │
│     • Insert into user_playbooks collection        │
│                                                     │
│  5. Ask user: Upload to server?                    │
│     ○ Keep Private → Done                          │
│     ○ Upload for Review → Continue to Flow 2       │
│     ○ Upload Trusted → Continue to Flow 2          │
└────────────────────────────────────────────────────┘
```

### Flow 2: Publish Metadata to Server

```
User chooses: "Upload for Review" or "Upload Trusted"
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service - Metadata Publisher              │
├────────────────────────────────────────────────────┤
│  1. Extract metadata (no scripts)                  │
│     {                                               │
│       playbook_id, version, name, description,     │
│       keywords, cloud_provider, resource_types,    │
│       storage_strategy: "uploaded_for_review",     │
│       vector: [0.123, -0.456, ...]                 │
│     }                                               │
│                                                     │
│  2. Upload scripts to tenant space                 │
│     POST /api/playbook/upload                      │
│     → S3: s3://escher-tenant-data/tenants/{id}/... │
│                                                     │
│  3. Publish metadata to server                     │
│     POST /api/playbook/metadata/publish            │
│     → Server updates tenant RAG collection         │
│                                                     │
│  4. Move to synced directory                       │
│     mv ~/.escher/playbooks/local/my-playbook       │
│        ~/.escher/playbooks/synced/my-playbook      │
│                                                     │
│  5. Update local RAG (mark as synced)              │
└────────────────────────────────────────────────────┘
```

### Flow 3: Download & Cache Script

```
Server returns playbook reference
    ↓
Frontend needs to execute playbook
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service - Script Manager                  │
├────────────────────────────────────────────────────┤
│  1. Check if script exists locally                 │
│     • Check: ~/.escher/playbooks/local/            │
│     • Check: ~/.escher/playbooks/synced/           │
│     • If found → Return path                       │
│                                                     │
│  2. Check cache                                    │
│     • Check: ~/.escher/cache/scripts/{id}/         │
│     • If found & not expired → Return path         │
│                                                     │
│  3. Download from server                           │
│     GET /api/script/download?s3_path=...           │
│     → Receive pre-signed URL                       │
│     → Download script                              │
│                                                     │
│  4. Verify integrity                               │
│     • Check SHA-256 checksum                       │
│                                                     │
│  5. Cache for future use                           │
│     • Write to ~/.escher/cache/scripts/{id}/       │
│     • Set expiry: 7 days                           │
│                                                     │
│  6. Return path to execution engine                │
└────────────────────────────────────────────────────┘
```

---

## Documentation

- **[metadata-publisher.md](./metadata-publisher.md)** - Metadata sync logic, conflict resolution
- **[script-manager.md](./script-manager.md)** - Download, cache, eviction strategies
- **[local-rag.md](./local-rag.md)** - user_playbooks collection management
- **[file-structure.md](./file-structure.md)** - Directory organization, naming conventions
- **[api.md](./api.md)** - Rust API reference
- **[storage-strategies.md](./storage-strategies.md)** - local_only, uploaded_for_review, uploaded_trusted

---

## API Overview

```rust
pub struct PlaybookService {
    config: PlaybookConfig,
    storage: Arc<StorageService>,  // Access to Qdrant
    http_client: reqwest::Client,
}

impl PlaybookService {
    // Playbook Management
    pub async fn create_playbook(&self, playbook: Playbook) -> Result<String>;
    pub async fn get_playbook(&self, id: &str, version: &str) -> Result<Playbook>;
    pub async fn update_playbook(&self, id: &str, playbook: Playbook) -> Result<()>;
    pub async fn delete_playbook(&self, id: &str, version: &str) -> Result<()>;
    pub async fn list_playbooks(&self) -> Result<Vec<PlaybookMetadata>>;

    // Metadata Publishing
    pub async fn publish_metadata(&self, playbook_id: &str) -> Result<()>;
    pub async fn sync_all_metadata(&self) -> Result<SyncReport>;
    pub async fn start_background_sync(&self) -> Result<()>;

    // Script Management
    pub async fn resolve_script(&self, playbook_id: &str, format: ScriptFormat) -> Result<PathBuf>;
    pub async fn download_script(&self, s3_path: &str) -> Result<PathBuf>;
    pub async fn cache_script(&self, playbook_id: &str, script: &[u8]) -> Result<()>;
    pub async fn evict_old_cache(&self) -> Result<u64>;  // Returns bytes freed

    // Local RAG
    pub async fn search_local_playbooks(&self, query: &str) -> Result<Vec<PlaybookSearchResult>>;
    pub async fn add_to_rag(&self, playbook: &Playbook) -> Result<()>;
    pub async fn remove_from_rag(&self, playbook_id: &str) -> Result<()>;
}
```

---

## Integration with Other Services

### Storage Service
- Playbook Service uses Storage Service for Qdrant access
- user_playbooks collection is managed by Playbook Service
- Storage Service provides embedding generation

### Execution Engine
- Playbook Service resolves script paths
- Execution Engine runs the scripts
- Results tracked in executed_operations collection

### Server (Playbook Agent)
- Server searches both Escher + Tenant RAG
- Returns playbook references
- Client resolves full playbooks locally

---

## Configuration

```rust
pub struct PlaybookConfig {
    /// Base directory for playbooks
    pub playbooks_dir: PathBuf,  // ~/.escher/playbooks/

    /// Cache directory
    pub cache_dir: PathBuf,  // ~/.escher/cache/

    /// Cache settings
    pub cache_max_size_mb: u64,  // 500 MB
    pub cache_ttl_days: u64,     // 7 days

    /// Sync settings
    pub sync_enabled: bool,
    pub sync_interval_minutes: u64,  // 5 minutes

    /// Server connection
    pub server_url: String,
    pub tenant_id: String,
}
```

---

## Next Steps

1. Implement core PlaybookService struct
2. Build Metadata Publisher with background sync
3. Implement Script Manager with caching
4. Integrate with Storage Service (user_playbooks collection)
5. Add Tauri commands for frontend integration
6. Write comprehensive tests

---

See individual documentation files for detailed specifications.