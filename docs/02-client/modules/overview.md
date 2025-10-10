# Client Modules Overview

The client is organized into separate, focused modules (pure Rust crates) for maintainability and reusability.

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
│   └── AWS command execution + History
├── ✅ Module 3: Estate Scanner (COMPLETE)
│   └── AWS resource discovery + IAM + Embeddings
├── ✅ Module 4: Common Types (COMPLETE)
│   └── Shared types across all modules
├── 🔄 Module 5: Request Builder (NEXT)
│   └── Context enrichment + Server communication
└── 🔄 Module 6: Frontend (PENDING)
    └── React UI components
```

## Modules

### [Storage Service](storage-service/)
**Crate**: `cloudops-storage-service`

Handles all local storage operations:
- Estate storage with RAG (semantic search)
- Chat context and conversation history
- S3 backup management
- Encryption (AES-256-GCM)

### [Execution Engine](execution-engine/)
**Crate**: `cloudops-execution-engine` | **Status**: ✅ Complete

Pure Rust crate for command execution (like npm package):
- **Tokio + Streaming** - Async, non-blocking with real-time output
- **Background Execution** - Returns execution_id immediately
- **Multiple Strategies** - Serial, parallel, dependency-based
- **Command Types** - Script, Exec, Shell, AwsCli
- **Event System** - Pluggable handlers (Tauri, WebSocket, logging)
- **Standardized I/O** - Type-safe with JSON schemas
- **Features** - Timeout, cancellation, retry, error handling

[Complete Documentation →](execution-engine/)

### [Estate Scanner](estate-scanner/)
**Crate**: `cloudops-estate-scanner` | **Status**: ✅ Complete

Thin orchestrator for AWS resource discovery and enrichment:
- **Thin Orchestrator** - Coordinates Execution Engine + Storage Service
- **Pluggable Scanners** - Trait-based: EC2, RDS, S3, Lambda, VPC (extensible)
- **IAM Discovery** - Per-resource permissions via SimulatePrincipalPolicy
- **Semantic Embeddings** - 384D vectors using all-MiniLM-L6-v2
- **4-Level Parallelism** - Accounts → Regions → Services → Resources
- **Graceful Degradation** - Partial success on failures
- **Performance** - ~30-50s for 1000 resources

[Complete Documentation →](estate-scanner/)

### [Common Types](common/)
**Crate**: `cloudops-common` | **Status**: ✅ Complete

Shared types and utilities used across all client modules:
- **Pure Data Structures** - No business logic, zero framework dependencies
- **AWS Types** - AWSResource, IAMPermissions, UserContext, ResourceConstraints, Account
- **Config Types** - RetryConfig, EmbeddingModelConfig
- **Error Types** - ErrorCategory
- **Type Safety** - Single source of truth, prevents duplication
- **Backward Compatible** - Designed for stability
- **Dependencies** - Only serde, chrono, serde_json

[Complete Documentation →](common/)

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
- Estate Scanner uses Execution Engine (for AWS CLI) and Storage Service (for persistence)
- Request Builder uses Storage Service (for context enrichment)
- Storage Service is the central data layer

## Design Principles

1. **Separation of Concerns**: Each module has one clear responsibility
2. **Reusability**: Modules can be used independently
3. **Testability**: Each module tested in isolation
4. **Clear Interfaces**: Well-defined public APIs
5. **Minimal Dependencies**: Modules depend on storage, not on each other