# Escher Client Architecture - Complete Guide

**👋 New to this documentation?** Start here, then pick your path.

---

## 🚀 Quick Start (5 Minutes)

Read: [Architecture Overview](./architecture/overview.md) - 10,000-foot view of the client architecture

---

## 📚 Documentation by Role

### I'm a Frontend Developer
**Focus**: React + TypeScript development

1. [Frontend Architecture](./frontend/README.md) - MVC pattern, state management
2. [MVC Architecture](./frontend/mvc-architecture.md) - Detailed MVC design
3. [User Flows](./frontend/user-flows.md) - 8 complete user journeys
4. [UI Components](./frontend/ui-components.md) - 30+ component vocabulary
5. [UI Rendering Engine](./frontend/ui-rendering-engine.md) - Server-driven UI
6. [Authentication & Security](./frontend/authentication-security.md) - Cognito integration

---

### I'm a Backend Developer (Rust)
**Focus**: Rust modules and Tauri integration

1. [Modules Overview](./modules/overview.md) - Overview of all Rust crates
2. **Detailed Module Docs** - Navigate to [/docs/04-services/](../04-services/) for:
   - [Storage Service](../04-services/storage-service/) - Local data persistence + RAG
   - [Execution Engine](../04-services/execution-engine/) - Command execution
   - [Estate Scanner](../04-services/estate-scanner/) - Cloud resource discovery
   - [Common Types](../04-services/common/) - Shared data structures
3. [Integration Guide](./modules/integration-guide.md) - How frontend uses backend modules _(coming soon)_

---

### I'm Building the UI (Parallel Development)
**Focus**: Independent frontend development with mocks

→ **[UI Team Implementation Guide](./ui-team-implementation/README.md)**

⚠️ **Note**: This guide is specifically for the UI team developing in parallel. Other readers should use the Frontend Architecture docs above.

---

### I'm Integrating Frontend ↔ Rust (Tauri)
**Focus**: IPC bridge, commands, and events

1. [Tauri Integration Overview](./tauri-integration/README.md)
2. **Commands** (Frontend → Rust):
   - [Storage Commands](./tauri-integration/commands-storage.md) - 25+ commands
   - [Execution Commands](./tauri-integration/commands-execution.md) - 15+ commands
   - [Estate Scanner Commands](./tauri-integration/commands-estate-scanner.md) - 15+ commands
   - [Auth Commands](./tauri-integration/commands-auth.md) - 15+ commands
   - [Request Builder Commands](./tauri-integration/commands-request-builder.md) - Preliminary
3. **Events** (Rust → Frontend):
   - [Scan Events](./tauri-integration/events-scan.md) - 6 events
   - [Execution Events](./tauri-integration/events-execution.md) - 6 events
   - [System Events](./tauri-integration/events-system.md) - 8 events

---

### I Need the Complete Reference
**Focus**: Comprehensive architecture documentation

→ **[Architecture Summary](./architecture/summary.md)** (400-500 lines, complete reference)

---

## 🗺️ Complete Documentation Map

### Core Architecture
- [Architecture Overview](./architecture/overview.md) - 10,000-foot view (150-200 lines)
- [Architecture Summary](./architecture/summary.md) - Comprehensive reference (400-500 lines)

### Frontend (React + TypeScript)
- [Frontend README](./frontend/README.md) - Frontend overview
- [MVC Architecture](./frontend/mvc-architecture.md) - Model-View-Controller pattern
- [User Flows](./frontend/user-flows.md) - 8 complete user journeys
- [UI Components](./frontend/ui-components.md) - 30+ presentation components
- [UI Rendering Engine](./frontend/ui-rendering-engine.md) - Server-driven UI
- [Authentication & Security](./frontend/authentication-security.md) - Cognito integration

### Backend Modules (Rust Crates)
- [Modules Overview](./modules/overview.md) - Overview of all crates
- **Detailed Docs** → [/docs/04-services/](../04-services/)

### Tauri Integration (IPC Bridge)
- [Tauri Integration README](./tauri-integration/README.md)
- **Commands**: 5 files documenting 70+ commands
- **Events**: 3 files documenting 15+ events

### UI Team (Parallel Development)
- [UI Team Implementation Guide](./ui-team-implementation/README.md)
- [Architecture Guide](./ui-team-implementation/01-architecture.md) - 4,800 lines
- [Implementation Plan](./ui-team-implementation/02-implementation-plan.md) - 5,200 lines
- [Project Structure](./ui-team-implementation/03-project-structure.md) - 4,100 lines
- [Mock Contracts](./ui-team-implementation/04-mock-contracts.md) - 4,900 lines
- [Claude Prompts](./ui-team-implementation/05-claude-prompts.md) - 25 ready-to-use prompts

---

## 📋 Documentation Status

- [Completion Status](./meta/COMPLETION-STATUS.md) - What's complete, what's in progress

---

## 🔍 Quick Answers to Common Questions

**Q: Where do cloud credentials get stored?**
A: OS Keychain (macOS/Windows/Linux) - see [Architecture Overview](./architecture/overview.md)

**Q: How does the client store cloud estate data?**
A: Qdrant Vector DB (6-collection strategy) - see [Storage Service](../04-services/storage-service/)

**Q: Is this AWS-only?**
A: No! Multi-cloud support for AWS/Azure/GCP - see [Architecture Overview](./architecture/overview.md)

**Q: Does the server store any user data?**
A: No! Server is 100% stateless - see [Architecture Overview](./architecture/overview.md)

**Q: How do I call Rust functions from React?**
A: Via Tauri commands - see [Tauri Integration](./tauri-integration/README.md)

**Q: How many Rust modules are there?**
A: 5 core modules (Storage, Execution, Scanner, Request Builder, Common Types) - see [Modules Overview](./modules/overview.md)

---

## 🎯 Key Architectural Principles

1. **Privacy-First**: All data stays on client, server is 100% stateless
2. **Multi-Cloud**: Support for AWS, Azure, GCP with pluggable architecture
3. **Local-First**: Cloud credentials and estate data never leave the device
4. **Server-Driven UI**: Dynamic rendering with 30+ components
5. **Type Safety**: Full TypeScript (frontend) + Rust (backend) coverage
6. **Progressive Enhancement**: Text streams first, rich UI arrives later

---

## 📚 Related Documentation

- **Main Project Summary**: [/PROJECT-SUMMARY.md](../../PROJECT-SUMMARY.md)
- **Product Vision**: [/docs/01-overview/PRODUCT-VISION.md](../01-overview/PRODUCT-VISION.md)
- **Server Architecture**: [/docs/03-server/](../03-server/)
- **Shared Services**: [/docs/04-services/](../04-services/)

---

**Total Client Documentation**: 42+ files, ~25,000 lines
**Status**: 90% complete
**Last Updated**: October 2025