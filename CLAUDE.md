# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the **v2-architecture-docs** repository - the single source of truth for architectural documentation of the Escher Multi-Cloud Operations Platform (V2). This is a **10,000-foot view documentation project**, not an implementation repository.

## Key Principles

1. **Architectural Purity**: Document the "what" and "why", not implementation details or code-level specifics
2. **High-Level Focus**: Maintain the big picture view across the entire system ecosystem
3. **Client-Server Separation**: The architecture has two distinct worlds:
   - **Client**: Tauri-based app with local cloud estate (AWS/Azure/GCP), Qdrant vector DB, and secure credential storage
   - **Server**: Multi-agent system with microservices for operations intelligence (NO cloud credentials/estate stored)

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
├── 01-overview/          # System overview, key decisions, tech stack
├── 02-client/            # Client architecture (Tauri app)
│   ├── overview.md       # Client architecture overview
│   ├── CLIENT-SUMMARY.md # Complete architecture summary with all UI
│   ├── COMPLETION-STATUS.md # Progress tracking
│   ├── modules/          # Client-specific modules
│   │   ├── overview.md
│   │   └── request-builder/      🔄 Next (0% - to be designed)
│   ├── frontend/         # Frontend architecture (React + TypeScript)
│   │   ├── README.md
│   │   ├── mvc-architecture.md   ✅ Complete (MVC pattern)
│   │   ├── user-flows.md         ✅ Complete (8 flows)
│   │   ├── ui-components.md ✅ Complete (30+ presentation components)
│   │   ├── ui-rendering-engine.md ✅ Complete (client-side orchestrator)
│   │   └── authentication-security.md ✅ Complete (Cognito + JWT)
│   ├── tauri-integration/ # IPC Bridge (Frontend ↔ Rust)
│   │   ├── README.md
│   │   ├── commands-storage.md   ✅ Complete (25+ commands)
│   │   ├── commands-execution.md ✅ Complete (15+ commands)
│   │   ├── commands-estate-scanner.md ✅ Complete (15+ commands)
│   │   ├── commands-auth.md      ✅ Complete (15+ commands)
│   │   ├── events-scan.md        ✅ Complete (6 events)
│   │   ├── events-execution.md   ✅ Complete (6 events)
│   │   └── events-system.md      ✅ Complete (8 events)
│   └── ui-team-implementation/ ✅ NEW! Complete parallel dev guide (~22,700 lines, 6 files)
│       ├── README.md             # Quick start + overview
│       ├── 01-architecture.md    # MVC pattern, data flow (~4,800 lines)
│       ├── 02-implementation-plan.md # 7 phases with examples (~5,200 lines)
│       ├── 03-project-structure.md # Complete folder layout (~4,100 lines)
│       ├── 04-mock-contracts.md  # TypeScript interfaces (~4,900 lines)
│       └── 05-claude-prompts.md  # 25 ready-to-use prompts (~3,700 lines)
├── 03-server/            # Server ecosystem (its own complex world)
│   ├── agents/           # Multi-agent system
│   │   ├── overview.md
│   │   ├── ui-agent.md           ✅ Complete (comprehensive) - Server-side UI intelligence
│   │   └── playbook-agent.md     ✅ Complete (~4,637 lines) - LLM + RAG intelligence
│   ├── microservices/    # Service-oriented architecture
│   ├── data/             # Redis, Qdrant (playbooks), Git repo
│   ├── infrastructure/   # Deployment, service mesh, scaling
│   └── integration/      # APIs and external integrations
├── 04-services/          # Shared services used by both client and server
│   ├── storage-service/          ✅ Complete (~4,000 lines, 10 files)
│   │   ├── architecture.md       # Overall design, components
│   │   ├── api.md                # Complete Rust API reference
│   │   ├── collections.md        # Qdrant schemas + IAM
│   │   ├── configuration.md      # Config structure
│   │   ├── encryption.md         # AES-256-GCM + Keychain
│   │   ├── backup-restore.md     # S3 backup workflows
│   │   ├── point-management.md   # ID strategies
│   │   ├── initialization.md     # Setup and bootstrap
│   │   ├── operations.md         # Code examples
│   │   └── testing.md            # Testing strategies
│   ├── execution-engine/         ✅ Complete (~6,000 lines, 9 files)
│   ├── estate-scanner/           ✅ Complete (~3,000 lines, 4 files)
│   ├── common/                   ✅ Complete (~650 lines, 1 file)
│   └── playbook-service/         ✅ Complete (client-side playbook management)
├── 05-flows/             # Request, sync, execution flows
├── 06-security/          # Security and privacy architecture
├── 07-data/              # Data models, schemas, API contracts
└── 08-operations/        # Deployment, monitoring, DR

working-docs/             # Active design documents
├── CLIENT-DESIGN-WORKING-DOC-V2.md   # Storage Service design
├── CLIENT-MODULE-ARCHITECTURE.md     # Module overview
├── MODULE-INTERACTION-ANALYSIS.md    # How modules communicate
├── COMMON-TYPES-ANALYSIS.md          # Shared type definitions
├── SESSION-SUMMARY.md                # Development session notes
└── DOCS-STRUCTURE.md                 # Documentation plan

reference/                # Reference implementations
├── CONTEXT-MANAGEMENT-ARCHITECTURE.md # Node.js reference
└── COMPARISON-NODE-VS-RUST-STORAGE.md # Design comparison

diagrams/
├── system/               # Overall, client, server architecture diagrams
├── flows/                # Sequence and flow diagrams
├── components/           # Service topology and integration maps
└── source/               # Editable diagram sources (draw.io, mermaid, etc.)

adr/                      # Architecture Decision Records
repositories/             # Catalog of all related repositories
```

## Documentation Guidelines

### When Adding Documentation

1. **Focus on Architecture**: Describe component interactions, data flows, and system-level decisions
2. **Avoid Implementation Details**: Don't document code structure, function signatures, or line-by-line logic
3. **Server Complexity**: The server side is a complex ecosystem with multiple agents and microservices - give it proper depth
4. **Maintain Separation**: Keep client and server concerns clearly separated
5. **Update Diagrams**: When architecture changes, update both diagrams and their source files
6. **Validate Architectural Correctness**: Always verify who does what (especially parameter extraction!)

### Document Quality Checklist

Before committing documentation changes, verify:
- [ ] Client responsibilities clearly separated from server responsibilities
- [ ] No client-side parameter extraction references (only Master Agent extracts)
- [ ] Correct terminology: "extraction_hint" not "auto_fill_strategy"
- [ ] Multi-cloud context: "cloud estate" not "AWS estate" (unless AWS-specific)
- [ ] Scripts embedded in response, not downloaded by client
- [ ] Consistent with [PROJECT-SUMMARY.md](PROJECT-SUMMARY.md) architecture

### When to Create ADRs

Create an Architecture Decision Record (ADR) for:
- Technology stack choices (e.g., Tauri over Electron)
- Architectural patterns (e.g., stateless server, local vector DB)
- Security models
- Data storage strategies
- Integration approaches

Use the [adr/template.md](adr/template.md) for consistency.

### Markdown Conventions

- Use relative links between documentation files
- Link to specific line numbers in [architecture.md](architecture.md) when referencing details
- Include ASCII diagrams for flows and component interactions
- Keep documentation hierarchical: overview → detail

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

### Client Module Architecture
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

### UI Team Implementation Guide (Parallel Development Enabler)
- **Purpose**: Enables UI team to build 100% independently without waiting for Platform or Server teams
- **Approach**: Build with mocks first (MockTauriService, MockWebSocketService, MockAPIService), integrate later
- **Contents**:
  - Complete MVC architecture with data flow examples
  - 7-phase implementation plan with code examples
  - Complete project structure (folder layout, naming conventions)
  - Mock contracts for all 70+ Tauri commands and WebSocket/HTTP APIs
  - 25 ready-to-use Claude Code prompts for rapid development
- **Strategy**: UI team builds full functional application with mock data, then swaps mocks for real implementations (zero controller changes needed)

### Shared Services Architecture (docs/04-services)

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

- **Playbook Service** (playbook-service crate):
  - Client uses: Manage local playbooks with full scripts (encrypted in Qdrant)
  - Server uses: Not used (server has Playbook Agent instead)
  - Storage strategy: local_only, uploaded_for_review, uploaded_trusted, using_default

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
  - **Purpose**: Server-side intelligence that decides optimal UI presentation
  - **Input**: Raw responses from other agents (Cost Agent, Analysis Agent, etc.)
  - **Output**: Structured JSON with UI markers + dual outputs (UI mode + history digest)
  - **Template vs Dynamic**: 20% fast templates (<200ms) vs 80% LLM-powered dynamic (<1.5s)
  - **30+ Component Vocabulary**: Complete registry for dynamic UI generation
  - **Dual Outputs**: UI mode (5 KB, full components) + history digest (300 bytes, 20x smaller for LLM context)
  - **Key Benefit**: Makes frontend infinitely scalable - add visualizations without backend changes
  - **Documentation**: [docs/03-server/agents/ui-agent.md](docs/03-server/agents/ui-agent.md)

- **Playbook Agent**: LLM + RAG intelligent playbook search and recommendation
  - **4-Step Intelligence Flow**: LLM Intent Understanding → RAG Vector Search → LLM Ranking & Reasoning → Package & Return
  - **CRITICAL INPUT**: Receives structured JSON from Master Agent with **parameters already extracted**
  - **Does NOT Extract Parameters**: Master Agent does this - Playbook Agent just searches and ranks
  - **Normalized Storage Architecture**: Scripts and playbooks separated (database-style with foreign key references)
  - **Three Playbook Types**: Pure Script (only script_refs), Pure Orchestration (only playbook_refs), Hybrid (mix of both)
  - **Multi-Implementation Support**: Same script in bash/python/node/terraform/cloudformation - user chooses at runtime
  - **orchestration.json Format**: Step sequences with external references (script_id + version, playbook_id + version)
  - **Server-Side RAG**: Embedded Qdrant with `escher_library` (global) and `tenant_{id}_playbooks` (per-tenant) collections
  - **LLM Integration**: Uses Claude/GPT for intent parsing and intelligent ranking with explanations
  - **Playbook Lifecycle**: 10 status values (draft, ready, active, deprecated, archived, pending_review, approved, rejected, broken, needs_update)
  - **Storage Strategy**: Metadata + S3 paths in RAG, full scripts in S3 (escher-library and escher-tenant-data buckets)
  - **S3 Structure**: Two folders - scripts/ (reusable library) and playbooks/ (orchestration files)
  - **Scripts Embedded in Response**: Client receives scripts in API payload, does NOT download from S3
  - **Parameter Handling**: Merges extracted_parameters from Master Agent with playbook parameter definitions
  - **Response**: Returns playbook with pre-filled parameters (from Master Agent) + placeholders (user must fill)
  - **Advanced Features**:
    - Nested sub-steps (1.1, 1.2, 1.3) with blocking behavior
    - Auto-remediation with LLM-powered error analysis (70-90% transient failures auto-fixed)
    - Resume capability for mid-execution failures
  - **Review Workflow**: User uploads → pending_review → approved/rejected → active (with state transitions)
  - **What's in RAG**: Metadata (name, description, keywords, status, execution stats) + S3 script paths (NOT full scripts)
  - **Documentation**: [docs/03-server/agents/playbook-agent.md](docs/03-server/agents/playbook-agent.md) (~4,637 lines, comprehensive)

- **Classification Agent**: Intent recognition and routing
- **Operations Agent**: Script generation from playbooks
- **Validation Agent**: Feasibility and safety checks
- **Risk Assessment Agent**: Risk scoring and approval requirements

## Common Tasks

### Adding New Architecture Documentation
1. Determine which section (01-overview, 02-client, 03-server, etc.)
2. Create markdown file with clear heading structure
3. Link from parent overview.md
4. Update relevant diagrams if needed
5. Validate architectural correctness (see checklist above)

### Documenting a New Component
1. **Client components**: Add to [docs/02-client/](docs/02-client/)
2. **Server agents**: Add to [docs/03-server/agents/](docs/03-server/agents/)
3. **Server microservices**: Add to [docs/03-server/microservices/](docs/03-server/microservices/)
4. Update integration/flow documentation if it affects request flow
5. Verify agent responsibilities are correctly documented (especially parameter extraction)

### Documenting Module Interactions
1. **Identify dependencies**: Which modules call which (e.g., Estate Scanner → Execution Engine + Storage Service)
2. **Document data contracts**: What types are passed between modules (use Common Types)
3. **Update MODULE-INTERACTION-ANALYSIS.md**: Keep the interaction map current
4. **Show async boundaries**: Clarify where async operations happen

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

## Cross-References

- Main architecture: [architecture.md](architecture.md)
- Product vision: [docs/01-overview/PRODUCT-VISION.md](docs/01-overview/PRODUCT-VISION.md)
- Project summary: [PROJECT-SUMMARY.md](PROJECT-SUMMARY.md)
- Client overview: [docs/02-client/overview.md](docs/02-client/overview.md)
- Server overview: [docs/03-server/overview.md](docs/03-server/overview.md)
- ADR index: [adr/README.md](adr/README.md)
- Repository catalog: [repositories/repos.md](repositories/repos.md)

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