# Client Modules Overview

The client is organized into separate, focused modules (pure Rust crates) for maintainability and reusability.

**Important**: Most Rust modules have been promoted to **shared services** in [/docs/04-services/](../../04-services/). See below for navigation to detailed documentation.

## Privacy-First Design

**Critical**: All Rust modules operate locally on the client. Cloud credentials (AWS/Azure/GCP) and estate data NEVER leave the device. The server is 100% stateless and stores nothing about the client's data.

For details, see [Privacy & Security Architecture](../architecture/privacy-security.md) _(coming soon)_.

## Module Status

| Module | Status | Files | Lines | Crate Name |
|--------|--------|-------|-------|------------|
| **Storage Service** | ✅ Complete | 9 | ~4,000 | `cloudops-storage-service` |
| **Execution Engine** | ✅ Complete | 9 | ~6,000 | `cloudops-execution-engine` |
| **Estate Scanner** | ✅ Complete | 4 | ~3,000 | `cloudops-estate-scanner` |
| **Common Types** | ✅ Complete | 1 | ~650 | `cloudops-common` |
| **Request Builder** | 🔄 Next | - | - | `cloudops-request-builder` |
| **Frontend** | 🔄 Pending | - | - | React + TypeScript + Tauri |

**Total Progress**: 4/5 core modules complete (80%) | ~13,650 lines of documentation

## Module Architecture

```
Client Application
├── ✅ Module 1: Storage Service (COMPLETE)
│   └── Estate storage + Chat storage + Backup
├── ✅ Module 2: Execution Engine (COMPLETE)
│   └── Cloud command execution (AWS/Azure/GCP) + History
├── ✅ Module 3: Estate Scanner (COMPLETE)
│   └── Multi-cloud resource discovery (AWS/Azure/GCP) + IAM + Embeddings
├── ✅ Module 4: Common Types (COMPLETE)
│   └── Shared types across all modules
├── 🔄 Module 5: Request Builder (NEXT)
│   └── Context enrichment + Server communication
└── 🔄 Module 6: Frontend (PENDING)
    └── React UI components
```

## Modules

### Storage Service
**Crate**: `cloudops-storage-service` | **Status**: ✅ Complete

**Documentation**: [/docs/04-services/storage-service/](../../04-services/storage-service/)

Handles all local storage operations with 6-collection RAG strategy:

**6 Qdrant Collections** (~20-30 MB embedded):
1. **Cloud Estate Inventory**: Real 384D vectors (semantic search + IAM permissions)
2. **Chat History**: Dummy 1D vectors (filter-based access)
3. **Executed Operations**: Operation history and audit trail
4. **Immutable Reports**: Cost reports, audit logs, compliance (daily sync at 2am)
5. **Alerts & Events**: Alert rules, scan results, auto-remediation settings
6. **User Playbooks**: Real 384D vectors (user-created playbooks with full scripts)

**Features**:
- S3/Blob/GCS backup management
- AES-256-GCM encryption for sensitive data
- Point management (deterministic IDs prevent duplicates)

### Execution Engine
**Crate**: `cloudops-execution-engine` | **Status**: ✅ Complete

**Documentation**: [/docs/04-services/execution-engine/](../../04-services/execution-engine/)

Pure Rust crate for multi-cloud command execution:
- **Tokio + Streaming** - Async, non-blocking with real-time output
- **Background Execution** - Returns execution_id immediately
- **Multiple Strategies** - Serial, parallel, dependency-based
- **Command Types** - Script, Exec, Shell, Cloud CLIs (AWS/Azure/GCP)
- **Event System** - Pluggable handlers (Tauri, WebSocket, logging)
- **Standardized I/O** - Type-safe with JSON schemas
- **Features** - Timeout, cancellation, retry, error handling

### Estate Scanner
**Crate**: `cloudops-estate-scanner` | **Status**: ✅ Complete

**Documentation**: [/docs/04-services/estate-scanner/](../../04-services/estate-scanner/)

Thin orchestrator for multi-cloud resource discovery and enrichment (AWS/Azure/GCP):
- **Thin Orchestrator** - Coordinates Execution Engine + Storage Service
- **Pluggable Scanners** - Trait-based architecture for all cloud providers:
  - **AWS**: EC2, RDS, S3, Lambda, VPC, IAM
  - **Azure**: VMs, SQL Database, Blob Storage, Functions, Key Vault
  - **GCP**: Compute Engine, Cloud SQL, Cloud Storage, Cloud Functions
- **IAM Discovery** - Per-resource permissions for all cloud providers:
  - **AWS**: SimulatePrincipalPolicy
  - **Azure**: Role assignments + resource permissions
  - **GCP**: IAM policy analysis
- **Semantic Embeddings** - 384D vectors using all-MiniLM-L6-v2
- **4-Level Parallelism** - Accounts → Regions → Services → Resources
- **Graceful Degradation** - Partial success on failures
- **Performance** - ~30-50s for 1000 resources

### Common Types
**Crate**: `cloudops-common` | **Status**: ✅ Complete

**Documentation**: [/docs/04-services/common/](../../04-services/common/)

Shared types and utilities used across all client modules:
- **Pure Data Structures** - No business logic, zero framework dependencies
- **Cloud-Agnostic Types** - CloudResource, IAMPermissions, UserContext, ResourceConstraints, Account
- **Multi-Cloud Support** - Works with AWS, Azure, GCP through cloud_provider field
- **Config Types** - RetryConfig, EmbeddingModelConfig
- **Error Types** - ErrorCategory
- **Type Safety** - Single source of truth, prevents duplication
- **Backward Compatible** - Designed for stability
- **Dependencies** - Only serde, chrono, serde_json

### [Request Builder](request-builder/)
**Crate**: `cloudops-request-builder`

Enriches and sends requests:
- Context enrichment (adds resource metadata + chat history)
- Server communication (HTTP client)
- Request/response handling

### Frontend
**Stack**: React + TypeScript + Tauri

User interface:
- Chat interface
- Resource explorer
- Execution monitor
- Conversation history
- Account management

## Module Dependencies

```
┌─────────────────────────────────────────────────┐
│           Frontend (React + TypeScript)         │
└─────────────────────────────────────────────────┘
                    ↓ (Tauri IPC)
┌─────────────────────────────────────────────────┐
│              Tauri Backend                       │
│  ┌──────────────────────────────────────────┐   │
│  │         Request Builder                  │   │
│  └──────────────────────────────────────────┘   │
│       ↓              ↓             ↓             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Storage  │  │ Estate   │  │Execution │      │
│  │ Service  │  │ Scanner  │  │ Engine   │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│       ↓              ↓             ↓             │
│  ┌──────────────────────────────────────────┐   │
│  │         cloudops-common                  │   │
│  │  (Shared Types: AWSResource, IAM, etc.)  │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Key Dependencies**:
- All modules depend on `cloudops-common` for shared types
- Estate Scanner uses Execution Engine (for cloud CLIs) and Storage Service (for persistence)
- Request Builder uses Storage Service (for context enrichment)
- Storage Service is the central data layer (6 collections)

## Design Principles

1. **Separation of Concerns**: Each module has one clear responsibility
2. **Reusability**: Modules can be used independently
3. **Testability**: Each module tested in isolation
4. **Clear Interfaces**: Well-defined public APIs
5. **Minimal Dependencies**: Modules depend on storage, not on each other