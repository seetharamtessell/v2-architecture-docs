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

The **Storage Service** is a **reusable npm package** that abstracts all local storage operations.

### Purpose
- Centralized storage layer used by Tauri Rust backend
- Handles both chat history and AWS estate data
- Uses Qdrant as the underlying storage engine
- Provides clean API for storage operations

### Two Main Responsibilities

#### 1. Context & Chat History Management
```javascript
// Exposed Methods
storageService.chat.append(contextId, message)
storageService.chat.getHistory(contextId, limit)
storageService.chat.delete(contextId)
storageService.chat.updateContext(contextId, metadata)
storageService.chat.searchConversations(query)
```

#### 2. RAG with AWS Estate
```javascript
// Exposed Methods
storageService.estate.upsertResource(resource)
storageService.estate.search(query, filters)
storageService.estate.getResource(resourceId)
storageService.estate.deleteResource(resourceId)
storageService.estate.bulkUpsert(resources)
storageService.estate.getByAccount(accountId)
storageService.estate.getByRegion(region)
```

### Storage Strategy

```
┌─────────────────────────────────────────────┐
│         Storage Service (npm package)        │
├─────────────────────────────────────────────┤
│                                              │
│  Qdrant Client                               │
│    ↓                                         │
│  ┌──────────────┐  ┌──────────────┐        │
│  │ Collection:  │  │ Collection:  │        │
│  │ chat_history │  │ aws_estate   │        │
│  └──────────────┘  └──────────────┘        │
│         ↓                  ↓                 │
└─────────────────────────────────────────────┘
         ↓                  ↓
┌─────────────────────────────────────────────┐
│         PRIMARY: Local Filesystem            │
│         (Encrypted at rest)                  │
│                                              │
│  ~/.cloudops/data/                           │
│    ├── qdrant/                               │
│    │   ├── chat_history.db                  │
│    │   └── aws_estate.db                    │
│    └── metadata/                             │
│        └── sync_status.json                  │
└─────────────────────────────────────────────┘
         ↓ (backup)
┌─────────────────────────────────────────────┐
│         BACKUP: S3                           │
│         (Optional, configurable)             │
│                                              │
│  s3://user-cloudops-backup/                  │
│    ├── {user_id}/                            │
│    │   ├── chat_history/                     │
│    │   │   └── {date}.encrypted.tar.gz      │
│    │   └── aws_estate/                       │
│    │       └── {date}.encrypted.tar.gz      │
└─────────────────────────────────────────────┘
```

### Storage Service Architecture

```typescript
// Package structure: @cloudops/storage-service

interface StorageServiceConfig {
  localPath: string;           // ~/.cloudops/data/
  encryptionKey: string;        // For at-rest encryption
  qdrantUrl: string;            // Local Qdrant endpoint
  s3Backup?: {
    enabled: boolean;
    bucket: string;
    region: string;
    syncInterval: number;       // Hours
  };
}

class StorageService {
  // Chat operations
  chat: ChatStorage;

  // Estate operations
  estate: EstateStorage;

  // Backup operations
  backup: BackupManager;

  // Initialize
  async init(config: StorageServiceConfig): Promise<void>;

  // Health check
  async health(): Promise<StorageHealth>;
}

interface ChatStorage {
  append(contextId: string, message: Message): Promise<void>;
  getHistory(contextId: string, limit?: number): Promise<Message[]>;
  delete(contextId: string): Promise<void>;
  updateContext(contextId: string, metadata: any): Promise<void>;
  searchConversations(query: string): Promise<Conversation[]>;
  getAllContexts(): Promise<string[]>;
}

interface EstateStorage {
  upsertResource(resource: AWSResource): Promise<void>;
  bulkUpsert(resources: AWSResource[]): Promise<void>;
  search(query: string, filters?: ResourceFilter): Promise<AWSResource[]>;
  getResource(resourceId: string): Promise<AWSResource | null>;
  deleteResource(resourceId: string): Promise<void>;
  getByAccount(accountId: string): Promise<AWSResource[]>;
  getByRegion(region: string): Promise<AWSResource[]>;
  getByType(resourceType: string): Promise<AWSResource[]>;
  count(): Promise<number>;
}

interface BackupManager {
  backupToS3(): Promise<void>;
  restoreFromS3(date: string): Promise<void>;
  listBackups(): Promise<Backup[]>;
  autoBackup(enabled: boolean): Promise<void>;
}
```

### Encryption Strategy

**At Rest:**
- Qdrant data files encrypted using AES-256
- Encryption key stored in OS Keychain
- Key derivation from user password (optional)

**In Transit (to S3):**
- Files tar.gz + encrypted before upload
- S3 bucket has encryption enabled
- IAM policies restrict access

### Backup Strategy

**Automatic Backup to S3:**
- Scheduled: Every 24 hours (configurable)
- Incremental: Only changes since last backup
- Retention: Last 30 days (configurable)

**Manual Backup:**
- User can trigger on-demand
- Export to local file (encrypted zip)

**Restore:**
- From S3 backup (select date)
- From local export file
- Merge with existing data or replace

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
│  qdrant_manager                                     │
│    → Initialize local Qdrant instance               │
│    → Create collections                             │
│    → Insert/update vectors                          │
│    → Semantic search                                │
│    → Exact match fallback                           │
│                                                      │
│  chat_store                                         │
│    → SQLite database for chat history               │
│    → CRUD operations on conversations               │
│    → Context retrieval (last N messages)            │
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
├─────────────────────────────────────────────────────┤
│  • Qdrant                                           │
│      - Vector database for AWS resources            │
│      - Collection: "aws_resources"                  │
│      - Embeddings for semantic search               │
│                                                      │
│  • SQLite                                           │
│      - Chat history                                 │
│      - Execution logs                               │
│      - Sync metadata                                │
│                                                      │
│  • File Cache (optional)                            │
│      - Estate snapshots                             │
│      - Embeddings cache                             │
│                                                      │
│  • OS Keychain (optional)                           │
│      - Encrypted AWS credentials                    │
│      - Alternative to ~/.aws/credentials            │
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

### Chat Message (SQLite)

```sql
CREATE TABLE conversations (
    id TEXT PRIMARY KEY,
    title TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    conversation_id TEXT REFERENCES conversations(id),
    role TEXT, -- 'user' | 'assistant' | 'system'
    content TEXT,
    timestamp TIMESTAMP,
    metadata JSON -- { resources_mentioned: [...], commands_executed: [...] }
);
```

### Execution Log (SQLite)

```sql
CREATE TABLE executions (
    id TEXT PRIMARY KEY,
    conversation_id TEXT REFERENCES conversations(id),
    message_id TEXT REFERENCES messages(id),
    command TEXT,
    status TEXT, -- 'pending' | 'running' | 'success' | 'failed'
    output TEXT,
    error TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    metadata JSON -- { resource_ids: [...], approval_required: bool, approved_by: '...' }
);
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

// Sync
#[tauri::command]
async fn trigger_sync(sync_type: String) -> Result<(), String> // 'full' | 'incremental'

#[tauri::command]
async fn get_sync_status() -> Result<SyncStatus, String>
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
  - Local model (e.g., sentence-transformers)?
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
- **Vector DB**: qdrant-client (Rust client for Qdrant)
- **Database**: rusqlite (SQLite for Rust)
- **JSON**: serde_json
- **Async Runtime**: tokio

### Local Services
- **Qdrant**: Run as local service or embedded?
- **SQLite**: File-based (app data directory)

---

## Open Questions & Notes

### Performance
- How to handle slow AWS API calls during initial scan?
  - Show progress bar?
  - Background scan with notification?
- Search performance in Qdrant with 10k+ resources?
- React performance with long chat history?

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

### Testing Strategy
- Unit tests for Rust modules
- Integration tests for Tauri commands
- E2E tests for React UI
- Mock AWS APIs for testing?

---

## Next Steps

1. **Nail down core decisions** (execution model, state management, etc.)
2. **Create detailed Rust module designs**
3. **Design React component hierarchy**
4. **Define data schemas (Qdrant collections, SQLite tables)**
5. **Create sequence diagrams for key flows**
6. **Prototype critical paths** (scan → search → execute)

---

## Notes from Discussion

- Connection unstable (working from airplane)
- Need offline-capable design doc
- Focus on getting architecture right before implementation
- Server side is separate - has its own complex world with agents/microservices
- Client must be self-contained and work independently

---

## References

- Main architecture doc: [architecture.md](architecture.md)
- Client overview: [docs/02-client/overview.md](docs/02-client/overview.md)
- Server overview: [docs/03-server/overview.md](docs/03-server/overview.md)