# Escher V2 - Product & Architecture Specification

**Product**: Escher Multi-Cloud Operations Platform
**Purpose**: Single source of truth for system architecture
**Last Updated**: October 2025
**Status**: 90% Architecture Complete

---

## 👋 Welcome

**New here?** Start with [Product Vision](./01-overview/PRODUCT-VISION.md) for a 5-minute overview of what we're building and why.

**Need complete reference?** See [PROJECT-SUMMARY.md](../PROJECT-SUMMARY.md) (998 lines) - comprehensive project overview.

**AI Assistant?** Check [CLAUDE.md](../CLAUDE.md) for development guidance and coding standards.

---

## 🏗️ System Architecture

### Architecture Principles

**Escher is built on these fundamental principles:**

1. **Client-Server Separation** (Most Critical)
   - **No credentials leave client** - Ever. AWS keys, Azure service principals, GCP service accounts stay local
   - **Execution happens on client** - All cloud operations run locally on user's machine/VM
   - **Server is the brain** - Provides intelligence, playbooks, recommendations (but has NO memory)
   - **Client is state + executor** - Owns all data, executes all operations

2. **Privacy-First Design**
   - Client owns ALL data (estate, credentials, chat history)
   - Server is 100% stateless - stores NOTHING about clients
   - Zero trust model - you control everything
   - Local encryption: AES-256-GCM for all data at rest

3. **Multi-Cloud Architecture**
   - Unified experience for AWS, Azure, and GCP
   - Cloud-agnostic resource types and operations
   - Consistent UX across all cloud providers
   - Cross-cloud optimization recommendations

4. **Precision Over Guessing**
   - Client sends complete context to server (but NO credentials/secrets)
   - Server generates exact scripts, not guesses
   - User approval required before execution
   - Full audit trail of all operations

### High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│  USER'S ENVIRONMENT (100% Private)                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Tauri Desktop App                                   │   │
│  │  ├─ React Frontend (Multi-cloud UI)                  │   │
│  │  ├─ Rust Backend (Execution Engine)                  │   │
│  │  ├─ Local Qdrant (6 RAG Collections, ~20-30 MB)      │   │
│  │  └─ OS Keychain (AWS/Azure/GCP Credentials)          │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↕  HTTPS
┌─────────────────────────────────────────────────────────────┐
│  ESCHER SERVER (100% Stateless, Multi-Tenant)               │
│  ├─ AI Agents (Classification, Playbook, Operations, etc.)  │
│  ├─ Playbook Library (Global + Tenant-Specific)             │
│  ├─ UI Intelligence (Templates + LLM Dynamic)               │
│  └─ Qdrant (Playbook Metadata Only, NO Client Data)         │
└─────────────────────────────────────────────────────────────┘
```

**Request Flow**:
```
User Query
   ↓
Client: Semantic Search (Local Qdrant) → Identify Resources
   ↓
Client: Build Enriched Context (NO credentials, just summaries)
   ↓
Server: Master Agent → Extract Parameters
   ↓
Server: Playbook Agent → LLM+RAG Search → Generate Script
   ↓
Client: Display to User → User Approval → Local Execution
```

---

## 📖 Product Specifications

### 01-overview/ - Product Vision & Architecture

**Entry Point**: [PRODUCT-VISION.md](./01-overview/PRODUCT-VISION.md)

**What's Here**:

| Document | Purpose | Status |
|----------|---------|--------|
| [PRODUCT-VISION.md](./01-overview/PRODUCT-VISION.md) | Complete product vision, deployment models, target users | ✅ Complete |
| [architecture.md](./01-overview/architecture.md) | Multi-cloud architecture overview | ✅ Complete |
| [system-overview.md](./01-overview/system-overview.md) | Non-technical overview for stakeholders | ✅ Complete |
| [technology-stack.md](./01-overview/technology-stack.md) | Technology decisions and rationale | ✅ Complete |
| [key-decisions.md](./01-overview/key-decisions.md) | Architecture Decision Records (ADRs) | ✅ Complete |

**Key Topics**:
- Two deployment models: "Run on Your Laptop" (local-only) vs "Extend to Your Cloud" (24/7 automation)
- Target users: DevOps engineers, SREs, platform engineering teams
- Multi-cloud support strategy (AWS/Azure/GCP)
- Privacy-first architecture and zero-trust model

---

### 02-client/ - Client Application Specification

**Entry Point**: [02-client/INDEX.md](./02-client/INDEX.md)

**What is the Client?**

The client is a **Tauri desktop application** that:
- Runs on user's laptop or their cloud VM
- Owns ALL data: estate, credentials, chat history
- Executes cloud operations locally (AWS/Azure/GCP CLIs)
- Never sends credentials to server (100% private)
- Uses local Qdrant vector DB for semantic search

**Key Components**:

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Frontend | React + TypeScript | MVC architecture, 30+ UI components, server-driven rendering |
| Rust Backend | Tauri + Tokio | Execution engine, estate scanner, storage service |
| Local Vector DB | Qdrant (embedded) | 6 RAG collections for semantic search + filtering |
| IPC Bridge | Tauri | 70+ commands, 15+ events (frontend ↔ Rust) |
| Credentials | OS Keychain | Secure storage for AWS/Azure/GCP credentials |

**Documentation Sections**:
- `architecture/` - Client architecture overview and design principles
- `frontend/` - React MVC architecture, UI components, user flows
- `modules/` - Rust backend modules (navigation to 04-services/)
- `tauri-integration/` - IPC bridge specification (commands + events)
- `ui-team-implementation/` - Parallel development guide for UI team
- `meta/` - Documentation status and completion tracking

**Status**: ✅ 90% Complete (~25,000+ lines documented)

---

### 03-server/ - Server System Specification

**Entry Point**: [03-server/overview.md](./03-server/overview.md)

**What is the Server?**

The server is a **100% stateless AI system** that:
- Stores NOTHING about clients (no credentials, no estate data, no chat history)
- Provides global cloud operations knowledge (playbooks, best practices)
- Processes requests transiently and forgets everything
- Multi-tenant by design (tenant isolation via API keys)
- Built with Go for performance and simplicity

**Key Components**:

| Component | Purpose | Status |
|-----------|---------|--------|
| **AI Agents** | 5 specialized agents | 🔄 40% Complete |
| - Master Agent | Parameter extraction, intent recognition | 📋 TBD |
| - [Playbook Agent](./03-server/agents/playbook-agent.md) | LLM + RAG intelligence, 4-step flow | ✅ Complete |
| - [UI Agent](./03-server/agents/ui-agent.md) | Presentation intelligence (templates + LLM) | ✅ Complete |
| - Operations Agent | Script generation from playbooks | 📋 TBD |
| - Validation Agent | Feasibility and safety checks | 📋 TBD |
| **Playbook Library** | Global + tenant-specific playbooks | ✅ Complete |
| **API Layer** | HTTP REST + WebSocket streaming | 📋 TBD |

**Key Specifications**:
- **Playbook Agent**: 4-step flow (LLM Intent → RAG Search → LLM Ranking → Package), 10 lifecycle states, auto-remediation
- **UI Agent**: Template-based (20%, <200ms) + LLM dynamic (80%, <1.5s), streaming architecture
- **Stateless Design**: Server processes requests, returns responses, forgets everything

**Status**: 🔄 40% Complete (Playbook + UI agents done, 3 agents in progress)

---

### 04-services/ - Shared Services Specification

**Entry Point**: [04-services/](./04-services/)

**What are Shared Services?**

Reusable **Rust crates** used by both client and server:
- Pure libraries with no framework dependencies
- Type-safe, trait-based architecture
- Well-tested with 80%+ coverage
- Production-ready and battle-tested

**Services**:

| Service | Purpose | Status | Lines |
|---------|---------|--------|-------|
| [storage-service](./04-services/storage-service/) | Qdrant + RAG (6 collections), encryption (AES-256-GCM) | ✅ Complete | ~4,000 |
| [execution-engine](./04-services/execution-engine/) | Async command execution, streaming output, background tasks | ✅ Complete | ~6,000 |
| [estate-scanner](./04-services/estate-scanner/) | Multi-cloud resource discovery (AWS/Azure/GCP) | ✅ Complete | ~3,000 |
| [common](./04-services/common/) | Shared data structures (cloud-agnostic types) | ✅ Complete | ~650 |
| [playbook-service](./04-services/playbook-service/) | Playbook management and lifecycle | ✅ Complete | TBD |

**Key Specifications**:

**Storage Service** - 6 RAG Collections:
1. **Cloud Estate Inventory** - Real 384D vectors (semantic search) + IAM permissions per resource
2. **Chat History** - Dummy 1D vectors (filter-based access only)
3. **Executed Operations** - Operation history and audit trail
4. **Immutable Reports** - Cost reports, audit logs, compliance (daily sync at 2am)
5. **Alerts & Events** - Alert rules, scan results, auto-remediation settings
6. **User Playbooks** - Real 384D vectors (user-created playbooks with full scripts)

**Execution Engine** - Async, non-blocking execution:
- Background execution (returns execution_id immediately)
- Serial, parallel, and dependency-based execution strategies
- Streaming output (real-time stdout/stderr)
- Pluggable event handlers (Tauri, WebSocket, logging)

**Estate Scanner** - 4-level parallelism:
- Level 1: Multiple accounts/subscriptions/projects in parallel
- Level 2: Multiple regions per account in parallel
- Level 3: Multiple services per region in parallel
- Level 4: Multiple resources per service in parallel
- Pluggable trait-based scanners for AWS/Azure/GCP

**Status**: ✅ 95% Complete (~13,650+ lines documented)

---

### 05-flows/ - System Flows & Integration

**Entry Point**: [05-flows/README.md](./05-flows/README.md)

**Key Flows**:
- User query → Server AI → Client execution
- Estate scanning and synchronization (incremental + full)
- Playbook execution lifecycle (explain → execute → monitor)
- Auto-remediation flows (detect → validate → auto-fix)
- Real-time streaming (text first → enhanced UI later)

**Status**: 📋 TBD (placeholder README exists)

---

### 06-security/ - Security & Privacy Model

**Entry Point**: [06-security/README.md](./06-security/README.md)

**Core Principles**:
- Client owns ALL data (estate, credentials, chat history)
- Server is 100% stateless (stores NOTHING about clients)
- Zero trust model - verify everything
- AES-256-GCM encryption for all local data
- Cloud credentials in OS Keychain (never transmitted over network)

**Key Topics**:
- Privacy-first architecture
- Data ownership and encryption model
- Credential management (AWS/Azure/GCP)
- Network security (TLS, API authentication)
- Audit logging and compliance

**Status**: 📋 TBD (placeholder README exists)

---

### 07-data/ - Data Architecture & Models

**Entry Point**: [07-data/README.md](./07-data/README.md)

**Key Topics**:
- 6 RAG collections strategy (client-side Qdrant)
- Vector embedding strategy (384D vs 1D, why each)
- IAM permissions embedded per resource
- Cloud-agnostic resource types (AWS/Azure/GCP)
- API contracts and schemas (client ↔ server)
- Point ID strategies (deterministic for estate, random for chat)

**Status**: 📋 TBD (placeholder README exists)

---

### 08-operations/ - Deployment & Operations

**Entry Point**: [08-operations/README.md](./08-operations/README.md)

**Deployment Models**:

**1. Run on Your Laptop** (Local-only)
- Desktop app only, no server required for basic operations
- No cloud compute costs (runs entirely on user's machine)
- Privacy-first: No data leaves your device
- Perfect for: Ad-hoc operations, development, testing

**2. Extend to Your Cloud** (24/7 Automation)
- Client runs in user's cloud VM (AWS EC2, Azure VM, GCP Compute)
- Server provides AI intelligence (multi-tenant SaaS or self-hosted)
- 24/7 operations: Scheduled jobs, monitoring, auto-remediation
- Real-time alerts and daily morning reports
- Perfect for: Production ops, continuous monitoring

**Key Topics**:
- Deployment architecture for both models
- Monitoring and observability
- Backup and disaster recovery
- Update mechanism (client + server)
- Scaling considerations

**Status**: 📋 TBD (placeholder README exists)

---

## 🔑 Key Architecture Concepts

### 1. Client-Server Separation

**The Most Critical Principle**:

```
CLIENT (User's Environment)               SERVER (Stateless AI)
├─ Owns ALL data                          ├─ Stores NOTHING
├─ Cloud credentials (NEVER sent)         ├─ Provides intelligence
├─ Cloud estate data (local only)         ├─ Global playbook library
├─ Chat history (encrypted locally)       ├─ LLM + RAG processing
├─ Executes ALL operations (locally)      └─ Forgets after response
└─ State + Executor
```

**Rules**:
- ❌ NO credentials ever leave the client
- ❌ NO estate data sent to server (only minimal summaries)
- ✅ ALL cloud operations execute on client
- ✅ Server provides intelligence, client executes

---

### 2. Privacy-First Design

```
CLIENT                                    SERVER
├─ Cloud estate (AWS/Azure/GCP)           ├─ Playbook library
├─ Credentials (OS Keychain)              ├─ Best practices
├─ Chat history (AES-256-GCM)             ├─ Global knowledge
├─ Operations history                     └─ Stateless processing
└─ Encrypted with AES-256-GCM
```

**Data Ownership**:
- Client owns: Estate data, credentials, chat history, operation logs
- Server stores: NOTHING about clients (zero data retention)
- Encryption: AES-256-GCM for all local data at rest
- Transmission: Minimal context only (no secrets)

---

### 3. Multi-Cloud Architecture (AWS/Azure/GCP)

**Cloud-Agnostic Design**:
- Resource types work across all clouds
- Unified UX regardless of provider
- Cross-cloud optimization recommendations
- Pluggable provider implementations

**Terminology**:
- ✅ "cloud estate (AWS/Azure/GCP)"
- ✅ "cloud resources"
- ✅ "cloud CLI (aws/az/gcloud)"
- ❌ NOT "AWS estate" (too specific)

---

### 4. 6 RAG Collections (Client-Side Qdrant)

**Local Vector Database** (~20-30 MB embedded):

| Collection | Vectors | Purpose |
|-----------|---------|---------|
| 1. Cloud Estate Inventory | 384D (real) | Semantic search + IAM permissions per resource |
| 2. Chat History | 1D (dummy) | Filter-based access (no semantic search needed) |
| 3. Executed Operations | 1D (dummy) | Operation history and audit trail |
| 4. Immutable Reports | 1D (dummy) | Cost, audit, compliance reports (daily sync 2am) |
| 5. Alerts & Events | 1D (dummy) | Alert rules, scan results, auto-remediation |
| 6. User Playbooks | 384D (real) | User-created playbooks with full scripts |

**Strategy**:
- **Real 384D vectors** (#1, #6): Need semantic search (fuzzy matching)
- **Dummy 1D vectors** (#2-5): Only need filtering (exact match)

---

### 5. Server-Driven UI

**Dynamic Component Rendering**:

```
Server Response
   ↓
Text (0ms, immediate) ─────→ Client renders text instantly
   ↓
Enhanced UI (500ms-2s) ────→ Client renders structured components
```

**30+ Component Vocabulary**:
- Text blocks, code blocks, tables, charts
- Playbook cards (with explain/execute buttons)
- Resource lists, cost breakdowns
- Alert widgets, progress indicators

**Dual Outputs**:
- **UI Mode**: Full 5KB structured response (for rendering)
- **History Mode**: 300-byte digest (for chat history, 20x smaller)

**Template + LLM Hybrid**:
- Templates: 20% common cases (<200ms response)
- LLM Dynamic: 80% novel queries (<1.5s response)

---

### 6. Playbook Intelligence (4-Step Flow)

**Server-Side LLM + RAG**:

```
Step 1: LLM Intent Recognition
   User query: "Stop all dev EC2 instances in us-east-1"
   LLM extracts: {operation: "stop", service: "ec2", environment: "dev", region: "us-east-1"}
   ↓
Step 2: RAG Vector Search
   Query Qdrant with extracted parameters
   Returns: Top 20 candidate playbooks
   ↓
Step 3: LLM Ranking (with reasoning)
   LLM ranks playbooks by relevance
   Explains: "Playbook A is best because it handles bulk EC2 stop operations"
   ↓
Step 4: Package & Return
   Returns: Top 3 playbooks with metadata
   Client displays to user for approval
```

**10 Lifecycle States**:
- Draft → Ready → Active → Deprecated → Archived → Deleted
- Review → Testing → Failed → Suspended

**Auto-Remediation**:
- 70-90% of transient failures can be auto-fixed
- Pre-approved operations run automatically
- User notified after completion

---

## 📊 Documentation Status

### ✅ Complete (90%)
- **Product Vision**: 3 files, ~1,500 lines
- **Client Specification**: 40+ files, ~25,000 lines
- **Shared Services**: 28 files, ~13,650 lines
- **Server**: Playbook Agent + UI Agent (~6,800 lines)
- **PROJECT-SUMMARY**: 998 lines comprehensive overview

**Total**: 60+ files, 48,000+ lines documented

### 🔄 In Progress
- **Server**: 3 remaining agents (Master, Operations, Validation)
- **Client**: Request Builder module specification

### 📋 Planned (TBD)
- **05-flows/**: System flows and integration
- **06-security/**: Security and privacy architecture
- **07-data/**: Data models and schemas
- **08-operations/**: Deployment and operations

---

## 🤖 For AI & Code Generators

**Using Claude Code, Cursor, GitHub Copilot, or other AI assistants?**

We provide **module-specific AI implementation prompts** that contain everything needed to generate production-ready code:
- Complete project context
- Architecture principles and constraints
- Module-specific code standards
- Testing requirements (unit + integration)
- Security checklist
- Documentation requirements
- Complete examples

### Available AI Prompts

📂 **Location**: [/working-docs/ai-prompts/](../working-docs/ai-prompts/)

**Planned Prompts** (Coming Soon):

| Prompt | For | Description |
|--------|-----|-------------|
| 📋 AI-PROMPT-FRONTEND.md | Frontend Developers | React + TypeScript, MVC architecture, Zustand, UI components |
| 📋 AI-PROMPT-BACKEND-RUST.md | Rust Backend Developers | Tauri integration, IPC commands, async execution |
| 📋 AI-PROMPT-SERVER-GO.md | Server Go Developers | Stateless AI agents, LLM integration, playbook intelligence |
| 📋 AI-PROMPT-STORAGE-SERVICE.md | Storage Developers | Qdrant, 6 RAG collections, encryption (AES-256-GCM) |
| 📋 AI-PROMPT-EXECUTION-ENGINE.md | Execution Developers | Command execution, streaming output, Tokio runtime |
| 📋 AI-PROMPT-ESTATE-SCANNER.md | Scanner Developers | Multi-cloud discovery, trait-based scanners, parallelism |

**Status**: AI prompts are in development. Check back soon or contribute by creating them!

### How to Use AI Prompts

1. **Navigate to** `/working-docs/ai-prompts/`
2. **Find the prompt** for your module (e.g., `AI-PROMPT-FRONTEND.md`)
3. **Copy the entire prompt** (from "BEGIN PROMPT" to "END PROMPT")
4. **Paste into your AI code generator** (Claude Code, Cursor, etc.)
5. **Add your specific request** (e.g., "Implement the Storage Service")
6. **AI generates** production-ready code with tests, security, and documentation

**Each prompt includes**:
- Project overview and architecture principles
- Module-specific context and requirements
- Code standards for that language/framework
- Testing requirements (unit + integration tests)
- Security checklist and best practices
- Documentation requirements
- Complete working examples
- Pre-implementation checklist

### Before Using AI Code Generators

**Critical Constraints to Understand**:

1. **Client-Server Separation**: No credentials leave client, execution happens locally
2. **Multi-Cloud**: Always use "cloud (AWS/Azure/GCP)", never AWS-only terminology
3. **6 RAG Collections**: NOT "dual collection" (that's wrong - it's 6 collections)
4. **Privacy-First**: Server stores NOTHING about clients (100% stateless)
5. **Parameter Extraction**: Master Agent (server LLM) extracts parameters, NOT client

**For more guidance**, see [CLAUDE.md](../CLAUDE.md) for development standards.

---

## 👩‍💻 For Developers

### Frontend Developers (React + TypeScript)
→ **[Client Frontend Documentation](./02-client/INDEX.md)**

**What's there**:
- MVC architecture with Zustand state management
- 30+ UI components vocabulary
- Server-driven UI rendering engine
- 8 complete user flows documented
- TypeScript strict mode standards

### Backend Developers (Rust)
→ **[Client Modules Overview](./02-client/modules/overview.md)**
→ **[Shared Services](./04-services/)**

**What's there**:
- Tauri integration (70+ commands, 15+ events)
- Storage Service (Qdrant + RAG)
- Execution Engine (Tokio + streaming)
- Estate Scanner (multi-cloud discovery)
- Common types (cloud-agnostic)

### Server Developers (Go)
→ **[Server Architecture](./03-server/overview.md)**

**What's there**:
- 5 AI agents specification
- Playbook intelligence (LLM + RAG)
- Stateless design patterns
- API contracts (REST + WebSocket)

### Integration Engineers (Client ↔ Server)
→ **[Tauri IPC Bridge](./02-client/tauri-integration/README.md)**
→ **[Data Contracts](./07-data/README.md)**

**What's there**:
- 70+ Tauri commands specification
- 15+ Tauri events specification
- API contracts and schemas
- WebSocket streaming protocol

### Development Setup & Standards
→ **[CLAUDE.md](../CLAUDE.md)** - Coding standards, development guidance, common pitfalls

---

## 📚 Quick Navigation

### By Goal

**I want to understand the product** → [PRODUCT-VISION.md](./01-overview/PRODUCT-VISION.md)

**I want to understand the architecture** → [architecture.md](./01-overview/architecture.md) + This INDEX

**I want complete reference** → [PROJECT-SUMMARY.md](../PROJECT-SUMMARY.md) (998 lines)

**I want to implement frontend** → [02-client/INDEX.md](./02-client/INDEX.md)

**I want to implement backend** → [04-services/](./04-services/)

**I want to implement server** → [03-server/overview.md](./03-server/overview.md)

**I want to use AI code generators** → [AI Prompts](#-for-ai--code-generators) (coming soon)

---

## 📞 Need Help?

- **General Questions**: Start with [PROJECT-SUMMARY.md](../PROJECT-SUMMARY.md)
- **Development Setup**: See [CLAUDE.md](../CLAUDE.md)
- **Documentation Organization**: Check [STRUCTURE.md](../STRUCTURE.md)
- **Recent Changes**: Review [working-docs/](../working-docs/)
- **GitHub Issues**: Report issues at repository issue tracker

---

**Last Updated**: October 2025
**Documentation Coverage**: 60+ files, 48,000+ lines
**Status**: 90% Complete, Ready for Development