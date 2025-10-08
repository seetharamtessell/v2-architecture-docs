# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the **v2-architecture-docs** repository - the single source of truth for architectural documentation of the AWS CloudOps AI Agent system (V2). This is a **10,000-foot view documentation project**, not an implementation repository.

## Key Principles

1. **Architectural Purity**: Document the "what" and "why", not implementation details or code-level specifics
2. **High-Level Focus**: Maintain the big picture view across the entire system ecosystem
3. **Client-Server Separation**: The architecture has two distinct worlds:
   - **Client**: Tauri-based app with local AWS estate, Qdrant vector DB, and secure credential storage
   - **Server**: Multi-agent system with microservices for operations intelligence (NO AWS credentials/estate stored)

## Architecture Overview

The system architecture is based on these core principles:
- **Client-side AWS Estate Management**: All AWS resource data and credentials stay local
- **Server-side Operations Knowledge**: Stateless server provides playbook execution logic
- **Privacy First**: AWS credentials and estate data never sent to server
- **Semantic Search**: Local Qdrant vector DB enables fast, fuzzy resource lookup
- **Precision Over Guessing**: Client sends complete context; server generates exact scripts

## Directory Structure

```
docs/
├── 01-overview/          # System overview, key decisions, tech stack
├── 02-client/            # Client architecture (Tauri app)
│   ├── frontend/         # UI layer
│   ├── backend/          # Rust core components
│   ├── storage/          # Local Qdrant + AWS estate cache
│   └── sync/             # AWS data synchronization
├── 03-server/            # Server ecosystem (its own complex world)
│   ├── agents/           # Multi-agent system (classification, operations, validation, risk)
│   ├── microservices/    # Service-oriented architecture
│   ├── data/             # Redis, Qdrant (playbooks), Git repo
│   ├── infrastructure/   # Deployment, service mesh, scaling
│   └── integration/      # APIs and external integrations
├── 04-flows/             # Request, sync, execution flows
├── 05-security/          # Security and privacy architecture
├── 06-data/              # Data models, schemas, API contracts
└── 07-operations/        # Deployment, monitoring, DR

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
3. **Server**: Receive complete context → Classification agent → RAG search → Operations agent → Generate script
4. **Client**: Receive ready-to-execute script → Display + approval → Execute locally

### Data Flow
- **AWS estate sync**: Periodic sync (6h full, 15min incremental) from AWS APIs → Transform → Embed → Qdrant
- **Credentials**: Stored in OS Keychain, never leave client
- **Playbooks**: Stored in Git on server, indexed in server-side Qdrant

### Server Agent System
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

### Documenting a New Component
1. **Client components**: Add to [docs/02-client/](docs/02-client/)
2. **Server agents**: Add to [docs/03-server/agents/](docs/03-server/agents/)
3. **Server microservices**: Add to [docs/03-server/microservices/](docs/03-server/microservices/)
4. Update integration/flow documentation if it affects request flow

### Creating Diagrams
1. Create source file in [diagrams/source/](diagrams/source/)
2. Export to PNG/SVG in appropriate category (system/flows/components)
3. Reference diagram in markdown documentation
4. Commit both source and exported files

## Cross-References

- Main architecture: [architecture.md](architecture.md)
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