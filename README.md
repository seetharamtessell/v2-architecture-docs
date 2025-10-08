# V2 Architecture Documentation

## Purpose

This repository maintains the **complete architecture documentation** for the AWS CloudOps AI Agent system (V2). It serves as the **single source of truth** for architectural decisions, component designs, and related repositories across the entire project ecosystem.

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

1. **Client-side AWS Estate Management**: All AWS resource data and credentials remain local
2. **Server-side Operations Knowledge**: Stateless server provides playbook execution logic
3. **Privacy First**: No AWS credentials or estate data sent to server
4. **Semantic Search**: Local Qdrant vector DB enables fast, fuzzy resource lookup
5. **Precision Over Guessing**: Client sends complete context; server generates exact scripts

## Repository Structure

- [architecture.md](architecture.md) - Main architecture document with detailed system design
- [v2-architecture-docs/](v2-architecture-docs/) - Detailed documentation by component/domain
- [v2-ui-agent-golang/](v2-ui-agent-golang/) - Related implementation repositories

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
