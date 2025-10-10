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

The client is organized into separate, focused modules (pure Rust crates):

### [Modules](modules/)

| Module | Status | Description |
|--------|--------|-------------|
| **[Storage Service](modules/storage-service/)** | ✅ Complete | Estate + Chat storage with RAG, IAM integration, Auto S3 backup |
| **[Execution Engine](modules/execution-engine/)** | ✅ Complete | Pure Rust crate for command execution with Tokio + Streaming |
| **[Estate Scanner](modules/estate-scanner/)** | 🔄 Next | AWS resource discovery, Multi-account scanning |
| **[Request Builder](modules/request-builder/)** | 🔄 Pending | Context enrichment, Server communication |
| **Frontend** | 🔄 Pending | React UI (to be documented) |

**Progress**: 2/4 core modules complete (50%) | ~10,000 lines of documentation

See [modules/overview.md](modules/overview.md) for complete module architecture and dependencies.

## Legacy Subdirectories (Will be reorganized)
- [frontend/](frontend/) - UI layer architecture
- [backend/](backend/) - Rust core components
- [storage/](storage/) - Local data storage
- [sync/](sync/) - AWS data synchronization

---
*See [architecture.md](../../architecture.md) for complete system context.*