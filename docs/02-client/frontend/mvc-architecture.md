# MVC Architecture Design

**Pattern**: Model-View-Controller
**Purpose**: Clean separation of concerns between data, presentation, and business logic

---

## Architecture Overview

```
USER INTERACTION
       ↓
    VIEWS (Pure Presentation)
       ↓
  CONTROLLERS (Business Logic)
       ↓
    MODELS (Data + State)
       ↓
   SERVICES (Infrastructure)
       ↓
EXTERNAL SYSTEMS (Rust, Server, AWS)
```

---

## 1. Models Layer

**Purpose**: Data structures, type definitions, and transient UI state

### 1.1 Structure

```
models/
├── types/              # Type definitions (interfaces, schemas)
│   ├── chat-types.ts
│   ├── playbook-types.ts
│   ├── estate-types.ts
│   ├── recommendation-types.ts
│   ├── alert-types.ts
│   └── permission-types.ts
│
├── state/              # UI state management (Zustand stores)
│   ├── ChatUIState.ts
│   ├── PlaybookUIState.ts
│   ├── EstateUIState.ts
│   ├── RecommendationUIState.ts
│   ├── AlertUIState.ts
│   └── AuthState.ts
│
└── transformers/       # Data transformation logic
    ├── PlaybookTransformer.ts
    ├── EstateTransformer.ts
    └── RecommendationTransformer.ts
```

### 1.2 Responsibilities

**Types**:
- Define TypeScript interfaces
- Mirror Rust types from cloudops-common
- Server response schemas
- No logic, only structure

**State**:
- Transient UI state only (dropdown open, selected item, filters)
- NOT persistent data (that lives in Rust or Server)
- Zustand stores for reactive updates

**Transformers**:
- Convert server format → UI format
- Flatten nested structures
- Add computed fields
- Format dates, numbers, etc.

### 1.3 Design Principles

- **No Persistence**: Models don't store server/Rust data
- **UI State Only**: Selected items, modal visibility, form state
- **Type Contracts**: Define clear interfaces for all data
- **Transformation**: Server data → UI-friendly format

---

## 2. Views Layer

**Purpose**: Pure presentation components, zero business logic

### 2.1 Structure

```
views/
├── auth/
│   ├── LoginView.tsx
│   └── components/
│       ├── LoginForm.tsx
│       └── AuthButton.tsx
│
├── chat/
│   ├── OpsChatView.tsx
│   └── components/
│       ├── MessageList.tsx
│       ├── MessageItem.tsx
│       ├── StreamingIndicator.tsx
│       └── InputArea.tsx
│
├── estate/
│   ├── LearningCenterView.tsx
│   ├── EstateScanView.tsx
│   └── components/
│       ├── EstateOverviewCard.tsx
│       ├── ResourceCard.tsx
│       ├── ScanProgressBar.tsx
│       └── ResourceList.tsx
│
├── playbook/
│   └── components/
│       ├── PlaybookCard.tsx
│       ├── PlaybookStepList.tsx
│       ├── PlaybookStep.tsx
│       └── ExecutionProgress.tsx
│
├── recommendations/
│   ├── RecommendationsView.tsx
│   └── components/
│       ├── RecommendationCard.tsx
│       └── RecommendationDetails.tsx
│
├── alerts/
│   ├── AlertsView.tsx
│   └── components/
│       ├── AlertList.tsx
│       ├── AlertCard.tsx
│       └── NotificationBell.tsx
│
├── permissions/
│   ├── PermissionsView.tsx
│   └── components/
│       ├── PolicyEditor.tsx
│       └── ResourceConstraintsPanel.tsx
│
└── shared/
    ├── ui-agent-components/    # Uses @escher/ui-agent-components package
    └── base/                   # Base UI components
```

### 2.2 Responsibilities

**Views**:
- Receive data via props
- Emit events via callbacks
- Render UI elements
- Handle local UI state (hover, focus)

**Views NEVER**:
- Make API calls
- Access services directly
- Implement business logic
- Manage global state

### 2.3 Component Contracts

Every view component follows:

**Input**: Props (data + callbacks)
```
Props:
  - data: Data to display
  - onAction: Callback for user actions
  - isLoading: Loading state
  - error: Error state
```

**Output**: Events
```
Events:
  - onSubmit(data)
  - onClick(id)
  - onChange(value)
```

---

## 3. Controllers Layer

**Purpose**: Business logic, orchestration, data flow coordination

### 3.1 Structure

```
controllers/
├── ChatController.ts
├── PlaybookController.ts
├── EstateScanController.ts
├── RecommendationController.ts
├── AlertController.ts
├── PermissionController.ts
└── AuthController.ts
```

### 3.2 Responsibilities

**Controllers**:
- Handle user actions from Views
- Call Services (Tauri, WebSocket, HTTP)
- Transform data using Transformers
- Update Models (UI state)
- Coordinate between multiple services
- Implement business rules

**Example Responsibilities**:

**ChatController**:
- Send messages to server
- Handle stream chunks
- Implement bucketing logic (UI Cache vs History)
- Save history to Rust Storage Service

**PlaybookController**:
- Receive playbooks from server
- Handle "Explain Plan" logic
- Trigger execution via Rust Execution Engine
- Monitor execution progress

**EstateScanController**:
- Trigger scans via Rust Estate Scanner
- Listen to progress events
- Fetch estate data from Rust Storage Service
- Update UI state (filters, selections)

### 3.3 Design Pattern

All controllers follow this pattern:

1. **Receive action** from View
2. **Validate** input
3. **Call Service** (Tauri/WebSocket/HTTP)
4. **Transform** response data
5. **Update Model** (UI state)
6. **View re-renders** automatically (reactive)

### 3.4 Key Implementation: Bucketing

**Critical Logic in ChatController**:

When enhancement arrives from server:
```
Enhancement {
  ui: UIMode          → BUCKET 1: ChatUIState.uiCache (transient)
  history: HistoryMode → BUCKET 2: Rust Storage Service (persistent)
}
```

**Why This Matters**:
- UI cache: For re-rendering (2-10KB per message)
- History: For LLM context (200-500 bytes per message)
- 20x size difference
- NEVER mix the two buckets

---

## 4. Services Layer

**Purpose**: Infrastructure abstraction, external system communication

### 4.1 Structure

```
services/
├── tauri/
│   ├── StorageService.ts
│   ├── ExecutionEngineService.ts
│   ├── EstateScannerService.ts
│   └── RequestBuilderService.ts
│
├── websocket/
│   └── WebSocketService.ts
│
├── cognito/
│   └── CognitoAuthService.ts
│
└── http/
    └── APIService.ts
```

### 4.2 Responsibilities

**Services**:
- Abstract infrastructure details
- Provide clean API for Controllers
- Handle low-level communication
- Manage connections/sessions
- Handle retries, errors

**Tauri Services**:
- Wrap Tauri `invoke()` commands
- Listen to Tauri events
- Type-safe Rust integration

**WebSocket Service**:
- Manage WebSocket connection
- Emit typed events (stream, enhance)
- Handle reconnection

**Cognito Service**:
- Authentication flows
- Token management
- Session handling

---

## Data Flow Examples

### Example 1: User Sends Chat Message

```
┌─────────────────────────────────────────────────────┐
│ 1. USER ACTION                                      │
│    User types: "Stop RDS db-prod-01"                │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 2. VIEW: OpsChatView                                │
│    - InputArea captures text                        │
│    - Calls: onSendMessage("Stop RDS db-prod-01")    │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 3. CONTROLLER: ChatController.sendMessage()         │
│    - Create user message object                     │
│    - Update ChatUIState.messages (add message)      │
│    - Fetch history from Rust Storage Service        │
│    - Send to server via WebSocketService            │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 4. SERVICE: WebSocketService.send()                 │
│    - Serialize: {content, context: {history}}       │
│    - Send over WebSocket                            │
└─────────────────────────────────────────────────────┘
                    ↓
            [SERVER PROCESSES]
                    ↓
┌─────────────────────────────────────────────────────┐
│ 5. SERVICE: WebSocketService emits 'stream' event   │
│    - Receives stream chunks                         │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 6. CONTROLLER: ChatController.handleStreamChunk()   │
│    - Update ChatUIState.messages (append chunk)     │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 7. MODEL: ChatUIState updates                       │
│    - Zustand triggers re-render                     │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 8. VIEW: OpsChatView re-renders                     │
│    - MessageList shows updated text                 │
│    - StreamingIndicator visible                     │
└─────────────────────────────────────────────────────┘
                    ↓
            [Stream completes]
                    ↓
┌─────────────────────────────────────────────────────┐
│ 9. SERVICE: WebSocketService emits 'enhance' event  │
│    - Receives Enhancement {ui, history}             │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 10. CONTROLLER: ChatController.handleEnhancement()  │
│     CRITICAL BUCKETING:                             │
│     - BUCKET 1: Cache ui in ChatUIState.uiCache     │
│     - BUCKET 2: Save history to Rust Storage        │
│     - Update message type to 'enhanced'             │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 11. VIEW: OpsChatView re-renders                    │
│     - MessageItem shows enhanced UI                 │
│     - DynamicRenderer displays playbook             │
└─────────────────────────────────────────────────────┘
```

### Example 2: User Executes Playbook

```
┌─────────────────────────────────────────────────────┐
│ 1. VIEW: PlaybookCard                               │
│    User clicks "Execute"                            │
│    Calls: onExecute()                               │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 2. CONTROLLER: PlaybookController.executePlaybook() │
│    - Update PlaybookUIState (executionInProgress)   │
│    - Loop through playbook steps                    │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 3. SERVICE: ExecutionEngineService.executeCommand() │
│    - Calls Tauri: invoke('execute_command')         │
│    - Returns executionId                            │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 4. RUST: Execution Engine                           │
│    - Executes AWS CLI command                       │
│    - Streams output via Tauri events                │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 5. SERVICE: ExecutionEngineService listens          │
│    - Tauri event: 'execution_output_{id}'           │
│    - Emits to Controller                            │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 6. CONTROLLER: PlaybookController receives output   │
│    - Updates step output (transient, not stored)    │
│    - Passes to View via props                       │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ 7. VIEW: ExecutionProgress re-renders               │
│    - Shows real-time command output                 │
│    - Updates progress indicators                    │
└─────────────────────────────────────────────────────┘
```

---

## Data Ownership

| Data Type | Owner | Frontend Storage | Access Method |
|-----------|-------|------------------|---------------|
| Playbooks | Server | None | Received via WebSocket |
| Chat Messages (display) | Frontend | ChatUIState (runtime) | N/A |
| Chat History (LLM) | Rust Storage | None | Via Tauri command |
| UI Cache | Frontend | ChatUIState.uiCache (runtime) | N/A |
| Estate Data | Rust Storage | None | Via Tauri command |
| Recommendations | Server | None | Via HTTP/WebSocket |
| Alerts | Server | None | Via HTTP/WebSocket |
| Policies | Rust Storage | None | Via Tauri command |

---

## Design Principles Summary

### 1. Models
- **Type definitions** (interfaces, schemas)
- **UI state only** (transient, not persistent)
- **No business logic**

### 2. Views
- **Pure presentation** (props in, events out)
- **Zero business logic**
- **Zero API calls**

### 3. Controllers
- **Business logic orchestration**
- **Service coordination**
- **Data transformation**
- **State management**

### 4. Services
- **Infrastructure abstraction**
- **Clean API for Controllers**
- **Handle low-level details**

---

## Benefits

### Separation of Concerns
- Each layer has single responsibility
- Easy to understand and maintain
- Clear boundaries

### Testability
- Models: Unit test transformers
- Views: Component test with mocked props
- Controllers: Unit test with mocked services
- Services: Integration test with real infrastructure

### Reusability
- Views can be reused with different data
- Controllers can be reused in different contexts
- Services are infrastructure-agnostic

### Maintainability
- Changes isolated to specific layers
- Bug fixes don't ripple across system
- Easy to refactor

---

This MVC architecture ensures clean, maintainable, and scalable frontend code.