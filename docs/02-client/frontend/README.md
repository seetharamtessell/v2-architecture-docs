# Frontend Architecture

**Status**: ✅ Design Complete | 📚 Tauri Integration Complete
**Framework**: React + TypeScript + Tauri
**Last Updated**: October 2025

---

## Overview

The Escher frontend is a **Tauri-based desktop application** that provides a chat-driven interface for cloud operations. It implements an **MVC architecture** with server-driven UI rendering powered by the UI Rendering Engine and UI Components library.

---

## Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│                    VIEWS                            │
│  Pure React Components (Presentation Layer)         │
│  • Feature Views (Chat, Reports, Estate, etc.)     │
│  • UI Components (30+ React components)             │
│  • UI Rendering Engine (Dynamic orchestrator)       │
└─────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────┐
│                 CONTROLLERS                         │
│  Business Logic & Orchestration Layer               │
│  • ChatController, PlaybookController, etc.         │
│  • Data transformation                              │
│  • Event handling                                   │
└─────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────┐
│                   MODELS                            │
│  Data Layer                                         │
│  • Type definitions                                 │
│  • UI state management (Zustand)                    │
│  • Data transformers                                │
└─────────────────────────────────────────────────────┘
                        ↕
┌─────────────────────────────────────────────────────┐
│                  SERVICES                           │
│  Infrastructure Layer                               │
│  • Tauri IPC (Rust modules)                         │
│  • WebSocket (Server communication)                 │
│  • Cognito (Authentication)                         │
│  • HTTP (API calls)                                 │
└─────────────────────────────────────────────────────┘
```

---

## Key Features

### 1. **Server-Driven UI**
- Server-side UI Agent generates UI specifications
- UI Rendering Engine (client-side) maps specs to React components
- UI Components library (30+ components) renders the actual UI
- Two-phase rendering: Stream (text) → Enhancement (rich UI)

### 2. **MVC Architecture**
- Clear separation of concerns
- Models: Data types + UI state
- Views: Pure presentation
- Controllers: Business logic

### 3. **Rust Integration**
- All backend operations via Tauri commands
- Estate Scanner, Execution Engine, Storage Service
- Real-time event streaming

### 4. **Intelligent Bucketing**
- BUCKET 1: UI Cache (transient, for rendering)
- BUCKET 2: History (persistent, for LLM context)
- Critical for efficient conversation history

---

## Project Structure

```
src/
├── models/              # Data layer
│   ├── types/          # TypeScript interfaces
│   ├── state/          # Zustand stores (UI state)
│   └── transformers/   # Data transformations
│
├── controllers/         # Business logic
│   ├── ChatController.ts
│   ├── PlaybookController.ts
│   ├── EstateScanController.ts
│   ├── RecommendationController.ts
│   ├── AlertController.ts
│   └── PermissionController.ts
│
├── views/               # Presentation layer
│   ├── auth/           # Login, signup
│   ├── chat/           # Ops chat interface
│   ├── estate/         # Learning center, scanner
│   ├── recommendations/
│   ├── alerts/
│   ├── permissions/
│   ├── reports/
│   └── shared/         # Shared components
│
├── services/            # Infrastructure
│   ├── tauri/          # Rust module wrappers
│   ├── websocket/      # Server communication
│   ├── cognito/        # Authentication
│   └── http/           # REST API
│
└── types/               # Shared TypeScript types
```

---

## User Journey Coverage

The frontend supports 8 complete user flows:

1. **Login** → Cognito authentication
2. **Estate Scan** → Trigger AWS resource discovery
3. **Learning Center** → View estate status dashboard
4. **Permissions** → Configure AI agent policies
5. **Recommendations** → View and apply optimization suggestions
6. **Alerts** → Real-time notifications
7. **Ops Chat** → Interactive operations with playbook execution (Explain → Execute → Monitor)
8. **Reports** → Generate visual analytics

---

## Documentation

**Frontend Design** (Complete):
- [MVC Architecture](./mvc-architecture.md) - Detailed MVC design
- [User Flows](./user-flows.md) - Complete user journey documentation (8 flows)
- [UI Components Package](./ui-components.md) - 30+ React presentation components
- [UI Rendering Engine](./ui-rendering-engine.md) - Client-side rendering orchestrator
- [Authentication & Security](./authentication-security.md) - Cognito + JWT + OS Keychain

**Tauri Integration** (Complete):
- [Tauri Commands](../tauri-integration/README.md) - 70+ commands documented
  - [Storage Commands](../tauri-integration/commands-storage.md) - Estate data, chat history, policies
  - [Execution Commands](../tauri-integration/commands-execution.md) - AWS CLI, playbooks
  - [Scanner Commands](../tauri-integration/commands-estate-scanner.md) - Estate scanning
  - [Auth Commands](../tauri-integration/commands-auth.md) - Token management
- [Tauri Events](../tauri-integration/README.md#event-categories) - 15+ events documented
  - [Scan Events](../tauri-integration/events-scan.md) - Real-time scan progress
  - [Execution Events](../tauri-integration/events-execution.md) - Command output streaming
  - [System Events](../tauri-integration/events-system.md) - App lifecycle, errors

**Still Needed**:
- [Data Flow](./data-flow.md) - How data moves through the system ⏳
- [Integration Patterns](./integration-patterns.md) - Frontend ↔ Rust ↔ Server ⏳
- [State Management](./state-management.md) - Zustand stores design ⏳

---

## Key Technologies

- **Framework**: React 18+ with TypeScript
- **Desktop**: Tauri 2.0
- **State Management**: Zustand
- **Styling**: Tailwind CSS
- **Charts**: Recharts
- **WebSocket**: Native WebSocket API
- **Authentication**: AWS Cognito (Amplify)
- **Build**: Vite

---

## Design Principles

### 1. **Separation of Concerns**
- Views are pure presentation (no business logic)
- Controllers handle all business logic
- Models define data structures and UI state
- Services abstract infrastructure

### 2. **Data Ownership**
- Server owns: Playbooks, recommendations, alerts
- Rust owns: Estate data, chat history, policies
- Frontend owns: Transient UI state, UI cache

### 3. **Progressive Enhancement**
- Text streams immediately (0ms wait)
- Rich UI arrives 500ms-2s later
- User never waits

### 4. **Type Safety**
- Full TypeScript coverage
- Shared types with Rust (via cloudops-common)
- Runtime validation

### 5. **Extensibility**
- Plugin system for custom UI components
- Configurable policies
- Themeable UI

---

## Performance Targets

| Metric | Target |
|--------|--------|
| First chunk latency | <100ms |
| Stream complete | <3s |
| Enhancement time | <2s |
| UI render | <100ms |
| Tauri command | <50ms |

---

## Next Steps

- Implement core MVC structure
- Integrate UI Components package and UI Rendering Engine
- Build Tauri command interfaces
- Setup WebSocket client
- Implement authentication flow

---

See individual documentation files for detailed design specifications.