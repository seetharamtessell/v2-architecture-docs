# Escher Client - Complete Summary

**Status**: 90% Complete | 42 Documentation Files | ~25,000 Lines
**Last Updated**: October 2025

---

## Executive Summary

The **Escher Client** is a **Tauri-based desktop application** that provides a chat-driven interface for AWS cloud operations. It combines a React frontend with Rust backend modules to enable:

- 🏗️ **Local AWS Estate Management** - Sync and store AWS resources locally
- 🔍 **Semantic Search** - Fast resource lookup using Qdrant vector DB
- ⚡ **Secure Execution** - Run AWS CLI commands with user approval
- 🎨 **Server-Driven UI** - Dynamic rendering with UI Agent components
- 🔐 **Credential Security** - AWS credentials never leave the device

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (React + TypeScript)            │
│  • MVC Architecture (Models, Views, Controllers, Services)  │
│  • 8 Complete User Flows (Login → Chat → Execution)        │
│  • 30+ UI Agent Components (Dynamic Rendering)              │
│  • AWS Cognito Authentication                               │
└─────────────────────────────────────────────────────────────┘
                              ↕
                    Tauri IPC Bridge
                    (70+ Commands, 15+ Events)
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                  RUST BACKEND (Tauri Core)                  │
│  • Storage Service - Estate + Chat + RAG                    │
│  • Execution Engine - AWS CLI + Playbooks                   │
│  • Estate Scanner - Resource Discovery                      │
│  • Request Builder - Context Enrichment (pending)           │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                      LOCAL STORAGE                          │
│  • Qdrant Vector DB (estate search + chat context)          │
│    - Chat: Dummy 1D vectors (filter-based access)           │
│    - Estate: Real 384D vectors (semantic search)            │
│  • OS Keychain (AWS credentials, encryption keys)           │
│  • S3 (backup/restore)                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Table of Contents

1. [Frontend Architecture](#frontend-architecture)
2. [Rust Backend Modules](#rust-backend-modules)
3. [Tauri Integration (IPC Bridge)](#tauri-integration-ipc-bridge)
4. [User Flows](#user-flows)
5. [Key Technologies](#key-technologies)
6. [Documentation Index](#documentation-index)
7. [What's Next](#whats-next)

---

## Frontend Architecture

### MVC Design Pattern

**Framework**: React 18+ with TypeScript

```
┌─────────────────────────────────────────────┐
│            VIEWS (Presentation)             │
│  Pure React Components                      │
│  • OpsChatView, EstateScanView, etc.       │
│  • UI Agent Components (Dynamic)            │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│         CONTROLLERS (Business Logic)        │
│  • ChatController, PlaybookController       │
│  • Data transformation & orchestration      │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│            MODELS (Data Layer)              │
│  • TypeScript interfaces                    │
│  • Zustand stores (UI state)                │
│  • Data transformers                        │
└─────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────┐
│         SERVICES (Infrastructure)           │
│  • Tauri IPC (Rust modules)                 │
│  • WebSocket (Server communication)         │
│  • Cognito (Authentication)                 │
│  • HTTP (REST API)                          │
└─────────────────────────────────────────────┘
```

### Project Structure

```
src/
├── models/              # Data layer
│   ├── types/          # TypeScript interfaces
│   ├── state/          # Zustand stores (UI state)
│   └── transformers/   # Data transformations
│
├── controllers/         # Business logic
│   ├── ChatController.ts
│   ├── PlaybookController.ts
│   ├── EstateScanController.ts
│   ├── RecommendationController.ts
│   ├── AlertController.ts
│   └── PermissionController.ts
│
├── views/               # Presentation layer
│   ├── auth/           # Login, signup
│   ├── chat/           # Ops chat interface
│   ├── estate/         # Learning center, scanner
│   ├── recommendations/
│   ├── alerts/
│   ├── permissions/
│   ├── reports/
│   └── shared/         # Shared components
│
├── services/            # Infrastructure
│   ├── tauri/          # Rust module wrappers
│   ├── websocket/      # Server communication
│   ├── cognito/        # Authentication
│   └── http/           # REST API
│
└── types/               # Shared TypeScript types
```

### Key Frontend Features

#### 1. **Server-Driven UI**
- Server sends UI specifications in enhancement messages
- Frontend renders dynamically using UI Agent components
- **Two-phase rendering**: Stream (text, immediate) → Enhancement (rich UI, 1-2s later)
- Users never wait for rich UI to start interacting

#### 2. **Intelligent Bucketing**
Critical for efficient conversation history:

```typescript
// When enhancement arrives from server
Enhancement {
  ui: {...}      → BUCKET 1: ChatUIState.uiCache (transient, 2-10KB/msg)
  history: {...} → BUCKET 2: Rust Storage Service (persistent, 200-500 bytes/msg)
}

// Next message to server includes history (not UI cache)
// Keeps context window manageable (10-100x smaller)
```

#### 3. **30+ UI Agent Components**
Packaged as `@escher/ui-agent-components`:

**Data Display**:
- Table, Card, List, KeyValuePairs, Badge, Tag

**Charts & Visualizations**:
- LineChart, BarChart, PieChart, AreaChart, RadarChart, ScatterChart

**Forms & Input**:
- TextInput, Select, Checkbox, Radio, DatePicker, Slider

**Interactive**:
- Button, Link, Tabs, Accordion, Modal, Drawer

**Operations**:
- PlaybookCard, ExecutionProgress, CommandOutput, StepIndicator

**Estate & Resources**:
- ResourceCard, ResourceTable, ResourceTree, ResourceGraph

**Feedback**:
- Alert, Toast, Notification, Progress, Spinner

#### 4. **Authentication & Security**
- **AWS Cognito** integration (Amplify)
- **JWT token** management (access + refresh)
- **OS Keychain** for secure token storage (macOS/Windows/Linux)
- **Session management** with auto-refresh
- Token expiry handling with graceful re-authentication

### Documentation
- [MVC Architecture](frontend/mvc-architecture.md)
- [User Flows](frontend/user-flows.md) - 8 complete flows
- [UI Agent Components](frontend/ui-agent-components.md)
- [Authentication & Security](frontend/authentication-security.md)

---

## Rust Backend Modules

The client's backend is organized into **5 focused Rust crates**:

### 1. Storage Service ✅

**Purpose**: Local data persistence with RAG capabilities

**Features**:
- **Single Qdrant Instance**: Embedded mode (~20-30 MB), dual collection strategy
- **Estate Storage**: Real vector embeddings (384D) for semantic search
- **Chat History**: Dummy vectors (1D) for efficient context storage
- **IAM Permissions**: Embedded with each AWS resource
- **S3 Backup/Restore**: Automatic cloud backup via snapshots
- **Encryption**: AES-256-GCM for sensitive data
- **Point Management**: Deterministic IDs prevent duplicates

**Key Operations**:
```rust
// Estate operations (Real vectors, semantic search)
upsert_resource(&resource) -> Result<()>
search(query, filters) -> Result<Vec<Resource>>  // Semantic search
get_by_account/region/type(filters) -> Result<Vec<Resource>>

// Chat operations (Dummy vectors, filter-based access)
append(context_id, message) -> Result<()>
get_history(context_id, limit) -> Result<Vec<Message>>
delete(context_id) -> Result<()>

// Backup/Restore (Qdrant snapshots)
create_snapshot(collection) -> Result<SnapshotInfo>
upload_to_s3(&snapshot) -> Result<()>
restore_from_s3(collection, date) -> Result<()>
```

**Collection Strategy**:
- **chat_history**: 1D dummy vectors, filter-based access, immutable messages
- **aws_estate**: 384D real vectors, semantic search + filters, deterministic point IDs

**Documentation** (9 files):
- [README](modules/storage-service/README.md) - Overview
- [Architecture](modules/storage-service/architecture.md)
- [Collections](modules/storage-service/collections.md) - Chat & Estate schemas
- [Operations](modules/storage-service/operations.md) - CRUD APIs
- [API](modules/storage-service/api.md) - Complete Rust API
- [Encryption](modules/storage-service/encryption.md) - AES-256-GCM
- [Point Management](modules/storage-service/point-management.md) - ID strategies
- [Backup/Restore](modules/storage-service/backup-restore.md) - S3 snapshots
- [Configuration](modules/storage-service/configuration.md)
- [Testing](modules/storage-service/testing.md)

---

### 2. Execution Engine ✅

**Purpose**: Secure command execution with real-time streaming

**Features**:
- **Command Types**: AWS CLI, shell scripts, custom executables
- **Streaming Output**: Real-time stdout/stderr via Tokio
- **Event System**: Progress updates and completion notifications
- **Playbook Support**: Multi-step execution with dependencies
- **Error Handling**: Comprehensive error recovery and reporting

**Key Operations**:
```rust
// Command execution
execute_command(command, context) -> Result<ExecutionId>
stream_output(execution_id) -> Stream<OutputChunk>
cancel_execution(execution_id) -> Result<()>

// Playbook execution
execute_playbook(playbook, context) -> Result<PlaybookExecutionId>
pause_playbook(execution_id) -> Result<()>
resume_playbook(execution_id) -> Result<()>

// Status tracking
get_execution_status(execution_id) -> Result<ExecutionStatus>
get_execution_history() -> Result<Vec<Execution>>
```

**Documentation** (9 files):
- [README](modules/execution-engine/README.md) - Overview
- [Architecture](modules/execution-engine/architecture.md)
- [API](modules/execution-engine/api.md) - Public interfaces
- [Types](modules/execution-engine/types.md) - Data structures
- [Usage](modules/execution-engine/usage.md) - Examples
- [Event Handlers](modules/execution-engine/event-handlers.md)
- [Configuration](modules/execution-engine/configuration.md)
- [Error Handling](modules/execution-engine/error-handling.md)
- [Cargo Integration](modules/execution-engine/cargo-integration.md)

---

### 3. Estate Scanner ✅

**Purpose**: AWS resource discovery and indexing

**Features**:
- **Multi-Account Scanning**: Parallel account processing
- **Multi-Region Support**: Scan across all AWS regions
- **Service-Specific Scanners**: EC2, RDS, S3, Lambda, IAM, VPC, etc.
- **IAM Permissions Discovery**: Analyze effective permissions
- **Semantic Embeddings**: Generate vectors for search
- **Incremental Updates**: Delta scanning for efficiency

**Key Operations**:
```rust
// Scanning
start_scan(config) -> Result<ScanId>
get_scan_progress(scan_id) -> Result<ScanProgress>
cancel_scan(scan_id) -> Result<()>

// Results
get_scan_results(scan_id) -> Result<ScanResults>
get_discovered_resources(filters) -> Result<Vec<Resource>>

// Configuration
configure_scanner(config) -> Result<()>
get_supported_services() -> Vec<ServiceInfo>
```

**Documentation** (4 files):
- [README](modules/estate-scanner/README.md) - Overview
- [Architecture](modules/estate-scanner/architecture.md)
- [Service Scanners](modules/estate-scanner/service-scanners.md) - 15+ services
- [Types](modules/estate-scanner/types.md)

---

### 4. Request Builder ⚠️

**Status**: Design needed (architecture decision pending)

**Purpose**: Enrich user requests with context before sending to server

**Planned Features**:
- Context enrichment from estate data
- Chat history integration
- Server communication
- Response parsing

**Design Options**:
1. **Lightweight Approach**: Minimal enrichment, server-heavy
2. **Rich Approach**: Deep context analysis, client-heavy

**Documentation** (1 file):
- [README](modules/request-builder/README.md) - Design considerations

---

### 5. Common Types ✅

**Purpose**: Shared data structures across all modules

**Features**:
- AWS resource type definitions
- Cross-module interfaces
- Serialization/deserialization helpers
- Type conversions

**Documentation** (1 file):
- [README](modules/common/README.md)

---

## Tauri Integration (IPC Bridge)

The **Tauri Integration Layer** connects the React frontend with Rust backend modules using Tauri's IPC (Inter-Process Communication) system.

### Overview

```
Frontend (TypeScript)          Rust Backend
       ↓                             ↑
TypeScript Service Wrappers
       ↓                             ↑
    invoke('command', params) → #[tauri::command]
       ↓                             ↑
    listen('event', handler) ← emit('event', payload)
```

### 70+ Commands Documented

Commands are organized by module:

#### Storage Commands (25+)
```typescript
// Estate operations
await invoke('get_estate_resources', { filters })
await invoke('semantic_search', { query, limit })

// Chat history
await invoke('save_message', { message })
await invoke('get_conversation_history', { sessionId })

// Policies
await invoke('get_policy', {})
await invoke('update_policy', { policy })

// Backup/Restore
await invoke('backup_to_s3', {})
await invoke('restore_from_s3', { backupId })
```

#### Execution Commands (15+)
```typescript
// Command execution
await invoke('execute_command', { command, context })
await invoke('cancel_execution', { executionId })

// Playbook execution
await invoke('execute_playbook', { playbook, context })
await invoke('pause_playbook', { executionId })
await invoke('resume_playbook', { executionId })

// Status
await invoke('get_execution_status', { executionId })
```

#### Estate Scanner Commands (15+)
```typescript
// Scanning
await invoke('start_scan', { config })
await invoke('cancel_scan', { scanId })

// Results
await invoke('get_scan_progress', { scanId })
await invoke('get_scan_results', { scanId })

// Configuration
await invoke('configure_scanner', { config })
await invoke('get_supported_services', {})
```

#### Authentication Commands (15+)
```typescript
// Token management
await invoke('store_tokens', { accessToken, refreshToken })
await invoke('get_access_token', {})
await invoke('refresh_token', {})
await invoke('clear_tokens', {})

// Session
await invoke('validate_session', {})
await invoke('get_session_info', {})
```

### 15+ Events Documented

Events enable real-time communication from Rust → Frontend:

#### Scan Events (6 events)
```typescript
// Listen to scan progress
await listen<ScanProgressPayload>('scan_progress', (event) => {
  const { scanId, progress, currentTask, elapsedSeconds } = event.payload
  updateProgressBar(progress.percentComplete)
  updateStatusText(currentTask)
})

// Other scan events:
// - scan_started: Scan initiated
// - scan_service_complete: Service scan finished
// - scan_complete: All services scanned
// - scan_cancelled: User cancelled
// - scan_error: Error occurred
```

#### Execution Events (6 events)
```typescript
// Listen to execution output
await listen<ExecutionOutputPayload>('execution_output', (event) => {
  const { executionId, output, stream } = event.payload
  appendOutput(executionId, output, stream) // stdout or stderr
})

// Other execution events:
// - execution_started: Command started
// - execution_complete: Command finished
// - execution_cancelled: User cancelled
// - playbook_step_start: Step started
// - playbook_step_complete: Step finished
```

#### System Events (8 events)
```typescript
// Session expired
await listen<SessionExpiredPayload>('session_expired', (event) => {
  clearUserState()
  showNotification('Session expired, please login')
  navigateToLogin()
})

// Other system events:
// - token_refresh_success: Token refreshed
// - token_refresh_failed: Refresh failed
// - network_status_changed: Online/offline
// - notification: General notification
// - background_task_complete: Task done
// - app_update_available: New version
// - system_error: Critical error
```

### TypeScript Service Wrappers

Services encapsulate Tauri commands for clean controller integration:

```typescript
// StorageService.ts
export class StorageService {
  async getEstateResources(filters: ResourceFilters): Promise<Resource[]> {
    return await invoke('get_estate_resources', { filters })
  }

  async semanticSearch(query: string, limit: number): Promise<Resource[]> {
    return await invoke('semantic_search', { query, limit })
  }
}

// ExecutionEngineService.ts
export class ExecutionEngineService {
  async executeCommand(command: Command, context: ExecutionContext): Promise<string> {
    return await invoke('execute_command', { command, context })
  }

  listenToOutput(executionId: string, callback: (output: string) => void) {
    return listen<ExecutionOutputPayload>('execution_output', (event) => {
      if (event.payload.executionId === executionId) {
        callback(event.payload.output)
      }
    })
  }
}

// EstateScannerService.ts
export class EstateScannerService {
  async startScan(config: ScanConfig): Promise<string> {
    return await invoke('start_scan', { config })
  }

  listenToProgress(scanId: string, callback: (progress: ScanProgress) => void) {
    return listen<ScanProgressPayload>('scan_progress', (event) => {
      if (event.payload.scanId === scanId) {
        callback(event.payload.progress)
      }
    })
  }
}
```

### Documentation
- [Tauri Integration README](tauri-integration/README.md)
- **Commands** (5 files):
  - [Storage Commands](tauri-integration/commands-storage.md)
  - [Execution Commands](tauri-integration/commands-execution.md)
  - [Estate Scanner Commands](tauri-integration/commands-estate-scanner.md)
  - [Auth Commands](tauri-integration/commands-auth.md)
  - [Request Builder Commands](tauri-integration/commands-request-builder.md) (preliminary)
- **Events** (3 files):
  - [Scan Events](tauri-integration/events-scan.md)
  - [Execution Events](tauri-integration/events-execution.md)
  - [System Events](tauri-integration/events-system.md)

---

## User Flows

The frontend supports **8 complete user journeys**:

### 1. Login & Authentication
- User enters credentials → Cognito authentication → Token stored in OS Keychain → Redirect to dashboard

### 2. Estate Scanning
- Select accounts/regions/services → Trigger scan → Real-time progress → Resources stored → Summary displayed

### 3. Learning Center (Estate Status)
- View estate overview → Total resources, costs, security score → Resource breakdown by service/region → Recent changes

### 4. Permissions Management
- View current policy → Edit allowed/blocked operations → Set resource constraints → Save policy

### 5. Recommendations
- View AI-generated suggestions → Apply (generates playbook) → Explain (detailed impact) → Postpone/Dismiss

### 6. Alerts & Notifications
- Real-time alerts via WebSocket → Banner for critical → Notification bell badge → Acknowledge/Resolve

### 7. Ops Chat with Playbook Execution
**Complete lifecycle**:
1. User types message (e.g., "Stop RDS db-prod-01")
2. **Phase 1**: Text streams immediately
3. **Phase 2**: Rich UI (playbook) arrives 1-2s later
4. User clicks **"Explain Plan"** → Modal shows detailed breakdown
5. User clicks **"Execute"** → Playbook runs step-by-step
6. **Real-time monitoring**: Live output for each step
7. Completion summary → User can continue chatting

### 8. Reports Generation
- Request report in chat (e.g., "Cost report for December") → Text streams → Rich UI with charts/tables → Export as PDF/CSV

### Cross-Flow Patterns

**Recommendation → Playbook**:
Recommendation "Apply" → Server generates playbook → Sent to chat → User executes

**Alert → Ops Chat**:
Alert "Resolve" → Opens chat with context → User takes action

**Learning Center → Ops Chat**:
Resource "Manage" → Opens chat with resource context → User performs operations

### Documentation
- [User Flows](frontend/user-flows.md) - Complete documentation with state diagrams

---

## Key Technologies

### Frontend Stack
- **Framework**: React 18+ with TypeScript
- **Desktop**: Tauri 2.0
- **State Management**: Zustand
- **Styling**: Tailwind CSS
- **Charts**: Recharts
- **WebSocket**: Native WebSocket API
- **Authentication**: AWS Cognito (Amplify)
- **Build**: Vite

### Rust Stack
- **Framework**: Tauri 2.0
- **Async Runtime**: Tokio
- **Vector DB**: Qdrant (embedded mode, qdrant-client)
- **Encryption**: AES-256-GCM (aes-gcm)
- **Key Storage**: OS Keychain (keyring)
- **AWS SDK**: aws-sdk-rust (aws-config, aws-types, aws-credential-types)
- **Serialization**: serde, serde_json
- **HTTP**: reqwest
- **Testing**: tokio-test, mockall

### Infrastructure
- **Local Storage**: Qdrant (embedded, ~20-30 MB), OS Keychain
- **Cloud Storage**: S3 (backup/restore via snapshots)
- **Authentication**: AWS Cognito
- **Server Communication**: WebSocket + HTTP

---

## Documentation Index

### Overview & Status
- [Client Overview](overview.md) - Architecture layers, responsibilities
- [Completion Status](COMPLETION-STATUS.md) - What's done, what's missing
- **This Document** - Complete summary with all UI

### Rust Modules (24 files)
- [Modules Overview](modules/overview.md)

**Storage Service** (9 docs):
- [README](modules/storage-service/README.md)
- [Architecture](modules/storage-service/architecture.md)
- [Collections](modules/storage-service/collections.md)
- [Operations](modules/storage-service/operations.md)
- [API](modules/storage-service/api.md)
- [Encryption](modules/storage-service/encryption.md)
- [Point Management](modules/storage-service/point-management.md)
- [Backup/Restore](modules/storage-service/backup-restore.md)
- [Configuration](modules/storage-service/configuration.md)
- [Testing](modules/storage-service/testing.md)

**Execution Engine** (9 docs):
- [README](modules/execution-engine/README.md)
- [Architecture](modules/execution-engine/architecture.md)
- [API](modules/execution-engine/api.md)
- [Types](modules/execution-engine/types.md)
- [Usage](modules/execution-engine/usage.md)
- [Event Handlers](modules/execution-engine/event-handlers.md)
- [Configuration](modules/execution-engine/configuration.md)
- [Error Handling](modules/execution-engine/error-handling.md)
- [Cargo Integration](modules/execution-engine/cargo-integration.md)

**Estate Scanner** (4 docs):
- [README](modules/estate-scanner/README.md)
- [Architecture](modules/estate-scanner/architecture.md)
- [Service Scanners](modules/estate-scanner/service-scanners.md)
- [Types](modules/estate-scanner/types.md)

**Common Types** (1 doc):
- [README](modules/common/README.md)

**Request Builder** (1 doc):
- [README](modules/request-builder/README.md) - Design considerations

### Frontend (4 files)
- [README](frontend/README.md) - Frontend overview
- [MVC Architecture](frontend/mvc-architecture.md)
- [User Flows](frontend/user-flows.md)
- [UI Agent Components](frontend/ui-agent-components.md)
- [Authentication & Security](frontend/authentication-security.md)

### Tauri Integration (9 files)
- [README](tauri-integration/README.md)
- [Storage Commands](tauri-integration/commands-storage.md)
- [Execution Commands](tauri-integration/commands-execution.md)
- [Estate Scanner Commands](tauri-integration/commands-estate-scanner.md)
- [Auth Commands](tauri-integration/commands-auth.md)
- [Request Builder Commands](tauri-integration/commands-request-builder.md)
- [Scan Events](tauri-integration/events-scan.md)
- [Execution Events](tauri-integration/events-execution.md)
- [System Events](tauri-integration/events-system.md)

**Total**: 42 documentation files, ~25,000 lines

---

## What's Next

### Completed (90%)
✅ All core Rust modules (Storage, Execution, Scanner, Common)
✅ Frontend MVC architecture complete
✅ All 8 user flows documented
✅ UI Agent components package designed
✅ Authentication & security complete
✅ Tauri commands documented (70+ commands)
✅ Tauri events documented (15+ events)

### Missing (10%)

#### 1. Server Integration (Priority 1)
- **WebSocket Protocol**: Message formats for streaming
- **HTTP API**: REST endpoints specification
- **UI Agent Enhancement Format**: 2-phase rendering protocol

#### 2. Data Flow & Integration (Priority 2)
- **Data Flow**: End-to-end sequences (chat, scan, execution)
- **Integration Patterns**: Frontend ↔ Rust ↔ Server
- **State Management**: Zustand stores design

#### 3. Request Builder Module (Priority 3)
- Architecture decision (Lightweight vs Rich)
- Context enrichment logic
- Server communication patterns

#### 4. Operational Readiness (Priority 4)
- Error handling strategy
- Testing strategy
- Build & deployment guide

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| First chunk latency | <100ms | Text streaming |
| Stream complete | <3s | Full text response |
| Enhancement time | <2s | Rich UI rendering |
| UI render | <100ms | Component updates |
| Tauri command | <50ms | IPC round-trip |
| Estate scan | <5 min | 10 accounts, 5 regions |
| Semantic search | <100ms | 10K resources |

---

## Design Principles

### 1. **Separation of Concerns**
- Views: Pure presentation (no business logic)
- Controllers: Business logic and orchestration
- Models: Data structures and UI state
- Services: Infrastructure abstraction

### 2. **Data Ownership**
- **Server owns**: Playbooks, recommendations, alerts
- **Rust owns**: Estate data, chat history, policies
- **Frontend owns**: Transient UI state, UI cache

### 3. **Progressive Enhancement**
- Text streams immediately (0ms wait)
- Rich UI arrives 1-2s later
- User never blocked

### 4. **Security First**
- AWS credentials never leave device
- OS Keychain for token storage
- Encrypted S3 backups (AES-256-GCM)
- JWT-based authentication

### 5. **Type Safety**
- Full TypeScript coverage (frontend)
- Strong typing in Rust (backend)
- Shared types across layers
- Runtime validation

---

## Contact & Resources

- **Architecture Docs**: `/docs/02-client/`
- **Completion Status**: [COMPLETION-STATUS.md](COMPLETION-STATUS.md)
- **Main README**: [README.md](../../README.md)

---

*This document provides a complete overview of the Escher Client architecture, covering all documented aspects: Rust modules, frontend design, Tauri integration, user flows, and UI components. For detailed specifications, refer to individual documentation files.*