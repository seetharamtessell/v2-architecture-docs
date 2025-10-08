# Architecture Decision Records (ADR)

## What is an ADR?
An Architecture Decision Record (ADR) captures an important architectural decision made along with its context and consequences.

## When to Write an ADR?
Write an ADR when making decisions about:
- Technology stack choices
- System architecture patterns
- Data storage strategies
- Integration approaches
- Security models
- Performance trade-offs

## ADR Format
Use the [template.md](template.md) for consistency.

## ADR Naming Convention
`NNNN-short-title.md` where NNNN is a sequential number (e.g., `0001-tauri-over-electron.md`)

## ADR Index

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](0001-tauri-over-electron.md) | Tauri over Electron | Proposed | TBD |
| [0002](0002-local-vector-db.md) | Local Vector DB for Resource Search | Proposed | TBD |
| [0003](0003-stateless-server.md) | Stateless Server Architecture | Proposed | TBD |
| [0004](0004-microservices-architecture.md) | Microservices for Server | Proposed | TBD |
| [0005](0005-agent-orchestration.md) | Multi-Agent System Design | Proposed | TBD |

---
*ADRs are immutable once accepted. To change a decision, create a new ADR that supersedes the old one.*