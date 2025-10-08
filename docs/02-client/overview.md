# Client Architecture Overview

## Purpose
The client is a Tauri-based desktop application that manages AWS estate data locally and executes operations securely.

## Key Responsibilities
1. **AWS Estate Management**: Sync and store AWS resource data locally
2. **Semantic Search**: Fast resource lookup using Qdrant vector DB
3. **Request Building**: Enrich user requests with complete resource context
4. **Execution**: Run AWS CLI commands with user approval
5. **Credential Management**: Secure storage of AWS credentials (never sent to server)

## Architecture Layers

```
┌─────────────────────────────────────┐
│         Frontend (UI)               │
│   React/Vue/Svelte + Components     │
└─────────────────────────────────────┘
              ↕
┌─────────────────────────────────────┐
│      Rust Backend (Tauri)           │
│  • Request Builder                  │
│  • Resource Lookup                  │
│  • Execution Engine                 │
│  • Sync Orchestrator                │
└─────────────────────────────────────┘
              ↕
┌─────────────────────────────────────┐
│       Local Storage                 │
│  • Qdrant (vectors)                 │
│  • AWS Estate Cache                 │
│  • OS Keychain (credentials)        │
└─────────────────────────────────────┘
```

## Modular Architecture

The client is organized into separate, focused modules (Rust crates):

### [Modules](modules/)
- **[Storage Service](modules/storage-service/)** - Estate + Chat storage with RAG, Backup management ✅
- **[Execution Engine](modules/execution-engine/)** - AWS command execution, Approval workflow 🔄
- **[Estate Scanner](modules/estate-scanner/)** - AWS resource discovery, Multi-account scanning 🔄
- **[Request Builder](modules/request-builder/)** - Context enrichment, Server communication 🔄
- **Frontend** - React UI (to be documented) 🔄

See [modules/overview.md](modules/overview.md) for complete module architecture.

## Legacy Subdirectories (Will be reorganized)
- [frontend/](frontend/) - UI layer architecture
- [backend/](backend/) - Rust core components
- [storage/](storage/) - Local data storage
- [sync/](sync/) - AWS data synchronization

---
*See [architecture.md](../../architecture.md) for complete system context.*