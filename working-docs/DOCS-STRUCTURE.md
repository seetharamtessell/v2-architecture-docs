# Documentation Structure

## Overview

This repository contains architecture documentation organized by concern.

## Directory Structure

```
v2-architecture-docs/
├── README.md                          # Repository purpose and guidelines
├── CLAUDE.md                          # AI assistant guidance
├── architecture.md                    # Main architecture document
├── CLIENT-MODULE-ARCHITECTURE.md      # Client modular design
├── CLIENT-DESIGN-WORKING-DOC-V2.md    # Detailed client design (working doc)
├── COMPARISON-NODE-VS-RUST-STORAGE.md # Node.js vs Rust comparison
│
├── docs/
│   ├── 01-overview/                   # System overview
│   │   ├── system-overview.md
│   │   ├── key-decisions.md
│   │   └── technology-stack.md
│   │
│   ├── 02-client/                     # Client architecture
│   │   ├── overview.md                # Client overview
│   │   └── modules/                   # ✨ NEW: Modular architecture
│   │       ├── overview.md            # Module architecture overview
│   │       ├── storage-service/       # Module 1: Storage (✅ Designed)
│   │       │   └── README.md
│   │       ├── execution-engine/      # Module 2: Execution (🔄 TBD)
│   │       │   └── README.md
│   │       ├── estate-scanner/        # Module 3: Scanner (🔄 TBD)
│   │       │   └── README.md
│   │       └── request-builder/       # Module 4: Request Builder (🔄 TBD)
│   │           └── README.md
│   │
│   ├── 03-server/                     # Server ecosystem
│   │   ├── overview.md
│   │   ├── agents/
│   │   │   └── overview.md
│   │   ├── microservices/
│   │   │   └── overview.md
│   │   ├── data/
│   │   ├── infrastructure/
│   │   └── integration/
│   │
│   ├── 04-flows/                      # Request/sync/execution flows
│   ├── 05-security/                   # Security architecture
│   ├── 06-data/                       # Data models and schemas
│   └── 07-operations/                 # Deployment, monitoring, ops
│
├── diagrams/                          # Architecture diagrams
│   ├── system/
│   ├── flows/
│   ├── components/
│   └── source/
│
├── adr/                               # Architecture Decision Records
│   ├── README.md
│   └── template.md
│
├── repositories/                      # Repository catalog
│   └── repos.md
│
└── reference/                         # Reference implementations
    └── CONTEXT-MANAGEMENT-ARCHITECTURE.md  # Node.js reference
```

## Client Module Documentation (NEW)

The client documentation now reflects the modular architecture:

### Module Structure
```
docs/02-client/modules/
├── overview.md                    # Module architecture overview
├── storage-service/               # ✅ Designed
│   ├── README.md
│   ├── architecture.md           # (To be added)
│   ├── api.md                    # (To be added)
│   ├── configuration.md          # (To be added)
│   └── encryption.md             # (To be added)
├── execution-engine/              # 🔄 To be designed
│   ├── README.md
│   ├── architecture.md           # (TBD)
│   ├── api.md                    # (TBD)
│   ├── execution-strategies.md   # (TBD)
│   └── approval-workflow.md      # (TBD)
├── estate-scanner/                # 🔄 To be designed
│   ├── README.md
│   ├── architecture.md           # (TBD)
│   ├── api.md                    # (TBD)
│   ├── scanning-strategies.md    # (TBD)
│   └── service-support.md        # (TBD)
└── request-builder/               # 🔄 To be designed
    ├── README.md
    ├── architecture.md           # (TBD)
    ├── api.md                    # (TBD)
    ├── request-flow.md           # (TBD)
    └── server-protocol.md        # (TBD)
```

## Key Documents

### High-Level
- [README.md](README.md) - Repository purpose
- [architecture.md](architecture.md) - Main architecture (10,000-foot view)
- [CLIENT-MODULE-ARCHITECTURE.md](CLIENT-MODULE-ARCHITECTURE.md) - Modular client design

### Working Documents
- [CLIENT-DESIGN-WORKING-DOC-V2.md](CLIENT-DESIGN-WORKING-DOC-V2.md) - Detailed client design with decisions

### Comparisons
- [COMPARISON-NODE-VS-RUST-STORAGE.md](COMPARISON-NODE-VS-RUST-STORAGE.md) - Node.js reference vs Rust design

### References
- [reference/CONTEXT-MANAGEMENT-ARCHITECTURE.md](reference/CONTEXT-MANAGEMENT-ARCHITECTURE.md) - Node.js implementation

## Documentation Status

### ✅ Complete
- System overview
- Storage Service module design
- Encryption strategy
- Backup strategy
- Node.js comparison

### 🔄 In Progress
- Execution Engine module
- Estate Scanner module
- Request Builder module
- Frontend architecture
- Flow diagrams

### 📋 To Do
- Server agent details
- Server microservice details
- Security implementation details
- Data model schemas
- Deployment architecture
- ADRs (Architecture Decision Records)

## Navigation

### For Developers
Start with: [docs/02-client/modules/overview.md](docs/02-client/modules/overview.md)

### For Architects
Start with: [architecture.md](architecture.md)

### For Implementation
Start with: [CLIENT-MODULE-ARCHITECTURE.md](CLIENT-MODULE-ARCHITECTURE.md)

## Adding New Documentation

### For Client Modules
Add to: `docs/02-client/modules/{module-name}/`

### For Server Components
Add to: `docs/03-server/{component-type}/`

### For Flows
Add to: `docs/04-flows/`

### For ADRs
Add to: `adr/` using [adr/template.md](adr/template.md)

## Maintenance

- Keep module READMEs up-to-date with implementation status
- Update overview documents when architecture changes
- Link between related documents
- Maintain consistent naming conventions