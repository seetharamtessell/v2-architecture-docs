# Client Module Architecture

**Purpose**: Modular design with separated concerns for maintainability and reusability

---

## Module Overview

```
Tauri Client Application
├── Module 1: Storage Management (Rust crate)
│   ├── Estate Storage + RAG
│   └── Context & Chat Storage
│
├── Module 2: Execution Engine (Rust crate)
│   ├── AWS Command Execution
│   └── Execution History & Logs
│
├── Module 3: Estate Scanner (Rust crate)
│   ├── AWS API Integration
│   └── Resource Discovery
│
├── Module 4: Request Builder (Rust crate)
│   ├── Context Enrichment
│   └── Server Communication
│
└── Module 5: Frontend (React)
    └── UI Components
```

---

## Module 1: Storage Management ✅ (DESIGNED)

**Crate Name**: `cloudops-storage-service`

### Responsibilities
1. **Estate Storage with RAG**
   - Store AWS resources in Qdrant
   - Semantic search over resources
   - Vector embeddings for fuzzy matching
   - Filter by account, region, type

2. **Context & Chat Storage**
   - Store conversation history
   - Maintain context per conversation
   - Retrieve relevant chat history
   - Semantic search over conversations

### Public API
```rust
// Crate: cloudops-storage-service

pub struct StorageService {
    pub estate: EstateStorage,
    pub chat: ChatStorage,
    pub backup: BackupManager,
}

// Estate Storage
impl EstateStorage {
    pub async fn upsert_resource(&self, resource: AWSResource) -> Result<()>;
    pub async fn bulk_upsert(&self, resources: Vec<AWSResource>) -> Result<()>;
    pub async fn search(&self, query: &str, filters: Option<ResourceFilter>) -> Result<Vec<AWSResource>>;
    pub async fn get_resource(&self, resource_id: &str) -> Result<Option<AWSResource>>;
    pub async fn get_by_account(&self, account_id: &str) -> Result<Vec<AWSResource>>;
    pub async fn get_by_region(&self, region: &str) -> Result<Vec<AWSResource>>;
    pub async fn count(&self) -> Result<usize>;
}

// Chat Storage
impl ChatStorage {
    pub async fn append(&self, context_id: &str, message: Message) -> Result<()>;
    pub async fn get_history(&self, context_id: &str, limit: Option<usize>) -> Result<Vec<Message>>;
    pub async fn search_conversations(&self, query: &str) -> Result<Vec<Conversation>>;
    pub async fn delete(&self, context_id: &str) -> Result<()>;
}

// Backup Manager
impl BackupManager {
    pub async fn create_snapshot(&self, collection: &str) -> Result<SnapshotInfo>;
    pub async fn upload_to_s3(&self, snapshot: &SnapshotInfo) -> Result<()>;
    pub async fn restore_from_s3(&self, collection: &str, date: &str) -> Result<()>;
    pub async fn start_auto_backup(&self) -> Result<()>;
}
```

### Internal Structure
```
cloudops-storage-service/
├── src/
│   ├── lib.rs              # Public API
│   ├── config.rs           # StorageConfig
│   ├── estate_storage.rs   # AWS resource storage + RAG
│   ├── chat_storage.rs     # Chat history storage
│   ├── backup_manager.rs   # S3 backup orchestration
│   ├── encryption.rs       # AES-256-GCM encryption
│   ├── qdrant_client.rs    # Qdrant wrapper
│   └── types.rs            # Shared types (AWSResource, Message, etc.)
├── Cargo.toml
└── tests/
    └── integration_tests.rs
```

### Dependencies
```toml
[dependencies]
qdrant-client = "1.7"
aws-sdk-s3 = "1.0"
aes-gcm = "0.10"
keyring = "2.0"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1.0"
chrono = "0.4"
uuid = { version = "1", features = ["v4"] }
```

---

## Module 2: Execution Engine (NEW)

**Crate Name**: `cloudops-execution-engine`

### Responsibilities
1. **AWS Command Execution**
   - Execute AWS CLI commands
   - Execute via AWS SDK for Rust
   - Stream output in real-time
   - Handle errors and retries

2. **Execution History & Logs**
   - Store execution logs
   - Track execution status
   - Provide execution history
   - Audit trail

### Public API
```rust
// Crate: cloudops-execution-engine

pub struct ExecutionEngine {
    executor: CommandExecutor,
    history: ExecutionHistory,
}

impl ExecutionEngine {
    pub async fn new(config: ExecutionConfig) -> Result<Self>;

    // Execute command
    pub async fn execute(&self, command: ExecutionRequest) -> Result<ExecutionResult>;

    // Execute with approval callback
    pub async fn execute_with_approval<F>(
        &self,
        command: ExecutionRequest,
        approval_fn: F,
    ) -> Result<ExecutionResult>
    where
        F: Fn(&ExecutionPlan) -> bool;

    // Get execution status
    pub async fn get_status(&self, execution_id: &str) -> Result<ExecutionStatus>;

    // Stream output
    pub async fn stream_output(&self, execution_id: &str) -> Result<impl Stream<Item = String>>;

    // Execution history
    pub async fn get_history(&self, filters: HistoryFilter) -> Result<Vec<ExecutionRecord>>;

    // Cancel execution
    pub async fn cancel(&self, execution_id: &str) -> Result<()>;
}

// Execution Request
#[derive(Debug, Clone)]
pub struct ExecutionRequest {
    pub context_id: String,
    pub command_type: CommandType,
    pub resource_id: String,
    pub parameters: HashMap<String, String>,
    pub approval_required: bool,
    pub risk_level: RiskLevel,
}

#[derive(Debug, Clone)]
pub enum CommandType {
    AwsCli { command: String, args: Vec<String> },
    AwsSdk { service: String, operation: String, params: Value },
    Script { script: String },
}

#[derive(Debug, Clone)]
pub enum RiskLevel {
    Low,    // Read-only operations
    Medium, // Modifications to non-critical resources
    High,   // Modifications to production resources
}

// Execution Result
#[derive(Debug, Clone)]
pub struct ExecutionResult {
    pub id: String,
    pub status: ExecutionStatus,
    pub output: String,
    pub error: Option<String>,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub duration_ms: Option<u64>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum ExecutionStatus {
    Pending,
    AwaitingApproval,
    Running,
    Success,
    Failed,
    Cancelled,
}
```

### Internal Structure
```
cloudops-execution-engine/
├── src/
│   ├── lib.rs                # Public API
│   ├── config.rs             # ExecutionConfig
│   ├── executor/
│   │   ├── mod.rs            # CommandExecutor trait
│   │   ├── aws_cli.rs        # AWS CLI execution
│   │   ├── aws_sdk.rs        # AWS SDK execution
│   │   └── script.rs         # Script execution
│   ├── history.rs            # ExecutionHistory (uses storage module)
│   ├── approval.rs           # Approval workflow
│   ├── streaming.rs          # Output streaming
│   └── types.rs              # Shared types
├── Cargo.toml
└── tests/
    └── integration_tests.rs
```

### Dependencies
```toml
[dependencies]
aws-sdk-ec2 = "1.0"
aws-sdk-rds = "1.0"
aws-sdk-s3 = "1.0"
# ... other AWS SDKs as needed

tokio = { version = "1", features = ["full", "process"] }
tokio-stream = "0.1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1.0"
chrono = "0.4"
uuid = { version = "1", features = ["v4"] }

# Use storage module for history
cloudops-storage-service = { path = "../cloudops-storage-service" }
```

---

## Module 3: Estate Scanner (NEW)

**Crate Name**: `cloudops-estate-scanner`

### Responsibilities
1. **AWS API Integration**
   - Connect to AWS APIs
   - Handle credentials from ~/.aws/credentials
   - Support multiple accounts/profiles
   - Handle rate limiting, pagination, retries

2. **Resource Discovery**
   - Scan EC2, RDS, S3, Lambda, etc.
   - Transform to standard format
   - Extract metadata (tags, state, permissions)
   - Incremental vs full sync

### Public API
```rust
// Crate: cloudops-estate-scanner

pub struct EstateScanner {
    aws_clients: AwsClientPool,
    config: ScannerConfig,
}

impl EstateScanner {
    pub async fn new(config: ScannerConfig) -> Result<Self>;

    // Full scan
    pub async fn scan_all(&self, profile: &str) -> Result<ScanResult>;

    // Incremental scan
    pub async fn scan_incremental(&self, profile: &str, since: DateTime<Utc>) -> Result<ScanResult>;

    // Scan specific service
    pub async fn scan_service(&self, profile: &str, service: AwsService) -> Result<Vec<AWSResource>>;

    // Scan with progress callback
    pub async fn scan_with_progress<F>(
        &self,
        profile: &str,
        progress_fn: F,
    ) -> Result<ScanResult>
    where
        F: Fn(ScanProgress);

    // List available profiles
    pub fn list_profiles(&self) -> Result<Vec<String>>;

    // Validate credentials
    pub async fn validate_credentials(&self, profile: &str) -> Result<bool>;
}

#[derive(Debug, Clone)]
pub enum AwsService {
    EC2,
    RDS,
    S3,
    Lambda,
    VPC,
    ELB,
    IAM,
    // ... more services
}

#[derive(Debug)]
pub struct ScanResult {
    pub resources: Vec<AWSResource>,
    pub total_count: usize,
    pub by_service: HashMap<AwsService, usize>,
    pub duration_ms: u64,
    pub errors: Vec<ScanError>,
}

#[derive(Debug)]
pub struct ScanProgress {
    pub service: AwsService,
    pub current: usize,
    pub total: usize,
    pub percentage: f32,
}
```

### Internal Structure
```
cloudops-estate-scanner/
├── src/
│   ├── lib.rs              # Public API
│   ├── config.rs           # ScannerConfig
│   ├── credentials.rs      # Read ~/.aws/credentials
│   ├── client_pool.rs      # AWS client management
│   ├── scanners/
│   │   ├── mod.rs          # Scanner trait
│   │   ├── ec2.rs          # EC2 scanner
│   │   ├── rds.rs          # RDS scanner
│   │   ├── s3.rs           # S3 scanner
│   │   ├── lambda.rs       # Lambda scanner
│   │   └── ... (more services)
│   ├── transform.rs        # Transform to standard AWSResource
│   ├── sync_strategy.rs    # Full vs incremental logic
│   └── types.rs            # Shared types
├── Cargo.toml
└── tests/
    └── integration_tests.rs
```

### Dependencies
```toml
[dependencies]
aws-config = "1.0"
aws-sdk-ec2 = "1.0"
aws-sdk-rds = "1.0"
aws-sdk-s3 = "1.0"
aws-sdk-lambda = "1.0"
# ... other AWS SDKs

tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
anyhow = "1.0"
chrono = "0.4"
```

---

## Module 4: Request Builder (NEW)

**Crate Name**: `cloudops-request-builder`

### Responsibilities
1. **Context Enrichment**
   - Take user prompt
   - Search estate for mentioned resources
   - Add full resource metadata
   - Add relevant chat history

2. **Server Communication**
   - Build request payload
   - Send to server
   - Parse server response
   - Handle retries and errors

### Public API
```rust
// Crate: cloudops-request-builder

pub struct RequestBuilder {
    storage: StorageService,
    server_client: ServerClient,
    config: RequestConfig,
}

impl RequestBuilder {
    pub async fn new(config: RequestConfig, storage: StorageService) -> Result<Self>;

    // Build enriched request
    pub async fn build_request(
        &self,
        context_id: &str,
        prompt: &str,
    ) -> Result<EnrichedRequest>;

    // Send to server and get response
    pub async fn send_to_server(
        &self,
        request: EnrichedRequest,
    ) -> Result<ServerResponse>;

    // Complete flow: build + send
    pub async fn process_prompt(
        &self,
        context_id: &str,
        prompt: &str,
    ) -> Result<ServerResponse>;
}

#[derive(Debug, Clone, Serialize)]
pub struct EnrichedRequest {
    pub prompt: String,
    pub identified_resources: Vec<AWSResource>,
    pub chat_history: Vec<Message>,
    pub user_context: UserContext,
}

#[derive(Debug, Clone, Deserialize)]
pub struct ServerResponse {
    pub response_type: ResponseType,
    pub content: String,
    pub execution_plan: Option<ExecutionPlan>,
    pub requires_approval: bool,
    pub risk_assessment: Option<RiskAssessment>,
}

#[derive(Debug, Clone, Deserialize)]
pub enum ResponseType {
    Command,        // Commands to execute
    Information,    // Just information
    Clarification,  // Need more info from user
}

#[derive(Debug, Clone, Deserialize)]
pub struct ExecutionPlan {
    pub steps: Vec<ExecutionStep>,
    pub estimated_duration: Option<u64>,
}

#[derive(Debug, Clone, Deserialize)]
pub struct ExecutionStep {
    pub step_id: String,
    pub command: String,
    pub args: Vec<String>,
    pub description: String,
}
```

### Internal Structure
```
cloudops-request-builder/
├── src/
│   ├── lib.rs              # Public API
│   ├── config.rs           # RequestConfig
│   ├── enrichment.rs       # Context enrichment logic
│   ├── resource_matcher.rs # Match resources from prompt
│   ├── server_client.rs    # HTTP client to server
│   └── types.rs            # Shared types
├── Cargo.toml
└── tests/
    └── integration_tests.rs
```

### Dependencies
```toml
[dependencies]
reqwest = { version = "0.11", features = ["json"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1.0"

# Use storage module
cloudops-storage-service = { path = "../cloudops-storage-service" }
```

---

## Module 5: Frontend (React)

**Not a Rust crate - React application**

### Structure
```
frontend/
├── src/
│   ├── components/
│   │   ├── ChatInterface/
│   │   ├── ResourceExplorer/
│   │   ├── ExecutionMonitor/
│   │   ├── ConversationHistory/
│   │   └── AccountManager/
│   ├── hooks/
│   │   ├── useTauriCommand.ts
│   │   ├── useChat.ts
│   │   ├── useResources.ts
│   │   └── useExecution.ts
│   ├── services/
│   │   └── tauri.ts           # Tauri command wrappers
│   ├── types/
│   │   └── index.ts           # TypeScript types
│   └── App.tsx
├── package.json
└── tsconfig.json
```

---

## Module Integration in Tauri

**Main Tauri Backend**: Integrates all modules

```rust
// src-tauri/src/main.rs

use cloudops_storage_service::StorageService;
use cloudops_execution_engine::ExecutionEngine;
use cloudops_estate_scanner::EstateScanner;
use cloudops_request_builder::RequestBuilder;

struct AppState {
    storage: StorageService,
    execution: ExecutionEngine,
    scanner: EstateScanner,
    request_builder: RequestBuilder,
}

#[tauri::command]
async fn send_message(
    state: tauri::State<'_, AppState>,
    context_id: String,
    prompt: String,
) -> Result<ServerResponse, String> {
    // Use request builder
    let response = state.request_builder
        .process_prompt(&context_id, &prompt)
        .await
        .map_err(|e| e.to_string())?;

    // Store in chat history
    state.storage.chat
        .append(&context_id, Message { content: prompt, role: "user", ... })
        .await
        .map_err(|e| e.to_string())?;

    Ok(response)
}

#[tauri::command]
async fn execute_command(
    state: tauri::State<'_, AppState>,
    execution_id: String,
    approved: bool,
) -> Result<ExecutionResult, String> {
    if !approved {
        return Err("Execution not approved".to_string());
    }

    // Use execution engine
    let result = state.execution
        .execute(/* ... */)
        .await
        .map_err(|e| e.to_string())?;

    Ok(result)
}

#[tauri::command]
async fn scan_aws_estate(
    state: tauri::State<'_, AppState>,
    profile: String,
) -> Result<ScanResult, String> {
    // Use estate scanner
    let scan_result = state.scanner
        .scan_all(&profile)
        .await
        .map_err(|e| e.to_string())?;

    // Store in storage
    state.storage.estate
        .bulk_upsert(scan_result.resources)
        .await
        .map_err(|e| e.to_string())?;

    Ok(scan_result)
}
```

---

## Module Dependencies

```
┌─────────────────────────────────────┐
│         Tauri Backend               │
│         (Integrates all modules)    │
└─────────────────────────────────────┘
         ↓       ↓       ↓       ↓
    ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
    │Storage │ │Execute │ │Scanner │ │Request │
    │Module  │ │Module  │ │Module  │ │Builder │
    └────────┘ └────────┘ └────────┘ └────────┘
         ↑                              ↑
         └──────────────────────────────┘
              (Request Builder uses Storage)
              (Execution uses Storage for logs)
```

---

## Summary

### ✅ **Module 1: Storage Management**
- Estate storage with RAG
- Chat context storage
- S3 backup management
- **Status**: Designed ✅

### ✅ **Module 2: Execution Engine** (NEW)
- AWS command execution
- Execution history & logs
- Approval workflow
- **Status**: To be designed 🔄

### ✅ **Module 3: Estate Scanner** (NEW)
- AWS API integration
- Resource discovery
- Multi-account support
- **Status**: To be designed 🔄

### ✅ **Module 4: Request Builder** (NEW)
- Context enrichment
- Server communication
- **Status**: To be designed 🔄

### ✅ **Module 5: Frontend** (React)
- UI components
- Tauri integration
- **Status**: To be designed 🔄

---

## Next Steps

1. ✅ Storage Module (Done)
2. 🔄 Design Execution Engine module in detail
3. 🔄 Design Estate Scanner module in detail
4. 🔄 Design Request Builder module in detail
5. 🔄 Design Frontend architecture