# Key Architecture Decisions

This document tracks major architectural decisions. For detailed ADRs, see the [adr/](../../adr/) directory.

## Summary of Key Decisions

1. **Tauri over Electron** - Smaller footprint, better performance, Rust backend
2. **Local Vector DB (Qdrant)** - Fast semantic search, privacy, offline capability
3. **Stateless Server** - Scalability, no AWS estate storage on server
4. **Multi-Agent System** - Specialized agents for classification, operations, risk assessment
5. **Microservices Architecture** - Server-side modularity and independent scaling

---
*See [adr/](../../adr/) for detailed Architecture Decision Records.*
