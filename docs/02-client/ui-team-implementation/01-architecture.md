# UI Architecture Guide

**Framework**: React 18+ with TypeScript + Tauri 2.0
**Pattern**: MVC (Model-View-Controller)
**State**: Zustand
**Styling**: Tailwind CSS

---

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│            VIEWS (Presentation)             │
│  Pure React Components                      │
│  • Feature Views (Chat, Scan, Estate, etc.)│
│  • UI Agent Components (Dynamic Rendering)  │
│  • Shared Components (Button, Card, etc.)   │
└─────────────────────────────────────────────┘
                    ↕ Props & Callbacks
┌─────────────────────────────────────────────┐
│         CONTROLLERS (Business Logic)        │
│  • ChatController                           │
│  • PlaybookController                       │
│  • EstateScanController                     │
│  • RecommendationController                 │
│  • AlertController                          │
│  • PermissionController                     │
└─────────────────────────────────────────────┘
                    ↕ Read/Write State
┌─────────────────────────────────────────────┐
│            MODELS (Data Layer)              │
│  • TypeScript Interfaces                    │
│  • Zustand Stores (UI State)                │
│  • Data Transformers                        │
└─────────────────────────────────────────────┘
                    ↕ Call Services
┌─────────────────────────────────────────────┐
│         SERVICES (Infrastructure)           │
│  • TauriService (Rust IPC)                  │
│  • WebSocketService (Server)                │
│  • APIService (HTTP REST)                   │
│  • AuthService (Cognito)                    │
└─────────────────────────────────────────────┘
```

---

## MVC Pattern Details

### **VIEWS** (Presentation Layer)

**Responsibility**: Pure UI rendering, no business logic

**Rules**:
- Only receive data via props
- Only communicate via callbacks
- No direct service calls
- No state management (except local UI state like form inputs)

**Types of Views**:

1. **Feature Views** (Pages)
   - `OpsChatView.tsx` - Main chat interface
   - `EstateScanView.tsx` - Estate scanner page
   - `LearningCenterView.tsx` - Dashboard
   - `PermissionsView.tsx` - Policy management
   - `RecommendationsView.tsx` - AI recommendations
   - `AlertsView.tsx` - Notifications
   - `ReportsView.tsx` - Analytics

2. **UI Agent Components** (Dynamic Rendering)
   - `PlaybookCard.tsx` - Playbook display
   - `ExecutionProgress.tsx` - Real-time execution
   - `ResourceCard.tsx` - AWS resource display
   - `ChartDisplay.tsx` - Dynamic charts
   - `TableDisplay.tsx` - Dynamic tables

3. **Shared Components** (Reusable)
   - `Button.tsx`, `Card.tsx`, `Modal.tsx`
   - `Input.tsx`, `Select.tsx`, `Checkbox.tsx`
   - `Toast.tsx`, `Spinner.tsx`, `ProgressBar.tsx`

**Example View**:
```typescript
// views/chat/OpsChatView.tsx
interface OpsChatViewProps {
  messages: Message[];
  onSendMessage: (text: string) => void;
  isLoading: boolean;
}

export const OpsChatView: React.FC<OpsChatViewProps> = ({
  messages,
  onSendMessage,
  isLoading
}) => {
  return (
    <div className="chat-container">
      <MessageList messages={messages} />
      <ChatInput onSend={onSendMessage} disabled={isLoading} />
    </div>
  );
};
```

---

### **CONTROLLERS** (Business Logic Layer)

**Responsibility**: Orchestrate data flow, handle events, transform data

**Rules**:
- Call services (Tauri, WebSocket, API)
- Update Zustand stores
- Transform data for views
- Handle complex logic
- Never render UI directly

**Controller Pattern**:
```typescript
// controllers/ChatController.ts
import { useChatStore } from '@/models/state/chatStore';
import { TauriService } from '@/services/tauri/TauriService';
import { WebSocketService } from '@/services/websocket/WebSocketService';

export class ChatController {
  private chatStore = useChatStore.getState();
  private tauriService: TauriService;
  private wsService: WebSocketService;

  constructor() {
    this.tauriService = new TauriService();
    this.wsService = new WebSocketService();
  }

  async sendMessage(text: string) {
    // 1. Add user message to UI
    this.chatStore.addMessage({
      role: 'user',
      content: text,
      timestamp: Date.now()
    });

    // 2. Get chat history from Rust Storage
    const history = await this.tauriService.getChatHistory(
      this.chatStore.contextId
    );

    // 3. Send to server via WebSocket
    this.wsService.send({
      type: 'chat_message',
      payload: { text, history }
    });

    // 4. Listen for streaming response
    this.wsService.on('text_chunk', this.handleTextChunk.bind(this));
    this.wsService.on('enhancement', this.handleEnhancement.bind(this));
  }

  private handleTextChunk(chunk: string) {
    this.chatStore.appendToLastMessage(chunk);
  }

  private async handleEnhancement(data: Enhancement) {
    // BUCKET 1: UI cache (transient)
    this.chatStore.cacheUI(data.ui);

    // BUCKET 2: History (persistent, via Rust)
    await this.tauriService.saveMessageHistory(data.history);
  }
}
```

**Using Controller in View**:
```typescript
// views/chat/OpsChatView.tsx
export const OpsChatView = () => {
  const { messages, isLoading } = useChatStore();
  const controller = useMemo(() => new ChatController(), []);

  const handleSend = (text: string) => {
    controller.sendMessage(text);
  };

  return (
    <div>
      <MessageList messages={messages} />
      <ChatInput onSend={handleSend} disabled={isLoading} />
    </div>
  );
};
```

---

### **MODELS** (Data Layer)

**Responsibility**: Define data structures and UI state

**Components**:

1. **TypeScript Interfaces** (`models/types/`)
   - Define all data shapes
   - Shared between layers
   - Type safety

```typescript
// models/types/Message.ts
export interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
  metadata?: MessageMetadata;
}

export interface MessageMetadata {
  resourcesMentioned?: string[];
  commandsExecuted?: string[];
}
```

2. **Zustand Stores** (`models/state/`)
   - Manage UI state
   - Global state accessible everywhere
   - Reactive updates

```typescript
// models/state/chatStore.ts
import { create } from 'zustand';

interface ChatState {
  contextId: string;
  messages: Message[];
  uiCache: Map<string, UIEnhancement>;
  isLoading: boolean;

  // Actions
  addMessage: (msg: Message) => void;
  appendToLastMessage: (text: string) => void;
  cacheUI: (msgId: string, ui: UIEnhancement) => void;
  clearCache: () => void;
}

export const useChatStore = create<ChatState>((set) => ({
  contextId: '',
  messages: [],
  uiCache: new Map(),
  isLoading: false,

  addMessage: (msg) => set((state) => ({
    messages: [...state.messages, msg]
  })),

  appendToLastMessage: (text) => set((state) => {
    const messages = [...state.messages];
    const lastMsg = messages[messages.length - 1];
    if (lastMsg) {
      lastMsg.content += text;
    }
    return { messages };
  }),

  cacheUI: (msgId, ui) => set((state) => {
    const cache = new Map(state.uiCache);
    cache.set(msgId, ui);
    return { uiCache: cache };
  }),

  clearCache: () => set({ uiCache: new Map() })
}));
```

3. **Data Transformers** (`models/transformers/`)
   - Convert between formats
   - API response → UI format
   - UI format → API request

```typescript
// models/transformers/resourceTransformer.ts
export class ResourceTransformer {
  static toUIFormat(apiResource: APIResource): Resource {
    return {
      id: apiResource.identifier,
      name: apiResource.name,
      type: apiResource.resource_type,
      region: apiResource.region,
      state: apiResource.state,
      tags: apiResource.tags || {}
    };
  }

  static toAPIFormat(resource: Resource): APIResource {
    return {
      identifier: resource.id,
      name: resource.name,
      resource_type: resource.type,
      region: resource.region,
      state: resource.state,
      tags: resource.tags
    };
  }
}
```

---

### **SERVICES** (Infrastructure Layer)

**Responsibility**: Abstract external systems (Rust, Server, AWS)

**Services**:

1. **TauriService** - Rust IPC calls
2. **WebSocketService** - Server streaming
3. **APIService** - HTTP REST calls
4. **AuthService** - Cognito authentication

**Service Pattern**:
```typescript
// services/tauri/TauriService.ts
import { invoke, listen } from '@tauri-apps/api';

export class TauriService {
  // Commands
  async getChatHistory(contextId: string): Promise<Message[]> {
    return await invoke('get_conversation_history', { contextId });
  }

  async saveMessageHistory(history: MessageHistory): Promise<void> {
    await invoke('save_message', { message: history });
  }

  async startScan(config: ScanConfig): Promise<string> {
    return await invoke('start_scan', { config });
  }

  async executePlaybook(playbook: Playbook): Promise<string> {
    return await invoke('execute_playbook', { playbook });
  }

  // Events
  onScanProgress(callback: (progress: ScanProgress) => void) {
    return listen('scan_progress', (event) => {
      callback(event.payload as ScanProgress);
    });
  }

  onExecutionOutput(callback: (output: ExecutionOutput) => void) {
    return listen('execution_output', (event) => {
      callback(event.payload as ExecutionOutput);
    });
  }
}
```

---

## Data Flow Examples

### Example 1: User Sends Chat Message

```
1. User types message in OpsChatView
   ↓
2. OpsChatView calls onSendMessage callback
   ↓
3. ChatController.sendMessage() triggered
   ↓
4. Controller adds message to chatStore (UI updates immediately)
   ↓
5. Controller calls tauriService.getChatHistory()
   ↓
6. Controller sends to server via wsService.send()
   ↓
7. Server streams response back
   ↓
8. Controller receives 'text_chunk' events
   ↓
9. Controller updates chatStore (UI updates in real-time)
   ↓
10. Server sends 'enhancement' with rich UI
    ↓
11. Controller buckets:
    - ui → chatStore.uiCache (BUCKET 1)
    - history → tauriService.saveMessageHistory() (BUCKET 2)
    ↓
12. View re-renders with enhanced UI
```

### Example 2: User Triggers Estate Scan

```
1. User submits scan form in EstateScanView
   ↓
2. EstateScanView calls onStartScan callback
   ↓
3. EstateScanController.startScan() triggered
   ↓
4. Controller updates scanStore (status: 'scanning')
   ↓
5. Controller calls tauriService.startScan()
   ↓
6. Rust Estate Scanner starts (async)
   ↓
7. Rust emits 'scan_progress' events
   ↓
8. TauriService.onScanProgress() receives events
   ↓
9. Controller updates scanStore with progress
   ↓
10. EstateScanView re-renders progress bar
    ↓
11. Scan completes
    ↓
12. Controller updates scanStore (status: 'complete')
    ↓
13. View shows summary
```

### Example 3: User Executes Playbook

```
1. User clicks "Execute" on PlaybookCard
   ↓
2. PlaybookCard calls onExecute callback
   ↓
3. PlaybookController.executePlaybook() triggered
   ↓
4. Controller updates playbookStore (status: 'executing')
   ↓
5. Controller calls tauriService.executePlaybook()
   ↓
6. Rust Execution Engine starts
   ↓
7. Rust emits 'execution_output' events
   ↓
8. TauriService.onExecutionOutput() receives events
   ↓
9. Controller appends output to playbookStore
   ↓
10. ExecutionProgress component re-renders with new output
    ↓
11. All steps complete
    ↓
12. Controller updates playbookStore (status: 'completed')
    ↓
13. View shows completion summary
```

---

## State Management Strategy

### Zustand Stores (One per domain)

```typescript
// 1. Chat Store
useChatStore - messages, uiCache, contextId

// 2. Estate Store
useEstateStore - resources, filters, selectedResource

// 3. Scan Store
useScanStore - scanProgress, scanResults, scanStatus

// 4. Playbook Store
usePlaybookStore - currentPlaybook, executionStatus, stepOutputs

// 5. Auth Store
useAuthStore - user, tokens, sessionInfo

// 6. Alert Store
useAlertStore - alerts, unreadCount

// 7. Recommendation Store
useRecommendationStore - recommendations, filters
```

### Store Organization Pattern

```typescript
interface Store {
  // State
  data: SomeData;
  status: 'idle' | 'loading' | 'success' | 'error';
  error?: string;

  // Actions
  fetch: () => Promise<void>;
  update: (data: Partial<SomeData>) => void;
  reset: () => void;
}
```

---

## Component Hierarchy

```
App
├── AppLayout
│   ├── Header
│   │   ├── UserMenu
│   │   └── NotificationBell
│   ├── Sidebar
│   │   └── Navigation
│   └── MainContent
│       └── <Route Views>
│
├── Routes
│   ├── /login → LoginView
│   ├── /dashboard → LearningCenterView
│   ├── /chat → OpsChatView
│   ├── /scan → EstateScanView
│   ├── /permissions → PermissionsView
│   ├── /recommendations → RecommendationsView
│   ├── /alerts → AlertsView
│   └── /reports → ReportsView
│
└── Global Components
    ├── ToastContainer
    ├── ModalManager
    └── LoadingOverlay
```

---

## Routing Structure

```typescript
// src/routes/index.tsx
import { createBrowserRouter } from 'react-router-dom';

export const router = createBrowserRouter([
  {
    path: '/',
    element: <AppLayout />,
    children: [
      { path: '/', element: <Navigate to="/dashboard" /> },
      { path: '/dashboard', element: <LearningCenterView /> },
      { path: '/chat', element: <OpsChatView /> },
      { path: '/scan', element: <EstateScanView /> },
      { path: '/permissions', element: <PermissionsView /> },
      { path: '/recommendations', element: <RecommendationsView /> },
      { path: '/alerts', element: <AlertsView /> },
      { path: '/reports', element: <ReportsView /> }
    ]
  },
  {
    path: '/login',
    element: <LoginView />
  }
]);
```

---

## Design Principles

### 1. **Separation of Concerns**
- Views render UI only
- Controllers handle logic
- Models define data
- Services handle I/O

### 2. **Unidirectional Data Flow**
- State lives in Zustand stores
- Controllers update stores
- Views read from stores
- Props flow down, callbacks flow up

### 3. **Type Safety**
- Everything typed with TypeScript
- No `any` types
- Interfaces for all data structures

### 4. **Component Reusability**
- Small, focused components
- Shared components in `/shared`
- Props-based configuration

### 5. **Performance**
- React.memo for expensive components
- useMemo/useCallback for heavy computations
- Lazy loading for routes
- Virtual scrolling for long lists

---

## Testing Strategy

### Unit Tests (Vitest)
- Controllers: Mock services
- Transformers: Pure functions
- Utilities: Pure functions

### Component Tests (React Testing Library)
- Views: Mock controllers
- Shared components: Test in isolation

### Integration Tests
- Full user flows
- Mock Tauri/WebSocket/API

### E2E Tests (Playwright)
- Critical paths only
- Real Tauri environment

---

## Next Steps

1. Setup project structure (see [03-project-structure.md](./03-project-structure.md))
2. Create mock services (see [04-mock-contracts.md](./04-mock-contracts.md))
3. Build UI components
4. Implement controllers
5. Wire up views

Ready to build! 🚀