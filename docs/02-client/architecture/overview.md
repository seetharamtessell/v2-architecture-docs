# Client Architecture Overview

**Status**: 90% Complete | [See Completion Status →](../meta/COMPLETION-STATUS.md)
**Last Updated**: October 2025

## Purpose
The client is a Tauri-based desktop application that manages multi-cloud estate data locally (AWS/Azure/GCP) and executes operations securely.

**Multi-Cloud Note**: This documentation uses AWS, Azure, and GCP examples throughout. All features support multi-cloud operations through the `cloud_provider` field in resource metadata and API requests. The architecture is cloud-agnostic, with pluggable cloud provider implementations.

## Key Responsibilities
1. **Cloud Estate Management**: Sync and store cloud resources (AWS/Azure/GCP) locally
2. **Semantic Search**: Fast resource lookup using Qdrant vector DB
3. **Request Building**: Enrich user requests with complete resource context
4. **Execution**: Run cloud CLI commands (AWS/Azure/GCP) with user approval
5. **Credential Management**: Secure storage of cloud credentials (AWS/Azure/GCP) - never sent to server
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

### [Modules](../modules/) (Rust Crates)

**Note**: Most Rust modules have been promoted to shared services in [/docs/04-services/](../../04-services/). See [Modules Overview](../modules/overview.md) for navigation.

| Module | Status | Documentation Location |
|--------|--------|----------------------|
| **[Storage Service](../../04-services/storage-service/)** | ✅ Complete | 6-collection RAG strategy, S3/Blob/GCS backup (9 docs) |
| **[Execution Engine](../../04-services/execution-engine/)** | ✅ Complete | Command execution with Tokio + Streaming (9 docs) |
| **[Estate Scanner](../../04-services/estate-scanner/)** | ✅ Complete | Multi-cloud resource discovery (AWS/Azure/GCP) (4 docs) |
| **[Request Builder](../modules/request-builder/)** | ⚠️ Design | Context enrichment, Server communication (pending) |
| **[Common Types](../../04-services/common/)** | ✅ Complete | Shared data structures across modules (1 doc) |

### [Frontend](../frontend/) (React + TypeScript)

| Component | Status | Description |
|-----------|--------|-------------|
| **[MVC Architecture](../frontend/mvc-architecture.md)** | ✅ Complete | Models, Views, Controllers, Services |
| **[User Flows](../frontend/user-flows.md)** | ✅ Complete | 8 complete user journeys |
| **[UI Components Package](../frontend/ui-components.md)** | ✅ Complete | 30+ React presentation components |
| **[UI Rendering Engine](../frontend/ui-rendering-engine.md)** | ✅ Complete | Client-side rendering orchestrator |
| **[Authentication](../frontend/authentication-security.md)** | ✅ Complete | Cognito + JWT + OS Keychain |

### [Tauri Integration](../tauri-integration/) (IPC Bridge)

| Component | Status | Description |
|-----------|--------|-------------|
| **[Commands](../tauri-integration/README.md)** | ✅ Complete | 70+ Tauri commands (5 docs) |
| **[Events](../tauri-integration/README.md#event-categories)** | ✅ Complete | 15+ Tauri events (3 docs) |
| **Setup Guides** | ⏳ Pending | Configuration and build docs |

### [UI Team Implementation Guide](../ui-team-implementation/) (Parallel Development)

| Component | Status | Description |
|-----------|--------|-------------|
| **[Complete Guide](../ui-team-implementation/README.md)** | ✅ Complete | Full independent implementation guide (6 docs, ~22,700 lines) |
| **Architecture** | ✅ Complete | MVC pattern, data flow, component structure |
| **Implementation Plan** | ✅ Complete | 7 phases with code examples |
| **Project Structure** | ✅ Complete | Complete folder layout |
| **Mock Contracts** | ✅ Complete | TypeScript interfaces, 70+ commands |
| **Claude Prompts** | ✅ Complete | 25 ready-to-use development prompts |

**Enables**: UI team can build 100% independently with zero dependencies on Platform/Server teams

---

**Progress**: 90% complete (4 modules + frontend + Tauri integration + UI guide) | ~48,000 lines of documentation

See [COMPLETION-STATUS.md](../meta/COMPLETION-STATUS.md) for detailed progress tracking.
See [Architecture Summary](./summary.md) for complete architecture overview.

---

## Privacy & Security

**Critical**: All Rust modules operate locally on the client. Cloud credentials and estate data NEVER leave the device. The server is 100% stateless and stores nothing about the client's data.

For details, see Privacy & Security Architecture _(coming soon)_.

---

*See [/PROJECT-SUMMARY.md](../../../PROJECT-SUMMARY.md) for complete system context.*