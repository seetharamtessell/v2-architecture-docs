# Client Modules Overview

The client is organized into separate, focused modules (Rust crates) for maintainability and reusability.

## Module Architecture

```
Client Application
├── Module 1: Storage Service
│   └── Estate storage + Chat storage + Backup
├── Module 2: Execution Engine
│   └── AWS command execution + History
├── Module 3: Estate Scanner
│   └── AWS API integration + Resource discovery
├── Module 4: Request Builder
│   └── Context enrichment + Server communication
└── Module 5: Frontend
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
**Crate**: `cloudops-execution-engine`

Executes AWS operations:
- AWS CLI/SDK command execution
- Execution history and logging
- Approval workflow
- Real-time output streaming

### [Estate Scanner](estate-scanner/)
**Crate**: `cloudops-estate-scanner`

Discovers AWS resources:
- Multi-account AWS scanning
- EC2, RDS, S3, Lambda, etc.
- Rate limiting and pagination
- Incremental sync strategy

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
Frontend (React)
    ↓ (Tauri IPC)
Tauri Backend
    ├─→ Storage Service ←─┐
    ├─→ Execution Engine   │ (uses for logs)
    ├─→ Estate Scanner     │
    └─→ Request Builder ───┘ (uses for data)
```

## Design Principles

1. **Separation of Concerns**: Each module has one clear responsibility
2. **Reusability**: Modules can be used independently
3. **Testability**: Each module tested in isolation
4. **Clear Interfaces**: Well-defined public APIs
5. **Minimal Dependencies**: Modules depend on storage, not on each other