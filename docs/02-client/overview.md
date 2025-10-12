# Client Architecture Overview

**Status**: 90% Complete | [See Completion Status →](./COMPLETION-STATUS.md)
**Last Updated**: October 2025

## Purpose
The client is a Tauri-based desktop application that manages AWS estate data locally and executes operations securely.

## Key Responsibilities
1. **AWS Estate Management**: Sync and store AWS resource data locally
2. **Semantic Search**: Fast resource lookup using Qdrant vector DB
3. **Request Building**: Enrich user requests with complete resource context
4. **Execution**: Run AWS CLI commands with user approval
5. **Credential Management**: Secure storage of AWS credentials (never sent to server)
6. **UI Rendering**: Server-driven UI with dynamic component system

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

The client is organized into separate, focused modules:

### [Modules](modules/) (Rust Crates)

| Module | Status | Description |
|--------|--------|-------------|
| **[Storage Service](modules/storage-service/)** | ✅ Complete | Estate + Chat storage with RAG, S3 backup (9 docs) |
| **[Execution Engine](modules/execution-engine/)** | ✅ Complete | Command execution with Tokio + Streaming (9 docs) |
| **[Estate Scanner](modules/estate-scanner/)** | ✅ Complete | AWS resource discovery, Multi-account scanning (4 docs) |
| **[Request Builder](modules/request-builder/)** | ⚠️ Design | Context enrichment, Server communication (pending) |
| **[Common Types](modules/common/)** | ✅ Complete | Shared data structures across modules (1 doc) |

### [Frontend](frontend/) (React + TypeScript)

| Component | Status | Description |
|-----------|--------|-------------|
| **[MVC Architecture](frontend/mvc-architecture.md)** | ✅ Complete | Models, Views, Controllers, Services |
| **[User Flows](frontend/user-flows.md)** | ✅ Complete | 8 complete user journeys |
| **[UI Agent Components](frontend/ui-agent-components.md)** | ✅ Complete | Dynamic rendering system (30+ components) |
| **[Authentication](frontend/authentication-security.md)** | ✅ Complete | Cognito + JWT + OS Keychain |

### [Tauri Integration](tauri-integration/) (IPC Bridge)

| Component | Status | Description |
|-----------|--------|-------------|
| **[Commands](tauri-integration/README.md)** | ✅ Complete | 70+ Tauri commands (5 docs) |
| **[Events](tauri-integration/README.md#event-categories)** | ✅ Complete | 15+ Tauri events (3 docs) |
| **Setup Guides** | ⏳ Pending | Configuration and build docs |

### [UI Team Implementation Guide](ui-team-implementation/) (Parallel Development)

| Component | Status | Description |
|-----------|--------|-------------|
| **[Complete Guide](ui-team-implementation/README.md)** | ✅ Complete | Full independent implementation guide (6 docs, ~22,700 lines) |
| **Architecture** | ✅ Complete | MVC pattern, data flow, component structure |
| **Implementation Plan** | ✅ Complete | 7 phases with code examples |
| **Project Structure** | ✅ Complete | Complete folder layout |
| **Mock Contracts** | ✅ Complete | TypeScript interfaces, 70+ commands |
| **Claude Prompts** | ✅ Complete | 25 ready-to-use development prompts |

**Enables**: UI team can build 100% independently with zero dependencies on Platform/Server teams

---

**Progress**: 90% complete (4 modules + frontend + Tauri integration + UI guide) | ~48,000 lines of documentation

See [COMPLETION-STATUS.md](./COMPLETION-STATUS.md) for detailed progress tracking.
See [CLIENT-SUMMARY.md](./CLIENT-SUMMARY.md) for complete architecture overview.

---
*See [architecture.md](../../architecture.md) for complete system context.*