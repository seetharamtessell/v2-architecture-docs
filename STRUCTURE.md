# Documentation Structure

## Root Level Files
- [README.md](README.md) - Project overview and getting started
- [CLAUDE.md](CLAUDE.md) - AI assistant guidance for working with this repository
- **[PROJECT-SUMMARY.md](PROJECT-SUMMARY.md)** - Complete project summary (998 lines, comprehensive overview)
- [STRUCTURE.md](STRUCTURE.md) - This file (documentation organization)

---

## Main Documentation (`docs/`)

### 01-overview/
High-level architecture and product vision:
- **[PRODUCT-VISION.md](docs/01-overview/PRODUCT-VISION.md)** - Complete product vision with deployment models
- [architecture.md](docs/01-overview/architecture.md) - Multi-cloud architecture overview
- [system-overview.md](docs/01-overview/system-overview.md) - System overview for stakeholders
- [technology-stack.md](docs/01-overview/technology-stack.md) - Technology decisions
- [key-decisions.md](docs/01-overview/key-decisions.md) - Architecture Decision Records

### 02-client/
Client application (Tauri desktop app):
- [overview.md](docs/02-client/overview.md) - Client architecture overview
- **[CLIENT-SUMMARY.md](docs/02-client/CLIENT-SUMMARY.md)** - Complete client architecture summary

#### Frontend (`docs/02-client/frontend/`)
React + TypeScript MVC architecture

#### Tauri Integration (`docs/02-client/tauri-integration/`)
IPC bridge: Tauri commands (70+) and events (15+)

#### UI Team Implementation (`docs/02-client/ui-team-implementation/`)
**Complete parallel development guide** (~22,700 lines, 6 files)

### 03-server/
Server architecture (stateless AI system):
- **[agents/playbook-agent.md](docs/03-server/agents/playbook-agent.md)** - ✅ Complete (~2,227 lines)
  - LLM + RAG intelligence with 4-step flow
  - 10 playbook lifecycle states

### 04-services/
**Shared Rust crates** used by both client and server:
- **storage-service** - ✅ Complete (~4,000 lines, 10 files) - RAG + Qdrant
- **execution-engine** - ✅ Complete (~6,000 lines) - Command execution
- **estate-scanner** - ✅ Complete (~3,000 lines) - Resource discovery
- **common** - ✅ Complete (~650 lines) - Shared data structures
- **playbook-service** - ✅ Complete - Playbook management

### 05-flows/
Request flows, sync flows, execution flows

### 06-security/
Security and privacy architecture

### 07-data/
Data models, schemas, API contracts

### 08-operations/
Deployment, monitoring, disaster recovery

---

## Working Documents (`working-docs/`)
Active design documents and completed fix records:
- **[CRITICAL-FIXES-2025-10-15-COMPLETED.md](working-docs/CRITICAL-FIXES-2025-10-15-COMPLETED.md)** - Record of 61 fixes

---

## Archive (`archive/`)
Resolved gap analyses:
- **[GAP-ANALYSIS-2025-10-14-RESOLVED.md](archive/GAP-ANALYSIS-2025-10-14-RESOLVED.md)** - All critical issues resolved

---

## Reference Material (`reference/`)
Reference implementations and comparisons

## Architecture Decision Records (`adr/`)
ADR templates and records

## Diagrams (`diagrams/`)
System, component, and flow diagrams

## Whiteboard Images (`whiteboard-images/`)
Whiteboard photos and sketches from architecture discussions:
- [README.md](whiteboard-images/README.md) - Organization and naming conventions
- Photos organized by date (YYYY-MM-DD format)
- Captures design discussions and brainstorming sessions

## Repositories (`repositories/`)
Related repository links

---

## Documentation Status

### ✅ Complete (90%)
- Product Vision (3 files, ~1,500 lines)
- Client Frontend (25+ files, ~25,000 lines)
- UI Implementation Guide (6 files, ~22,700 lines)
- Shared Services (28 files, ~13,650 lines)
- Server: Playbook Agent (1 file, ~2,227 lines)
- PROJECT-SUMMARY (1 file, ~998 lines)

**Total**: 57+ files, 42,377+ lines documented

### 🔄 In Progress
- Server: Classification, Operations, Validation, Risk Assessment agents
- Client: Request Builder module

---

## Key Architecture Highlights

### 6 RAG Collections (Client-Side)
1. Cloud Estate Inventory (multi-cloud: AWS/Azure/GCP)
2. Chat History
3. Executed Operations
4. Immutable Reports
5. Alerts & Events
6. User Playbooks

### Multi-Cloud Architecture
- **Product**: Escher Multi-Cloud Operations Platform
- **Clouds**: AWS, Azure, GCP
- **Naming**: `cloud_estate` (not `aws_estate`)

### Deployment Options
1. Run on Your Laptop - Local-only
2. Extend to Your Cloud - 24/7 automation

---

## Quick Start for New Contributors

1. **Start Here**: [PROJECT-SUMMARY.md](PROJECT-SUMMARY.md)
2. **Product Vision**: [PRODUCT-VISION.md](docs/01-overview/PRODUCT-VISION.md)
3. **Client Architecture**: [CLIENT-SUMMARY.md](docs/02-client/CLIENT-SUMMARY.md)
4. **UI Development**: [ui-team-implementation/](docs/02-client/ui-team-implementation/)
5. **Shared Services**: [docs/04-services/](docs/04-services/)

---

**Last Updated**: October 15, 2025
**Status**: 90% complete. All critical inconsistencies resolved. Ready for stakeholder review.