# V2 Architecture Documentation

## Purpose

This repository maintains the **complete architecture documentation** for the Escher Multi-Cloud Operations Platform (V2). It serves as the **single source of truth** for architectural decisions, component designs, and related repositories across the entire project ecosystem.

## Scope

This is a **10,000-foot view** documentation project that covers:

- High-level system architecture
- Component interactions and data flows
- Technology stack decisions and rationale
- Security and privacy architecture
- Performance considerations
- Integration patterns
- Repository organization and references

## Key Architecture Principles

1. **Client-side Multi-Cloud Estate Management**: All cloud resource data (AWS/Azure/GCP) and credentials remain local
2. **Server-side Operations Knowledge**: Stateless server provides playbook execution logic
3. **Privacy First**: No AWS credentials or estate data sent to server
4. **Semantic Search**: Local Qdrant vector DB enables fast, fuzzy resource lookup
5. **Precision Over Guessing**: Client sends complete context; server generates exact scripts

## Project Status

### Client Modules

| Module | Status | Files | Lines | Description |
|--------|--------|-------|-------|-------------|
| **[Storage Service](docs/02-client/modules/storage-service/)** | ✅ Complete | 9 | ~4,000 | Estate + Chat storage with RAG, IAM integration, Auto S3 backup |
| **[Execution Engine](docs/02-client/modules/execution-engine/)** | ✅ Complete | 9 | ~6,000 | Pure Rust crate for command execution with Tokio + Streaming |
| **[Estate Scanner](docs/02-client/modules/estate-scanner/)** | ✅ Complete | 4 | ~3,000 | Thin orchestrator for AWS resource discovery with IAM + embeddings |
| **[Common Types](docs/02-client/modules/common/)** | ✅ Complete | 1 | ~650 | Shared types across all modules (AWSResource, IAMPermissions, etc.) |
| **[Request Builder](docs/02-client/modules/request-builder/)** | 🔄 Next | - | - | Context enrichment, Server communication |

**Progress**: 4/5 modules complete (80%) | ~13,650 lines of documentation

### Storage Service Highlights

The Storage Service is fully designed and ready for implementation:

- **Single Qdrant Instance** - Embedded mode (~20-30 MB) for both chat and AWS estate
- **Dual Collection Strategy** - Chat history (1D dummy vectors) + AWS Estate (384D real embeddings)
- **IAM Integration** - Permissions embedded per resource (allowed/denied actions + user context)
- **Application-Level Encryption** - AES-256-GCM with OS Keychain integration
- **Point ID Management** - Random UUIDs for chat, deterministic IDs for estate (prevents duplicates)
- **Auto S3 Backup** - Background scheduler with configurable retention (7d local, 30d S3)

See [Storage Service Documentation](docs/02-client/modules/storage-service/) for complete API reference and implementation guide.

### Execution Engine Highlights

The Execution Engine is a pure Rust crate (like npm package) for executing commands:

- **Pure Rust Crate** - Reusable library, no framework dependencies, works anywhere
- **Tokio + Streaming** - Async, non-blocking execution with real-time output streaming
- **Background Execution** - Returns execution_id immediately, query status/results later
- **Multiple Strategies** - Serial, parallel, and dependency-based execution
- **Standardized I/O** - Type-safe with JSON schemas for all requests/responses
- **Event System** - Pluggable event handlers (Tauri, WebSocket, logging, etc.)
- **Cargo Integration** - Standard Rust package manager (equivalent to npm)

**Command Types**: Script, Exec, Shell, AwsCli | **Features**: Timeout, cancellation, retry, error handling

See [Execution Engine Documentation](docs/02-client/modules/execution-engine/) for complete API reference and usage examples.

### Estate Scanner Highlights

The Estate Scanner is a thin orchestrator that discovers and enriches AWS resources:

- **Thin Orchestrator** - Coordinates Execution Engine + Storage Service, no heavy lifting
- **Pluggable Scanners** - Trait-based architecture for EC2, RDS, S3, Lambda, VPC (easy to extend)
- **IAM Context-Aware** - Discovers per-resource permissions via SimulatePrincipalPolicy
- **Semantic Embeddings** - Generates 384D vectors using all-MiniLM-L6-v2 for fuzzy search
- **4-Level Parallelism** - Accounts → Regions → Services → Resources (optimized concurrency)
- **Graceful Degradation** - Continues scanning even if some services fail (partial success)

**Built-in Scanners**: EC2, RDS, S3, Lambda, VPC | **Performance**: ~30-50s for 1000 resources

See [Estate Scanner Documentation](docs/02-client/modules/estate-scanner/) for complete architecture and scanner implementations.

### Common Types Highlights

The Common Types module provides shared data structures used across all client modules:

- **Single Source of Truth** - All shared types in one crate (`cloudops-common`)
- **Zero Framework Dependencies** - Pure data structures with only serde, chrono
- **8 Core Types** - AWSResource, IAMPermissions, UserContext, ResourceConstraints, Account, RetryConfig, EmbeddingModelConfig, ErrorCategory
- **Type Safety Guaranteed** - Prevents duplication and incompatibility across modules
- **Well Documented** - Comprehensive docs with JSON schemas and usage examples
- **Backward Compatible** - Designed for stability and extensibility

**Used By**: Storage Service, Estate Scanner, Request Builder | **Dependencies**: serde, serde_json, chrono

See [Common Types Documentation](docs/02-client/modules/common/) for complete type definitions and usage patterns.

## Repository Structure

- [architecture.md](architecture.md) - Main architecture document with detailed system design
- [docs/](docs/) - Complete documentation organized by component/domain
  - [01-overview/](docs/01-overview/) - System overview, key decisions, tech stack
  - [02-client/](docs/02-client/) - Client architecture (Tauri app)
    - [modules/](docs/02-client/modules/) - Modular client architecture
  - [03-server/](docs/03-server/) - Server ecosystem (agents, microservices)
  - [04-flows/](docs/04-flows/) - Request, sync, execution flows
  - [05-security/](docs/05-security/) - Security and privacy architecture
  - [06-data/](docs/06-data/) - Data models, schemas, API contracts
  - [07-operations/](docs/07-operations/) - Deployment, monitoring, DR
- [working-docs/](working-docs/) - Active design documents
- [reference/](reference/) - Reference implementations and comparisons
- [diagrams/](diagrams/) - System, flow, and component diagrams
- [adr/](adr/) - Architecture Decision Records

## Guidelines for Contributors

- Maintain architectural purity: document **what** and **why**, not implementation details
- Keep diagrams and flows updated with any architectural changes
- Reference related repositories but avoid duplicating implementation docs
- Focus on system-level decisions, not code-level specifics

## Related Repositories

- [Link to UI repository]
- [Link to agent repository]
- [Link to server repository]
- [Link to playbook repository]

---

**Note**: This is architectural documentation. For implementation details, refer to specific component repositories.
