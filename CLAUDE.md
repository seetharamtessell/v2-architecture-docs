# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the **v2-architecture-docs** repository - the single source of truth for architectural documentation of the Escher Multi-Cloud Operations Platform (V2). This is a **10,000-foot view documentation project**, not an implementation repository.

## Key Principles

1. **Architectural Purity**: Document the "what" and "why", not implementation details or code-level specifics
2. **High-Level Focus**: Maintain the big picture view across the entire system ecosystem
3. **Client-Server Separation**: The architecture has two distinct worlds:
   - **Client**: Tauri-based app with local cloud estate (AWS/Azure/GCP), Qdrant vector DB, and secure credential storage
   - **Server**: Multi-agent system with services for operations intelligence (NO cloud credentials/estate stored)

## Architecture Overview

The system architecture is based on these core principles:
- **Client-side Cloud Estate Management**: All cloud resource data (AWS/Azure/GCP) and credentials stay local
- **Server-side Operations Knowledge**: Stateless server provides playbook execution logic and LLM intelligence
- **Privacy First**: Cloud credentials and estate data never sent to server
- **Semantic Search**: Local Qdrant vector DB enables fast, fuzzy resource lookup across all clouds
- **Precision Over Guessing**: Client sends complete context; server generates exact scripts

## Critical Architecture Corrections (October 2025)

**IMPORTANT**: Recent documentation review revealed and corrected fundamental architectural misunderstandings. When working on this codebase, remember:

### Parameter Extraction Flow (CRITICAL)

**CORRECT Flow**:
```
Client → Master Agent (extracts parameters using LLM) →
Playbook Agent (receives extracted parameters, searches playbooks) →
Client (displays with pre-filled + placeholder parameters) →
User (manually fills empty placeholders) →
Client (executes scripts locally)
```

**Key Points**:
- ✅ **Master Agent** (server LLM) extracts parameters from user prompt + estate context
- ✅ **Playbook Agent** receives parameters ALREADY EXTRACTED by Master Agent
- ✅ **Client** has NO LLM access, does NOT extract parameters, does NOT auto-fill
- ✅ **User** manually fills any empty parameter fields
- ✅ **Scripts** are embedded in response, NOT downloaded from S3 by client

**Common Mistakes to Avoid**:
- ❌ Saying "client extracts parameters" or "client auto-fills parameters"
- ❌ Saying "client downloads scripts from S3"
- ❌ Using terminology like "auto_fill_strategy" (use "extraction_hint" instead)
- ❌ Implying client has LLM capabilities

**Correct Terminology**:
- `extraction_hint` - tells Master Agent where to look for values
- "pre-filled by Master Agent" - parameters extracted by Master Agent
- "placeholder" - empty parameter that user must fill manually
- "embedded scripts" - scripts included in API response payload

### Agent Responsibilities

| Agent | Location | Has LLM? | Responsibility |
|-------|----------|----------|----------------|
| **Master Agent** | Server | ✅ Yes | Intent classification + **parameter extraction** |
| **Playbook Agent** | Server | ✅ Yes | Search & ranking (receives extracted params) |
| **Client** | Local | ❌ No | Display + user input + execution only |

## Directory Structure

```
docs/
├── INDEX.md              # Main entry point (product specification, 479 lines)
├── 01-overview/          # System overview, key decisions, tech stack
├── 02-client/            # Client architecture (Tauri app)
│   ├── INDEX.md          # Client-specific navigation (developer-first)
│   ├── CLIENT-SUMMARY.md # Complete architecture summary with all UI
│   ├── modules/          # Client-specific modules
│   │   ├── overview.md
│   │   └── request-builder/      🔄 Next (0% - to be designed)
│   ├── frontend/         # Frontend architecture (React + TypeScript)
│   │   ├── mvc-architecture.md   ✅ Complete (MVC pattern)
│   │   ├── user-flows.md         ✅ Complete (8 flows)
│   │   ├── ui-components.md      ✅ Complete (30+ components)
│   │   ├── ui-rendering-engine.md ✅ Complete (orchestrator)
│   │   └── authentication-security.md ✅ Complete (Cognito + JWT)
│   ├── tauri-integration/ # IPC Bridge (Frontend ↔ Rust)
│   │   ├── commands-storage.md   ✅ Complete (25+ commands)
│   │   ├── commands-execution.md ✅ Complete (15+ commands)
│   │   ├── commands-estate-scanner.md ✅ Complete (15+ commands)
│   │   ├── commands-auth.md      ✅ Complete (15+ commands)
│   │   ├── events-scan.md        ✅ Complete (6 events)
│   │   ├── events-execution.md   ✅ Complete (6 events)
│   │   └── events-system.md      ✅ Complete (8 events)
│   └── ui-team-implementation/   ✅ Complete parallel dev guide
│       └── 05-claude-prompts.md  # 25 ready-to-use prompts
├── 03-server/            # Server ecosystem
│   ├── agents/           # Multi-agent system (organized by category)
│   │   ├── core/                 # Request processing agents
│   │   ├── intelligence/         # LLM + RAG agents
│   │   │   ├── playbook-agent.md          ✅ Complete (~4,637 lines)
│   │   │   └── playbook-best-practices.md ✅ Complete
│   │   ├── validation/           # Safety and risk agents
│   │   └── presentation/         # UI generation agents
│   │       └── ui-agent.md       ✅ Complete - Server-side UI intelligence
│   └── (microservices moved to 04-services/services/)
├── 04-services/          # NEW STRUCTURE (October 2025)
│   ├── libraries/        # Reusable Rust crates
│   │   ├── common/               ✅ Complete (~650 lines)
│   │   ├── storage-service/      ✅ Complete (~4,000 lines, 10 files)
│   │   ├── execution-engine/     ✅ Complete (~6,000 lines, 9 files)
│   │   └── estate-scanner/       ✅ Complete (~3,000 lines, 4 files)
│   └── services/         # Running services (Rust/Go)
│       ├── playbook-service/     ✅ Complete
│       ├── api-gateway/          📋 Planned
│       ├── auth-service/         📋 Planned
│       ├── rag-service/          📋 Planned
│       ├── script-generator/     📋 Planned
│       ├── workflow-engine/      📋 Planned
│       └── notification-service/ 📋 Planned
├── 05-flows/             # Request, sync, execution flows
├── 06-security/          # Security and privacy architecture
├── 07-data/              # Data models, schemas, API contracts
├── 08-operations/        # Deployment, monitoring, DR
└── meta/                 # Documentation metadata
    └── CONTRIBUTING.md   # Contribution guidelines

working-docs/             # Active design documents
adr/                      # Architecture Decision Records
├── template.md
└── 001-documentation-index-architecture.md  ✅ Accepted

diagrams/
├── system/               # Overall architecture diagrams
├── flows/                # Sequence and flow diagrams
└── components/           # Service topology maps
```

## Recent Structural Changes (October 2025)

### 04-services/ Reorganization

**Important**: The `04-services/` folder structure was recently reorganized to distinguish between libraries and services:

**Old Structure**:
```
04-services/
├── common/
├── storage-service/
├── execution-engine/
├── estate-scanner/
└── playbook-service/
```

**New Structure** (Current):
```
04-services/
├── libraries/          # Reusable Rust crates
│   ├── common/
│   ├── storage-service/
│   ├── execution-engine/
│   └── estate-scanner/
└── services/           # Running services
    └── playbook-service/
```

**Key Distinction**:
- **Libraries** = Reusable Rust crates (compiled into applications)
- **Services** = Running services (deployed as standalone processes)

When referencing paths in documentation:
- ✅ Use `04-services/libraries/storage-service/`
- ❌ Not `04-services/storage-service/`

### Deleted Folders

- `03-server/microservices/` - Consolidated into `04-services/services/`

## Documentation Guidelines

### When Adding Documentation

1. **Focus on Architecture**: Describe component interactions, data flows, and system-level decisions
2. **Avoid Implementation Details**: Don't document code structure, function signatures, or line-by-line logic
3. **Server Complexity**: The server side is a complex ecosystem with multiple agents and services - give it proper depth
4. **Maintain Separation**: Keep client and server concerns clearly separated
5. **Update Diagrams**: When architecture changes, update both diagrams and their source files
6. **Validate Architectural Correctness**: Always verify who does what (especially parameter extraction!)

### Documentation Entry Points

| Purpose | File | Audience |
|---------|------|----------|
| **Product Specification Index** | [docs/INDEX.md](docs/INDEX.md) | PM, architects, developers, AI (70% spec) |
| **Comprehensive Narrative** | [PROJECT-SUMMARY.md](PROJECT-SUMMARY.md) | All audiences (998 lines deep dive) |
| **Developer Navigation** | [docs/02-client/INDEX.md](docs/02-client/INDEX.md) | Frontend/backend developers |
| **File Organization** | [STRUCTURE.md](STRUCTURE.md) | Contributors |
| **Development Guidance** | CLAUDE.md (this file) | Developers, AI assistants |
| **Contribution Guidelines** | [docs/meta/CONTRIBUTING.md](docs/meta/CONTRIBUTING.md) | All contributors |

### Document Quality Checklist

Before committing documentation changes, verify:
- [ ] Client responsibilities clearly separated from server responsibilities
- [ ] No client-side parameter extraction references (only Master Agent extracts)
- [ ] Correct terminology: "extraction_hint" not "auto_fill_strategy"
- [ ] Multi-cloud context: "cloud estate" not "AWS estate" (unless AWS-specific)
- [ ] Scripts embedded in response, not downloaded by client
- [ ] Correct path references after 04-services/ reorganization
- [ ] Consistent with [PROJECT-SUMMARY.md](PROJECT-SUMMARY.md) architecture

### When to Create ADRs

Create an Architecture Decision Record (ADR) for:
- Technology stack choices (e.g., Tauri over Electron)
- Architectural patterns (e.g., stateless server, local vector DB)
- Security models
- Data storage strategies
- Integration approaches
- Documentation structure decisions

Use the [adr/template.md](adr/template.md) for consistency.

### Markdown Conventions

- Use relative links between documentation files
- Include ASCII diagrams for flows and component interactions
- Keep documentation hierarchical: overview → detail
- Update [docs/INDEX.md](docs/INDEX.md) when adding/changing sections
- Follow style guide in [docs/meta/CONTRIBUTING.md](docs/meta/CONTRIBUTING.md)

## Key Architecture Concepts

### Request Flow
1. **Client**: User prompt → Semantic search in local Qdrant → Resource identification with full metadata
2. **Client**: Build enriched request with complete context (account, region, permissions, constraints)
3. **Server - Master Agent**: Receive complete context → Classify intent → **Extract parameters using LLM**
4. **Server - Playbook Agent**: Receive extracted parameters → RAG search → LLM ranking → Return playbook
5. **Client**: Receive playbook with pre-filled + placeholder parameters → User fills placeholders → Display + approval → Execute locally

### Data Flow
- **Cloud estate sync**: Periodic sync (6h full, 15min incremental) from AWS/Azure/GCP APIs → Transform → Embed → Qdrant
- **Credentials**: Stored in OS Keychain, never leave client
- **Playbooks**: Stored in S3 on server, indexed in server-side Qdrant, scripts embedded in API response

### Client Storage Service (6 RAG Collections)
- **Single Qdrant Instance**: Embedded mode, ~20-30 MB
- **Collections**:
  1. **Cloud Estate Inventory** (Real 384D vectors) - AWS/Azure/GCP resources with semantic search + IAM permissions
  2. **Chat History** (Dummy 1D vectors) - Filter-based access, encrypted content
  3. **Executed Operations** - History of operations executed
  4. **Immutable Reports** - Cost reports, audit logs, compliance reports
  5. **Alerts & Events** - Alert rules, scan results, auto-remediation settings
  6. **User Playbooks** (Real 384D vectors) - Custom playbooks with full scripts, encrypted
- **Point ID Strategy**: Random UUIDs for chat (immutable), deterministic IDs for estate (mutable, prevents duplicates)
- **Encryption**: AES-256-GCM with OS Keychain (macOS/Windows/Linux)
- **Backup**: Auto S3 sync with configurable retention (7 days local, 30 days S3)

### Client Module Architecture (Libraries)

**All libraries located in**: `docs/04-services/libraries/`

- **Storage Service**: Single Qdrant instance, 6 RAG collections, IAM integration, encryption
- **Execution Engine**: Pure Rust crate, Tokio + streaming, background execution
- **Estate Scanner**: Thin orchestrator, pluggable scanners (AWS/Azure/GCP), IAM discovery, semantic embeddings
- **Common Types**: Shared data structures (AWSResource, IAMPermissions, etc.), zero framework deps
- **Request Builder**: (To be designed) Context enrichment, server communication

### Client Frontend Architecture
- **MVC Pattern**: Models (Zustand stores), Views (React components), Controllers (business logic), Services (Tauri/WebSocket/API)
- **UI Components**: 30+ React presentation components for dynamic rendering
- **UI Rendering Engine**: Client-side orchestrator that maps server UI specifications to React components
- **Server-Driven UI**: Server-side UI Agent generates specifications, client-side Rendering Engine executes
- **Tauri Integration**: 70+ commands, 15+ events connecting React frontend to Rust backend
- **Authentication**: AWS Cognito with JWT tokens stored in OS Keychain

### Shared Services Architecture (docs/04-services/libraries/)

These are **Rust crates** (like npm packages) used by both client and server:

- **Storage Service** (storage-service crate):
  - Client uses: Embedded Qdrant for local cloud estate (AWS/Azure/GCP) + chat history + 6 RAG collections (encrypted)
  - Server uses: Embedded Qdrant for playbook metadata + S3 paths (not encrypted)
  - Key difference: Client stores full data, server stores metadata + S3 references
  - Both: Same API, same Rust crate, different Qdrant collections

- **Execution Engine** (execution-engine crate):
  - Client uses: Execute AWS/Azure/GCP CLI commands, bash scripts, Python scripts locally
  - Server uses: (Future) Execute validation scripts, test playbooks
  - Pure Rust crate with Tokio + streaming, no framework dependencies

- **Estate Scanner** (estate-scanner crate):
  - Client uses: Scan local cloud accounts (AWS/Azure/GCP) and populate Qdrant
  - Server uses: Not used (server has no cloud credentials)
  - Thin orchestrator over Execution Engine + Storage Service

- **Common Types** (cloudops-common crate):
  - Shared data structures: AWSResource, IAMPermissions, UserContext, etc.
  - Zero framework dependencies, just serde + chrono
  - Used by all other crates for type safety

**Key Insight**: Same Rust code, different deployment contexts. Client has full data + encryption, server has metadata + S3 references.

### Server Agent System

- **Master Agent** (Intent Classification & Parameter Extraction):
  - **CRITICAL**: This agent extracts parameters using LLM
  - **Input**: Raw user prompt + chat history + estate context from client
  - **Output**: Structured JSON with intent classification + **extracted_parameters**
  - **LLM-Powered**: Uses Claude/GPT to understand natural language and extract values
  - **Key Role**: The ONLY component that extracts parameters (neither client nor Playbook Agent does this)

- **UI Agent** (Server-Side Presentation Intelligence):
  - Transforms raw backend responses into structured UI specifications
  - Template vs Dynamic: 20% fast templates (<200ms) vs 80% LLM-powered dynamic (<1.5s)
  - 30+ Component Vocabulary for dynamic UI generation
  - Dual Outputs: UI mode (5 KB, full components) + history digest (300 bytes, 20x smaller)
  - Documentation: [docs/03-server/agents/presentation/ui-agent.md](docs/03-server/agents/presentation/ui-agent.md)

- **Playbook Agent**: LLM + RAG intelligent playbook search and recommendation
  - **4-Step Intelligence Flow**: LLM Intent Understanding → RAG Vector Search → LLM Ranking & Reasoning → Package & Return
  - **CRITICAL INPUT**: Receives structured JSON from Master Agent with **parameters already extracted**
  - **Does NOT Extract Parameters**: Master Agent does this - Playbook Agent just searches and ranks
  - **Normalized Storage**: Scripts and playbooks separated (database-style with foreign key references)
  - **Three Playbook Types**: Pure Script, Pure Orchestration, Hybrid
  - **Multi-Implementation Support**: Same script in bash/python/node/terraform/cloudformation
  - **10 Lifecycle States**: draft, ready, active, deprecated, archived, pending_review, approved, rejected, broken, needs_update
  - **Scripts Embedded in Response**: Client receives scripts in API payload, does NOT download from S3
  - **Documentation**: [docs/03-server/agents/intelligence/playbook-agent.md](docs/03-server/agents/intelligence/playbook-agent.md) (~4,637 lines)

- **Other Agents**: Classification, Operations, Validation, Risk Assessment (design phase)

## Common Tasks

### Adding New Architecture Documentation
1. Determine which section (01-overview, 02-client, 03-server, etc.)
2. Create markdown file with clear heading structure
3. Link from parent overview.md or INDEX.md
4. Update [docs/INDEX.md](docs/INDEX.md) if adding new section
5. Update [STRUCTURE.md](STRUCTURE.md) if adding new files
6. Validate architectural correctness (see checklist above)

### Documenting a New Component

**Client Components**:
- Add to [docs/02-client/](docs/02-client/)
- If it's a Rust crate, also document in [docs/04-services/libraries/](docs/04-services/libraries/)

**Server Agents**:
- Add to appropriate category in [docs/03-server/agents/](docs/03-server/agents/)
  - core/ - Request processing (master, classification)
  - intelligence/ - LLM + RAG (playbook, operations)
  - validation/ - Safety (validation, risk assessment)
  - presentation/ - UI generation (ui-agent)

**Server Services**:
- Add to [docs/04-services/services/](docs/04-services/services/)

**Important**: After adding documentation:
1. Update [docs/INDEX.md](docs/INDEX.md) with new section/component
2. Update [STRUCTURE.md](STRUCTURE.md) with file references
3. Verify agent responsibilities are correctly documented
4. Check all relative path references work

### Updating Path References

After the 04-services/ reorganization, when updating documentation:

**Find references** to old paths:
```bash
grep -r "04-services/storage-service" docs/
grep -r "04-services/execution-engine" docs/
grep -r "04-services/estate-scanner" docs/
grep -r "04-services/common" docs/
```

**Replace with** new paths:
- `04-services/storage-service/` → `04-services/libraries/storage-service/`
- `04-services/execution-engine/` → `04-services/libraries/execution-engine/`
- `04-services/estate-scanner/` → `04-services/libraries/estate-scanner/`
- `04-services/common/` → `04-services/libraries/common/`
- `04-services/playbook-service/` → `04-services/services/playbook-service/`

### Creating Diagrams
1. Create source file in [diagrams/source/](diagrams/source/)
2. Export to PNG/SVG in appropriate category (system/flows/components)
3. Reference diagram in markdown documentation
4. Commit both source and exported files

### Validating Documentation for Architectural Correctness

When reviewing or creating documentation, check for these common errors:

**Parameter Extraction**:
- ❌ "Client extracts parameters" → ✅ "Master Agent extracts parameters"
- ❌ "Client auto-fills parameters" → ✅ "Master Agent extracts, user manually fills placeholders"
- ❌ "auto_fill_strategy" → ✅ "extraction_hint"

**Script Delivery**:
- ❌ "Client downloads scripts from S3" → ✅ "Scripts embedded in API response"
- ❌ "Pre-signed URLs for scripts" → ✅ "Scripts in response payload"

**Agent Responsibilities**:
- ❌ "Playbook Agent extracts parameters" → ✅ "Playbook Agent receives extracted parameters from Master Agent"
- ❌ "Client has LLM for parameter extraction" → ✅ "Client has NO LLM, only displays and executes"

**Multi-Cloud**:
- ❌ "AWS estate" (unless AWS-specific) → ✅ "cloud estate (AWS/Azure/GCP)"

**Path References** (After October 2025 reorganization):
- ❌ `docs/04-services/storage-service/` → ✅ `docs/04-services/libraries/storage-service/`
- ❌ `docs/03-server/microservices/` → ✅ `docs/04-services/services/` (folder deleted)

## Cross-References

- Main architecture: [architecture.md](architecture.md)
- Product vision: [docs/01-overview/PRODUCT-VISION.md](docs/01-overview/PRODUCT-VISION.md)
- Project summary: [PROJECT-SUMMARY.md](PROJECT-SUMMARY.md) (998 lines)
- Product specification index: [docs/INDEX.md](docs/INDEX.md) (479 lines)
- Client overview: [docs/02-client/CLIENT-SUMMARY.md](docs/02-client/CLIENT-SUMMARY.md)
- Server agents overview: [docs/03-server/agents/](docs/03-server/agents/) (organized by category)
- Shared libraries: [docs/04-services/libraries/](docs/04-services/libraries/)
- Running services: [docs/04-services/services/](docs/04-services/services/)
- ADR index: [adr/](adr/)
- Contribution guidelines: [docs/meta/CONTRIBUTING.md](docs/meta/CONTRIBUTING.md)

## What NOT to Include

- Implementation details (those belong in component repositories)
- Code examples or API documentation
- Step-by-step tutorials or how-to guides
- Bug reports or feature requests
- Deployment-specific configurations (unless architectural)

## Notes for Claude Code

- This is a documentation-only repository - no code, no builds, no tests
- Focus on maintaining architectural clarity and coherence
- When asked to document a new component, first understand where it fits in the client vs server dichotomy
- The server side has significant depth - don't oversimplify it
- Always maintain the 10,000-foot perspective
- **CRITICAL**: Always verify architectural correctness, especially regarding parameter extraction flow
- When in doubt about who does what, refer to the "Critical Architecture Corrections" section above
- After the October 2025 reorganization, use correct paths: `04-services/libraries/` and `04-services/services/`
- When updating documentation, check [docs/meta/CONTRIBUTING.md](docs/meta/CONTRIBUTING.md) for standards