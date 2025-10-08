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

## Subdirectories
- [frontend/](frontend/) - UI layer architecture
- [backend/](backend/) - Rust core components
- [storage/](storage/) - Local data storage
- [sync/](sync/) - AWS data synchronization

---
*See [architecture.md](../../architecture.md) for complete system context.*