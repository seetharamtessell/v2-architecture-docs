# Documentation Structure

## Root Level
- [README.md](README.md) - Project overview and purpose
- [CLAUDE.md](CLAUDE.md) - AI assistant guidance
- [STRUCTURE.md](STRUCTURE.md) - This file

## Working Documents (`working-docs/`)
Active design documents being worked on:
- [CLIENT-DESIGN-WORKING-DOC-V2.md](working-docs/CLIENT-DESIGN-WORKING-DOC-V2.md) - Storage Service design (latest)
- [CLIENT-MODULE-ARCHITECTURE.md](working-docs/CLIENT-MODULE-ARCHITECTURE.md) - Module architecture overview
- [DOCS-STRUCTURE.md](working-docs/DOCS-STRUCTURE.md) - Documentation organization plan

## Reference Material (`reference/`)
Reference implementations and comparisons:
- [CONTEXT-MANAGEMENT-ARCHITECTURE.md](reference/CONTEXT-MANAGEMENT-ARCHITECTURE.md) - Node.js reference implementation
- [COMPARISON-NODE-VS-RUST-STORAGE.md](reference/COMPARISON-NODE-VS-RUST-STORAGE.md) - Rust vs Node.js comparison

## Architecture Documentation (`docs/`)

### 01-overview
High-level architecture and decisions:
- [architecture.md](docs/01-overview/architecture.md) - Original 10,000-foot architecture
- [system-overview.md](docs/01-overview/system-overview.md)
- [technology-stack.md](docs/01-overview/technology-stack.md)
- [key-decisions.md](docs/01-overview/key-decisions.md)

### 02-client
Client application documentation:
- [overview.md](docs/02-client/overview.md) - Client overview

#### Modules (`docs/02-client/modules/`)
- [overview.md](docs/02-client/modules/overview.md) - Module architecture

**Storage Service** (✅ Design in progress):
- [README.md](docs/02-client/modules/storage-service/README.md)

**Execution Engine** (🔄 To be designed):
- [README.md](docs/02-client/modules/execution-engine/README.md)
- [architecture.md](docs/02-client/modules/execution-engine/architecture.md)
- [api.md](docs/02-client/modules/execution-engine/api.md)

**Estate Scanner** (🔄 To be designed):
- [README.md](docs/02-client/modules/estate-scanner/README.md)

**Request Builder** (🔄 To be designed):
- [README.md](docs/02-client/modules/request-builder/README.md)

### 03-server
Server documentation:
- [overview.md](docs/03-server/overview.md)
- [agents/overview.md](docs/03-server/agents/overview.md)
- [microservices/overview.md](docs/03-server/microservices/overview.md)

## Architecture Decision Records (`adr/`)
- [README.md](adr/README.md)
- [template.md](adr/template.md)

## Diagrams (`diagrams/`)
- [README.md](diagrams/README.md)
- `components/` - Component diagrams
- `flows/` - Flow diagrams
- `system/` - System diagrams
- `source/` - Diagram source files

## Repositories (`repositories/`)
- [repos.md](repositories/repos.md) - Related repository links

## Design Status

| Module | Status | Working Doc | Architecture Doc |
|--------|--------|-------------|------------------|
| Storage Service | 🟡 In Progress | [Link](working-docs/CLIENT-DESIGN-WORKING-DOC-V2.md) | TBD |
| Execution Engine | ⚪ Not Started | TBD | [Link](docs/02-client/modules/execution-engine/architecture.md) |
| Estate Scanner | ⚪ Not Started | TBD | TBD |
| Request Builder | ⚪ Not Started | TBD | TBD |

## Next Steps

1. Complete Storage Service design in working doc
2. Move finalized design to `docs/02-client/modules/storage-service/`
3. Start Estate Scanner design
4. Start Request Builder design
5. Keep Execution Engine on hold until needed