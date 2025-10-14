# UI Team - Implementation Plan

**Approach**: Phased, bottom-up development
**Strategy**: Build with mocks first, integrate later
**Dependencies**: Zero for Phase 1-3

---

## Implementation Phases

### **Phase 1: Foundation & Infrastructure**

Build the base project structure and tooling.

#### 1.1 Project Setup
**What to build**:
- [ ] Initialize Tauri + React + TypeScript project
- [ ] Configure Vite build system
- [ ] Setup Tailwind CSS
- [ ] Configure TypeScript strict mode
- [ ] Setup ESLint + Prettier
- [ ] Initialize Git repository

**Commands**:
```bash
npm create tauri-app@latest escher-client
cd escher-client
npm install zustand recharts react-router-dom
npm install -D tailwindcss postcss autoprefixer
npm install -D @types/node
npx tailwindcss init -p
```

**Files to create**:
```
├── tsconfig.json          # TypeScript config
├── tailwind.config.js     # Tailwind config
├── vite.config.ts         # Vite config
├── .eslintrc.js           # ESLint rules
└── .prettierrc            # Prettier config
```

---

#### 1.2 Folder Structure
**What to build**:
- [ ] Create MVC folder structure
- [ ] Setup routing structure
- [ ] Create barrel exports

**Folders to create**:
```
src/
├── models/
│   ├── types/
│   ├── state/
│   └── transformers/
├── views/
│   ├── auth/
│   ├── chat/
│   ├── estate/
│   ├── recommendations/
│   ├── alerts/
│   ├── permissions/
│   ├── reports/
│   └── shared/
├── controllers/
├── services/
│   ├── tauri/
│   ├── websocket/
│   ├── api/
│   └── auth/
├── routes/
├── utils/
└── assets/
```

See [03-project-structure.md](./03-project-structure.md) for complete details.

---

#### 1.3 TypeScript Types Foundation
**What to build**:
- [ ] Define core types (Message, Resource, Playbook, etc.)
- [ ] Create shared interfaces
- [ ] Setup type exports

**Files to create**:
```
src/models/types/
├── index.ts               # Barrel export
├── Message.ts             # Chat message types
├── Resource.ts            # AWS resource types
├── Playbook.ts            # Playbook types
├── User.ts                # User/auth types
├── Recommendation.ts      # Recommendation types
├── Alert.ts               # Alert types
└── Common.ts              # Shared utility types
```

**Example**:
```typescript
// src/models/types/Message.ts
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

---

### **Phase 2: UI Components Library**

Build all reusable components with Storybook (optional but recommended).

#### 2.1 Basic Shared Components
**What to build**:
- [ ] Button (primary, secondary, danger variants)
- [ ] Card (with header, body, footer)
- [ ] Modal (with backdrop, close button)
- [ ] Input (text, email, password)
- [ ] Select (dropdown)
- [ ] Checkbox
- [ ] Radio
- [ ] Badge
- [ ] Tag
- [ ] Spinner
- [ ] ProgressBar
- [ ] Toast (notification)

**Location**: `src/views/shared/`

**Example**:
```typescript
// src/views/shared/Button.tsx
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'md',
  disabled = false,
  onClick,
  children
}) => {
  const baseClasses = 'rounded font-semibold transition-colors';
  const variantClasses = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
    danger: 'bg-red-600 text-white hover:bg-red-700'
  };
  const sizeClasses = {
    sm: 'px-3 py-1 text-sm',
    md: 'px-4 py-2 text-base',
    lg: 'px-6 py-3 text-lg'
  };

  return (
    <button
      className={`${baseClasses} ${variantClasses[variant]} ${sizeClasses[size]}`}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
};
```

---

#### 2.2 Chart Components
**What to build**:
- [ ] LineChart
- [ ] BarChart
- [ ] PieChart
- [ ] AreaChart
- [ ] RadarChart

**Dependencies**: Recharts

**Example**:
```typescript
// src/views/shared/charts/LineChart.tsx
import { LineChart as RechartsLine, Line, XAxis, YAxis, CartesianGrid, Tooltip } from 'recharts';

interface LineChartProps {
  data: Array<{ name: string; value: number }>;
  xKey: string;
  yKey: string;
  color?: string;
}

export const LineChart: React.FC<LineChartProps> = ({
  data,
  xKey,
  yKey,
  color = '#8884d8'
}) => {
  return (
    <RechartsLine data={data} width={600} height={300}>
      <CartesianGrid strokeDasharray="3 3" />
      <XAxis dataKey={xKey} />
      <YAxis />
      <Tooltip />
      <Line type="monotone" dataKey={yKey} stroke={color} />
    </RechartsLine>
  );
};
```

---

#### 2.3 UI Agent Components (Dynamic Rendering)
**What to build**:
- [ ] PlaybookCard
- [ ] ExecutionProgress
- [ ] StepIndicator
- [ ] CommandOutput (terminal-style)
- [ ] ResourceCard (multi-cloud: AWS/Azure/GCP)
- [ ] ResourceTable
- [ ] ResourceTree
- [ ] AlertBanner (CRITICAL/HIGH/MEDIUM severity levels)
- [ ] MorningReportCard (scannable sections with quick fixes)
- [ ] AlertConversation (Q&A interface for alerts)
- [ ] TimelineVisualization (incident response timeline)
- [ ] ConversationalReport (morning report with inline Q&A)
- [ ] QuickQuestion (pre-populated question buttons)
- [ ] ReportAnswer (AI-generated answer with follow-up)
- [ ] CostTrendChart (spending over time with anomalies)
- [ ] OptimizationCard (cost savings with 1-click fix)
- [ ] SavingsSummary (total monthly savings)
- [ ] DynamicRenderer (component registry)

**Example**:
```typescript
// src/views/shared/ui-agent/PlaybookCard.tsx
interface PlaybookCardProps {
  playbook: Playbook;
  onExplain?: () => void;
  onExecute?: () => void;
}

export const PlaybookCard: React.FC<PlaybookCardProps> = ({
  playbook,
  onExplain,
  onExecute
}) => {
  return (
    <Card>
      <CardHeader>
        <h3>{playbook.title}</h3>
        <Badge>{playbook.riskLevel}</Badge>
      </CardHeader>
      <CardBody>
        <p>{playbook.summary}</p>
        <div className="steps">
          {playbook.steps.length} steps
        </div>
      </CardBody>
      <CardFooter>
        <Button variant="secondary" onClick={onExplain}>
          Explain Plan
        </Button>
        <Button variant="primary" onClick={onExecute}>
          Execute
        </Button>
      </CardFooter>
    </Card>
  );
};
```

---

### **Phase 3: Mock Services Layer**

Create mock implementations for all external dependencies.

#### 3.1 Mock Tauri Service
**What to build**:
- [ ] MockTauriService with all 70+ commands
- [ ] Mock event emitters
- [ ] Fake data generators

**File**: `src/services/tauri/MockTauriService.ts`

**Example**:
```typescript
// src/services/tauri/MockTauriService.ts
export class MockTauriService {
  // Chat commands
  async getChatHistory(contextId: string): Promise<Message[]> {
    return [
      {
        id: '1',
        role: 'user',
        content: 'Stop RDS db-prod-01',
        timestamp: Date.now() - 60000
      },
      {
        id: '2',
        role: 'assistant',
        content: 'I will stop the RDS instance db-prod-01...',
        timestamp: Date.now() - 30000
      }
    ];
  }

  async saveMessage(message: Message): Promise<void> {
    console.log('Mock: Saved message', message);
  }

  // Scan commands
  async startScan(config: ScanConfig): Promise<string> {
    const scanId = `scan-${Date.now()}`;
    // Simulate progress events
    setTimeout(() => this.emitScanProgress(scanId, 25), 1000);
    setTimeout(() => this.emitScanProgress(scanId, 50), 2000);
    setTimeout(() => this.emitScanProgress(scanId, 75), 3000);
    setTimeout(() => this.emitScanComplete(scanId), 4000);
    return scanId;
  }

  // Event emitters
  private emitScanProgress(scanId: string, percent: number) {
    window.dispatchEvent(new CustomEvent('scan_progress', {
      detail: {
        scanId,
        progress: { percentComplete: percent },
        currentTask: `Scanning... ${percent}%`
      }
    }));
  }

  private emitScanComplete(scanId: string) {
    window.dispatchEvent(new CustomEvent('scan_complete', {
      detail: { scanId, resourcesFound: 42 }
    }));
  }

  // Listen to events
  onScanProgress(callback: (data: any) => void) {
    const handler = (e: any) => callback(e.detail);
    window.addEventListener('scan_progress', handler);
    return () => window.removeEventListener('scan_progress', handler);
  }
}
```

---

#### 3.2 Mock WebSocket Service
**What to build**:
- [ ] MockWebSocketService
- [ ] Simulate streaming responses
- [ ] Simulate enhancements

**File**: `src/services/websocket/MockWebSocketService.ts`

**Example**:
```typescript
// src/services/websocket/MockWebSocketService.ts
export class MockWebSocketService {
  private handlers: Map<string, Function[]> = new Map();

  send(message: any) {
    console.log('Mock WS: Sending', message);

    if (message.type === 'chat_message') {
      this.simulateStreamingResponse(message.payload.text);
    }
  }

  on(event: string, callback: Function) {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    this.handlers.get(event)!.push(callback);
  }

  private simulateStreamingResponse(query: string) {
    // Phase 1: Stream text response
    const response = `I'll help you with: ${query}`;
    const words = response.split(' ');

    words.forEach((word, i) => {
      setTimeout(() => {
        this.emit('text_chunk', { text: word + ' ' });
      }, i * 100);
    });

    // Phase 2: Send enhancement after streaming
    setTimeout(() => {
      this.emit('enhancement', {
        ui: {
          type: 'playbook',
          data: {
            title: 'Stop RDS Instance',
            steps: [
              { description: 'Verify instance state' },
              { description: 'Stop instance' },
              { description: 'Confirm stopped' }
            ]
          }
        },
        history: {
          role: 'assistant',
          content: response
        }
      });
    }, words.length * 100 + 500);
  }

  private emit(event: string, data: any) {
    const callbacks = this.handlers.get(event) || [];
    callbacks.forEach(cb => cb(data));
  }
}
```

---

#### 3.3 Mock HTTP API Service
**What to build**:
- [ ] MockAPIService
- [ ] Mock recommendations
- [ ] Mock alerts

**File**: `src/services/api/MockAPIService.ts`

**Example**:
```typescript
// src/services/api/MockAPIService.ts
export class MockAPIService {
  async getRecommendations(): Promise<Recommendation[]> {
    return [
      {
        id: 'rec-1',
        title: 'Resize overprovisioned EC2 instances',
        description: 'Save $500/month by downsizing',
        impact: 'high',
        category: 'cost',
        status: 'pending'
      },
      {
        id: 'rec-2',
        title: 'Enable encryption on S3 buckets',
        description: 'Improve security posture',
        impact: 'medium',
        category: 'security',
        status: 'pending'
      }
    ];
  }

  async applyRecommendation(id: string): Promise<void> {
    console.log('Mock: Applying recommendation', id);
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  async getAlerts(): Promise<Alert[]> {
    return [
      {
        id: 'alert-1',
        title: 'Cost spike detected',
        description: 'Your AWS spend increased 30% this week',
        severity: 'warning',
        timestamp: Date.now() - 3600000,
        status: 'unread'
      }
    ];
  }
}
```

---

### **Phase 4: Zustand Stores**

Create state management for all domains.

#### 4.1 Core Stores
**What to build**:
- [ ] useChatStore
- [ ] useEstateStore
- [ ] useScanStore
- [ ] usePlaybookStore
- [ ] useAuthStore
- [ ] useAlertStore (NEW - alerts, unreadCount, alertHistory)
- [ ] useMorningReportStore (NEW - latestReport, Q&A history)
- [ ] useDeploymentStore (NEW - deploymentModel, syncStatus)
- [ ] useCostStore (NEW - costSummary, optimizations, savings)
- [ ] useRecommendationStore

**Example**:
```typescript
// src/models/state/chatStore.ts
import { create } from 'zustand';

interface ChatState {
  contextId: string;
  messages: Message[];
  uiCache: Map<string, any>;
  isStreaming: boolean;

  // Actions
  setContextId: (id: string) => void;
  addMessage: (msg: Message) => void;
  appendToLastMessage: (text: string) => void;
  cacheUI: (msgId: string, ui: any) => void;
  clearCache: () => void;
  reset: () => void;
}

export const useChatStore = create<ChatState>((set) => ({
  contextId: `ctx-${Date.now()}`,
  messages: [],
  uiCache: new Map(),
  isStreaming: false,

  setContextId: (id) => set({ contextId: id }),

  addMessage: (msg) => set((state) => ({
    messages: [...state.messages, msg]
  })),

  appendToLastMessage: (text) => set((state) => {
    const messages = [...state.messages];
    const lastMsg = messages[messages.length - 1];
    if (lastMsg && lastMsg.role === 'assistant') {
      lastMsg.content += text;
    }
    return { messages };
  }),

  cacheUI: (msgId, ui) => set((state) => {
    const cache = new Map(state.uiCache);
    cache.set(msgId, ui);
    return { uiCache: cache };
  }),

  clearCache: () => set({ uiCache: new Map() }),

  reset: () => set({
    messages: [],
    uiCache: new Map(),
    isStreaming: false
  })
}));
```

---

### **Phase 5: Controllers**

Implement business logic layer.

#### 5.1 Core Controllers
**What to build**:
- [ ] ChatController
- [ ] PlaybookController
- [ ] EstateScanController
- [ ] RecommendationController
- [ ] AlertController (NEW - alert acknowledgment, timeline)
- [ ] MorningReportController (NEW - Q&A, report generation)
- [ ] DeploymentController (NEW - deployment model switching)
- [ ] AutoRemediationController (NEW - pre-approved actions)
- [ ] CostController (NEW - cost summary, optimizations)
- [ ] PermissionController

**Example**:
```typescript
// src/controllers/ChatController.ts
import { useChatStore } from '@/models/state/chatStore';
import { MockTauriService } from '@/services/tauri/MockTauriService';
import { MockWebSocketService } from '@/services/websocket/MockWebSocketService';

export class ChatController {
  private tauriService = new MockTauriService();
  private wsService = new MockWebSocketService();

  constructor() {
    this.setupEventListeners();
  }

  private setupEventListeners() {
    this.wsService.on('text_chunk', this.handleTextChunk.bind(this));
    this.wsService.on('enhancement', this.handleEnhancement.bind(this));
  }

  async sendMessage(text: string) {
    const { addMessage, contextId } = useChatStore.getState();

    // Add user message
    addMessage({
      id: `msg-${Date.now()}`,
      role: 'user',
      content: text,
      timestamp: Date.now()
    });

    // Add assistant placeholder
    addMessage({
      id: `msg-${Date.now() + 1}`,
      role: 'assistant',
      content: '',
      timestamp: Date.now()
    });

    // Get history from Rust
    const history = await this.tauriService.getChatHistory(contextId);

    // Send to server
    this.wsService.send({
      type: 'chat_message',
      payload: { text, history }
    });
  }

  private handleTextChunk(data: { text: string }) {
    const { appendToLastMessage } = useChatStore.getState();
    appendToLastMessage(data.text);
  }

  private async handleEnhancement(data: Enhancement) {
    const { cacheUI } = useChatStore.getState();
    const { messages, contextId } = useChatStore.getState();
    const lastMsg = messages[messages.length - 1];

    // BUCKET 1: Cache UI
    if (lastMsg) {
      cacheUI(lastMsg.id, data.ui);
    }

    // BUCKET 2: Save history to Rust
    await this.tauriService.saveMessage({
      contextId,
      ...data.history
    });
  }
}
```

---

### **Phase 6: Views & User Flows**

Build complete feature views.

#### 6.1 Priority Order
1. **Login View** (simplest)
2. **Learning Center** (dashboard with cards)
3. **Estate Scanner** (form + progress)
4. **Ops Chat** (most complex - chat + playbook)
5. **Permissions** (form-heavy)
6. **Recommendations** (list + actions)
7. **Alerts** (list + filters)
8. **Morning Report** (NEW - interactive daily report with Q&A)
9. **Alert History** (NEW - 7-day alert log with conversational search)
10. **Alert Timeline** (NEW - second-by-second incident response)
11. **Cost Dashboard** (NEW - real-time spend, trends, anomalies)
12. **Optimization** (NEW - cost savings with 1-click fixes)
13. **Settings** (NEW - deployment model, auto-remediation, templates)
14. **Deployment Setup Wizard** (NEW - "Extend My Laptop" configuration)
15. **Reports** (charts + tables)

#### 6.2 Example: Ops Chat View
```typescript
// src/views/chat/OpsChatView.tsx
import { useMemo, useEffect } from 'react';
import { useChatStore } from '@/models/state/chatStore';
import { ChatController } from '@/controllers/ChatController';
import { MessageList } from './MessageList';
import { ChatInput } from './ChatInput';

export const OpsChatView = () => {
  const { messages, isStreaming } = useChatStore();
  const controller = useMemo(() => new ChatController(), []);

  const handleSend = (text: string) => {
    controller.sendMessage(text);
  };

  return (
    <div className="flex flex-col h-screen">
      <div className="flex-1 overflow-y-auto p-4">
        <MessageList messages={messages} />
      </div>
      <div className="border-t p-4">
        <ChatInput
          onSend={handleSend}
          disabled={isStreaming}
        />
      </div>
    </div>
  );
};
```

---

### **Phase 7: Routing & Layout**

Wire everything together.

#### 7.1 App Layout
**What to build**:
- [ ] AppLayout component
- [ ] Header with navigation
- [ ] Sidebar with menu
- [ ] Main content area

#### 7.2 Routing
**What to build**:
- [ ] Router configuration
- [ ] Route guards (auth)
- [ ] 404 page

**Example**:
```typescript
// src/routes/index.tsx
import { createBrowserRouter, Navigate } from 'react-router-dom';
import { AppLayout } from '@/views/layouts/AppLayout';
import { OpsChatView } from '@/views/chat/OpsChatView';
import { LoginView } from '@/views/auth/LoginView';
// ... other imports

export const router = createBrowserRouter([
  {
    path: '/login',
    element: <LoginView />
  },
  {
    path: '/',
    element: <AppLayout />,
    children: [
      { index: true, element: <Navigate to="/dashboard" replace /> },
      { path: 'dashboard', element: <LearningCenterView /> },
      { path: 'chat', element: <OpsChatView /> },
      { path: 'scan', element: <EstateScanView /> },
      { path: 'permissions', element: <PermissionsView /> },
      { path: 'recommendations', element: <RecommendationsView /> },
      { path: 'alerts', element: <AlertsView /> },
      { path: 'reports', element: <ReportsView /> }
    ]
  }
]);
```

---

## Deliverables Checklist

### Phase 1: Foundation
- [ ] Project setup complete
- [ ] Folder structure created
- [ ] TypeScript types defined
- [ ] Build system working

### Phase 2: Components
- [ ] 20+ shared components built
- [ ] 10+ UI Agent components built
- [ ] Charts library integrated
- [ ] Component documentation

### Phase 3: Mock Services
- [ ] MockTauriService with 70+ commands
- [ ] MockWebSocketService with streaming
- [ ] MockAPIService with REST endpoints
- [ ] Event simulation working

### Phase 4: State Management
- [ ] 7 Zustand stores created
- [ ] State persistence working
- [ ] Store actions tested

### Phase 5: Controllers
- [ ] 6 controllers implemented
- [ ] All business logic working
- [ ] Mock integration complete

### Phase 6: Views
- [ ] 8 feature views built
- [ ] All user flows working
- [ ] UI polished and responsive

### Phase 7: Integration
- [ ] Routing complete
- [ ] Layout components done
- [ ] App shell working
- [ ] End-to-end flows functional

---

## Success Criteria

By end of implementation, you should have:

✅ **Full working application** with all 8 user flows
✅ **Complete UI** with mock data
✅ **All controllers** with business logic
✅ **Mock service layer** ready to swap with real implementations
✅ **Type-safe codebase** with zero `any` types
✅ **Responsive design** working on all screen sizes
✅ **Unit tests** for critical paths
✅ **Ready for integration** with Platform and Server teams

---

## Integration Readiness

When Platform and Server teams are ready:

### Replace Mock Tauri Service
```typescript
// Before
import { MockTauriService } from '@/services/tauri/MockTauriService';

// After
import { TauriService } from '@/services/tauri/TauriService';
```

### Replace Mock WebSocket
```typescript
// Before
const ws = new MockWebSocketService();

// After
const ws = new WebSocketService('ws://server-url');
```

### Replace Mock API
```typescript
// Before
const api = new MockAPIService();

// After
const api = new APIService('https://api-url');
```

**Zero controller changes needed!** 🎉

---

## Tips for Success

1. **Build incrementally**: Don't try to build everything at once
2. **Test as you go**: Write tests for critical paths
3. **Mock realistic data**: Use data that looks like production
4. **Review regularly**: Check code quality frequently
5. **Document decisions**: Keep notes on why choices were made
6. **Stay aligned**: Sync with Platform/Server teams regularly

Ready to implement! 🚀