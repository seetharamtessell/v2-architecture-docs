# UI Team - Project Structure

Complete folder layout and file organization for the Escher Client frontend.

---

## Complete Directory Structure

```
escher-client/
├── src/
│   ├── models/              # Data layer
│   │   ├── types/          # TypeScript interfaces
│   │   ├── state/          # Zustand stores
│   │   └── transformers/   # Data transformers
│   │
│   ├── views/              # Presentation layer
│   │   ├── auth/          # Authentication views
│   │   ├── chat/          # Ops chat views
│   │   ├── estate/        # Estate/learning center views
│   │   ├── recommendations/ # Recommendations views
│   │   ├── alerts/        # Alerts views
│   │   ├── permissions/   # Permissions views
│   │   ├── reports/       # Reports views
│   │   ├── layouts/       # Layout components
│   │   └── shared/        # Reusable components
│   │
│   ├── controllers/        # Business logic
│   │
│   ├── services/           # Infrastructure
│   │   ├── tauri/        # Tauri IPC
│   │   ├── websocket/    # WebSocket client
│   │   ├── api/          # HTTP API
│   │   └── auth/         # Authentication
│   │
│   ├── routes/             # Routing configuration
│   ├── utils/              # Utility functions
│   ├── hooks/              # Custom React hooks
│   ├── assets/             # Static assets
│   │   ├── images/
│   │   ├── icons/
│   │   └── fonts/
│   │
│   ├── App.tsx             # App root
│   ├── main.tsx            # Entry point
│   └── vite-env.d.ts       # Vite types
│
├── src-tauri/              # Tauri Rust backend
│   ├── src/
│   │   └── main.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
│
├── public/                 # Public static files
├── tests/                  # Test files
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── .vscode/                # VSCode settings
├── node_modules/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
├── postcss.config.js
├── .eslintrc.js
├── .prettierrc
├── .gitignore
└── README.md
```

---

## Detailed Structure by Layer

### **1. Models Layer** (`src/models/`)

#### `types/` - TypeScript Interfaces

```
src/models/types/
├── index.ts                 # Barrel export all types
├── Message.ts               # Chat message types
├── Resource.ts              # AWS resource types
├── Playbook.ts              # Playbook types
├── ScanConfig.ts            # Scanner configuration
├── Policy.ts                # Permission policy types
├── Recommendation.ts        # Recommendation types
├── Alert.ts                 # Alert types
├── User.ts                  # User/auth types
├── Report.ts                # Report types
├── Common.ts                # Shared utility types
└── ApiResponses.ts          # API response types
```

**Example - Message.ts**:
```typescript
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
  tokensUsed?: number;
}

export interface Enhancement {
  ui: UIEnhancement;
  history: MessageHistory;
}

export interface UIEnhancement {
  type: 'playbook' | 'chart' | 'table' | 'card';
  data: any;
}

export interface MessageHistory {
  role: 'assistant';
  content: string;
  metadata?: any;
}
```

#### `state/` - Zustand Stores

```
src/models/state/
├── index.ts                 # Export all stores
├── chatStore.ts             # Chat state management
├── estateStore.ts           # Estate resources state
├── scanStore.ts             # Scanner state
├── playbookStore.ts         # Playbook execution state
├── authStore.ts             # Authentication state
├── alertStore.ts            # Alerts state
├── recommendationStore.ts   # Recommendations state
└── uiStore.ts               # Global UI state (modals, toasts)
```

**Example - chatStore.ts**:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface ChatState {
  contextId: string;
  messages: Message[];
  uiCache: Map<string, UIEnhancement>;
  isStreaming: boolean;
  error: string | null;

  // Actions
  setContextId: (id: string) => void;
  addMessage: (msg: Message) => void;
  updateMessage: (id: string, updates: Partial<Message>) => void;
  appendToLastMessage: (text: string) => void;
  cacheUI: (msgId: string, ui: UIEnhancement) => void;
  clearCache: () => void;
  setStreaming: (streaming: boolean) => void;
  setError: (error: string | null) => void;
  reset: () => void;
}

export const useChatStore = create<ChatState>()(
  persist(
    (set) => ({
      contextId: `ctx-${Date.now()}`,
      messages: [],
      uiCache: new Map(),
      isStreaming: false,
      error: null,

      setContextId: (id) => set({ contextId: id }),

      addMessage: (msg) => set((state) => ({
        messages: [...state.messages, msg]
      })),

      updateMessage: (id, updates) => set((state) => ({
        messages: state.messages.map((msg) =>
          msg.id === id ? { ...msg, ...updates } : msg
        )
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

      setStreaming: (streaming) => set({ isStreaming: streaming }),

      setError: (error) => set({ error }),

      reset: () => set({
        messages: [],
        uiCache: new Map(),
        isStreaming: false,
        error: null
      })
    }),
    {
      name: 'chat-store',
      partialize: (state) => ({ contextId: state.contextId }) // Only persist context ID
    }
  )
);
```

#### `transformers/` - Data Transformers

```
src/models/transformers/
├── index.ts                 # Export all transformers
├── resourceTransformer.ts   # API ↔ UI resource format
├── messageTransformer.ts    # WebSocket ↔ UI message format
├── playbookTransformer.ts   # Server ↔ UI playbook format
└── dateTransformer.ts       # Date formatting utilities
```

---

### **2. Views Layer** (`src/views/`)

#### `auth/` - Authentication Views

```
src/views/auth/
├── LoginView.tsx            # Login page
├── SignupView.tsx           # Signup page
├── ForgotPasswordView.tsx   # Password reset
├── LoginForm.tsx            # Login form component
└── AuthLayout.tsx           # Auth pages layout
```

#### `chat/` - Ops Chat Views

```
src/views/chat/
├── OpsChatView.tsx          # Main chat interface
├── MessageList.tsx          # Message list container
├── MessageItem.tsx          # Individual message
├── ChatInput.tsx            # Message input box
├── PlaybookDisplay.tsx      # Playbook rendering in chat
├── ExecutionDisplay.tsx     # Execution progress in chat
└── ChatSidebar.tsx          # Chat history sidebar
```

#### `estate/` - Estate & Learning Center

```
src/views/estate/
├── LearningCenterView.tsx   # Dashboard/learning center
├── EstateOverview.tsx       # Overview cards
├── ResourceExplorer.tsx     # Browse resources
├── ResourceDetail.tsx       # Single resource view
└── EstateStats.tsx          # Statistics display
```

#### `scan/` - Estate Scanner

```
src/views/scan/
├── EstateScanView.tsx       # Scanner page
├── ScanConfigForm.tsx       # Configuration form
├── ScanProgress.tsx         # Progress display
├── ScanResults.tsx          # Results summary
└── ScanHistory.tsx          # Previous scans
```

#### `recommendations/` - Recommendations

```
src/views/recommendations/
├── RecommendationsView.tsx  # Main recommendations page
├── RecommendationCard.tsx   # Individual recommendation
├── RecommendationList.tsx   # List container
├── RecommendationDetail.tsx # Detailed view
└── RecommendationFilters.tsx # Filter controls
```

#### `alerts/` - Alerts & Notifications

```
src/views/alerts/
├── AlertsView.tsx           # Alerts page
├── AlertCard.tsx            # Individual alert
├── AlertList.tsx            # List container
├── AlertDetail.tsx          # Detailed view
└── AlertFilters.tsx         # Filter controls
```

#### `permissions/` - Permissions Management

```
src/views/permissions/
├── PermissionsView.tsx      # Main permissions page
├── PolicyEditor.tsx         # Policy editing form
├── OperationSelector.tsx    # Select allowed/blocked ops
├── ResourceConstraints.tsx  # Resource constraints form
└── PolicyPreview.tsx        # Preview policy JSON
```

#### `reports/` - Reports

```
src/views/reports/
├── ReportsView.tsx          # Reports page
├── CostReport.tsx           # Cost analysis
├── SecurityReport.tsx       # Security analysis
├── ComplianceReport.tsx     # Compliance status
└── CustomReport.tsx         # Custom report builder
```

#### `layouts/` - Layout Components

```
src/views/layouts/
├── AppLayout.tsx            # Main app layout
├── Header.tsx               # Top navigation
├── Sidebar.tsx              # Side navigation
├── Footer.tsx               # Footer
└── AuthLayout.tsx           # Authentication layout
```

#### `shared/` - Reusable Components

```
src/views/shared/
├── index.ts                 # Export all shared components
│
├── basic/                   # Basic components
│   ├── Button.tsx
│   ├── Card.tsx
│   ├── Badge.tsx
│   ├── Tag.tsx
│   ├── Spinner.tsx
│   ├── ProgressBar.tsx
│   └── Divider.tsx
│
├── forms/                   # Form components
│   ├── Input.tsx
│   ├── TextArea.tsx
│   ├── Select.tsx
│   ├── Checkbox.tsx
│   ├── Radio.tsx
│   ├── DatePicker.tsx
│   ├── FileUpload.tsx
│   └── FormField.tsx
│
├── feedback/                # Feedback components
│   ├── Toast.tsx
│   ├── Modal.tsx
│   ├── Drawer.tsx
│   ├── Alert.tsx
│   ├── Tooltip.tsx
│   └── Popover.tsx
│
├── charts/                  # Chart components
│   ├── LineChart.tsx
│   ├── BarChart.tsx
│   ├── PieChart.tsx
│   ├── AreaChart.tsx
│   ├── RadarChart.tsx
│   └── ScatterChart.tsx
│
├── data-display/            # Data display components
│   ├── Table.tsx
│   ├── List.tsx
│   ├── Tree.tsx
│   ├── KeyValuePairs.tsx
│   ├── Timeline.tsx
│   └── Accordion.tsx
│
└── ui-agent/                # UI Agent components
    ├── PlaybookCard.tsx
    ├── ExecutionProgress.tsx
    ├── StepIndicator.tsx
    ├── CommandOutput.tsx
    ├── ResourceCard.tsx
    ├── ResourceTable.tsx
    ├── ResourceTree.tsx
    ├── DynamicRenderer.tsx
    └── ComponentRegistry.ts
```

---

### **3. Controllers Layer** (`src/controllers/`)

```
src/controllers/
├── index.ts                 # Export all controllers
├── ChatController.ts        # Chat business logic
├── PlaybookController.ts    # Playbook execution logic
├── EstateScanController.ts  # Scanner logic
├── RecommendationController.ts # Recommendations logic
├── AlertController.ts       # Alerts logic
├── PermissionController.ts  # Permissions logic
├── AuthController.ts        # Authentication logic
└── ReportController.ts      # Reports logic
```

**Example - ChatController.ts**:
```typescript
import { useChatStore } from '@/models/state/chatStore';
import { TauriService } from '@/services/tauri/TauriService';
import { WebSocketService } from '@/services/websocket/WebSocketService';

export class ChatController {
  private tauriService: TauriService;
  private wsService: WebSocketService;

  constructor() {
    this.tauriService = new TauriService();
    this.wsService = new WebSocketService();
    this.setupEventListeners();
  }

  private setupEventListeners() {
    this.wsService.on('text_chunk', this.handleTextChunk.bind(this));
    this.wsService.on('enhancement', this.handleEnhancement.bind(this));
  }

  async sendMessage(text: string) {
    const { addMessage, setStreaming, contextId } = useChatStore.getState();

    // Add user message
    const userMsg: Message = {
      id: `msg-${Date.now()}`,
      role: 'user',
      content: text,
      timestamp: Date.now()
    };
    addMessage(userMsg);

    // Add assistant placeholder
    const assistantMsg: Message = {
      id: `msg-${Date.now() + 1}`,
      role: 'assistant',
      content: '',
      timestamp: Date.now()
    };
    addMessage(assistantMsg);

    setStreaming(true);

    try {
      // Get history from Rust
      const history = await this.tauriService.getChatHistory(contextId);

      // Send to server
      this.wsService.send({
        type: 'chat_message',
        payload: { text, history }
      });
    } catch (error) {
      console.error('Failed to send message:', error);
      setStreaming(false);
    }
  }

  private handleTextChunk(data: { text: string }) {
    const { appendToLastMessage } = useChatStore.getState();
    appendToLastMessage(data.text);
  }

  private async handleEnhancement(data: Enhancement) {
    const { cacheUI, setStreaming, messages, contextId } = useChatStore.getState();
    const lastMsg = messages[messages.length - 1];

    if (lastMsg) {
      // BUCKET 1: Cache UI
      cacheUI(lastMsg.id, data.ui);

      // BUCKET 2: Save history to Rust
      await this.tauriService.saveMessage({
        contextId,
        ...data.history
      });
    }

    setStreaming(false);
  }

  destroy() {
    this.wsService.disconnect();
  }
}
```

---

### **4. Services Layer** (`src/services/`)

#### `tauri/` - Tauri IPC Service

```
src/services/tauri/
├── index.ts                 # Export services
├── TauriService.ts          # Main Tauri service
├── MockTauriService.ts      # Mock for development
├── StorageCommands.ts       # Storage command wrappers
├── ExecutionCommands.ts     # Execution command wrappers
├── ScannerCommands.ts       # Scanner command wrappers
├── AuthCommands.ts          # Auth command wrappers
└── types.ts                 # Tauri-specific types
```

#### `websocket/` - WebSocket Service

```
src/services/websocket/
├── index.ts
├── WebSocketService.ts      # Real WebSocket client
└── MockWebSocketService.ts  # Mock for development
```

#### `api/` - HTTP API Service

```
src/services/api/
├── index.ts
├── APIService.ts            # Real HTTP client
├── MockAPIService.ts        # Mock for development
└── endpoints.ts             # API endpoint definitions
```

#### `auth/` - Authentication Service

```
src/services/auth/
├── index.ts
├── AuthService.ts           # Cognito integration
└── MockAuthService.ts       # Mock for development
```

---

### **5. Routes Layer** (`src/routes/`)

```
src/routes/
├── index.tsx                # Router configuration
├── PrivateRoute.tsx         # Protected route wrapper
└── routes.ts                # Route definitions
```

---

### **6. Utilities** (`src/utils/`)

```
src/utils/
├── index.ts                 # Export all utilities
├── dateUtils.ts             # Date formatting
├── stringUtils.ts           # String manipulation
├── validators.ts            # Form validation
├── formatters.ts            # Data formatting
├── constants.ts             # App constants
└── errorHandlers.ts         # Error handling utilities
```

---

### **7. Custom Hooks** (`src/hooks/`)

```
src/hooks/
├── index.ts                 # Export all hooks
├── useController.ts         # Controller lifecycle hook
├── useTauriEvent.ts         # Tauri event listener hook
├── useDebounce.ts           # Debounce hook
├── useLocalStorage.ts       # Local storage hook
└── useWebSocket.ts          # WebSocket connection hook
```

**Example - useTauriEvent.ts**:
```typescript
import { useEffect } from 'react';
import { listen } from '@tauri-apps/api/event';

export function useTauriEvent<T>(
  eventName: string,
  callback: (payload: T) => void
) {
  useEffect(() => {
    let unlisten: (() => void) | undefined;

    listen(eventName, (event) => {
      callback(event.payload as T);
    }).then((unlistenFn) => {
      unlisten = unlistenFn;
    });

    return () => {
      if (unlisten) {
        unlisten();
      }
    };
  }, [eventName, callback]);
}
```

---

### **8. Assets** (`src/assets/`)

```
src/assets/
├── images/
│   ├── logo.svg
│   ├── logo-dark.svg
│   └── placeholder.png
│
├── icons/
│   ├── aws-services/        # AWS service icons
│   ├── actions/             # Action icons
│   └── status/              # Status icons
│
└── fonts/
    └── inter/               # Inter font family
```

---

## Configuration Files

### `package.json`
```json
{
  "name": "escher-client",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "tauri": "tauri",
    "tauri:dev": "tauri dev",
    "tauri:build": "tauri build",
    "lint": "eslint src --ext ts,tsx",
    "format": "prettier --write \"src/**/*.{ts,tsx}\"",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "zustand": "^4.4.7",
    "recharts": "^2.10.0",
    "@tauri-apps/api": "^2.0.0",
    "clsx": "^2.0.0",
    "date-fns": "^2.30.0"
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2.0.0",
    "@types/react": "^18.2.43",
    "@types/react-dom": "^18.2.17",
    "@types/node": "^20.10.4",
    "@vitejs/plugin-react": "^4.2.1",
    "typescript": "^5.3.3",
    "vite": "^5.0.8",
    "tailwindcss": "^3.3.6",
    "postcss": "^8.4.32",
    "autoprefixer": "^10.4.16",
    "eslint": "^8.55.0",
    "prettier": "^3.1.1",
    "vitest": "^1.0.4"
  }
}
```

### `tsconfig.json`
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,

    /* Path aliases */
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/models/*": ["src/models/*"],
      "@/views/*": ["src/views/*"],
      "@/controllers/*": ["src/controllers/*"],
      "@/services/*": ["src/services/*"],
      "@/utils/*": ["src/utils/*"],
      "@/hooks/*": ["src/hooks/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### `vite.config.ts`
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  clearScreen: false,
  server: {
    port: 1420,
    strictPort: true,
  },
  envPrefix: ['VITE_', 'TAURI_'],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@/models': path.resolve(__dirname, './src/models'),
      '@/views': path.resolve(__dirname, './src/views'),
      '@/controllers': path.resolve(__dirname, './src/controllers'),
      '@/services': path.resolve(__dirname, './src/services'),
      '@/utils': path.resolve(__dirname, './src/utils'),
      '@/hooks': path.resolve(__dirname, './src/hooks'),
    },
  },
  build: {
    target: 'esnext',
    minify: 'esbuild',
    sourcemap: true,
  },
});
```

### `tailwind.config.js`
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
        },
      },
    },
  },
  plugins: [],
}
```

---

## File Naming Conventions

1. **Components**: PascalCase
   - `OpsChatView.tsx`
   - `MessageList.tsx`
   - `Button.tsx`

2. **Utilities**: camelCase
   - `dateUtils.ts`
   - `validators.ts`

3. **Stores**: camelCase with "Store" suffix
   - `chatStore.ts`
   - `authStore.ts`

4. **Controllers**: PascalCase with "Controller" suffix
   - `ChatController.ts`
   - `PlaybookController.ts`

5. **Services**: PascalCase with "Service" suffix
   - `TauriService.ts`
   - `WebSocketService.ts`

6. **Types**: PascalCase
   - `Message.ts`
   - `Resource.ts`

---

## Import Patterns

### Use Path Aliases
```typescript
// Good
import { Message } from '@/models/types';
import { useChatStore } from '@/models/state/chatStore';
import { Button } from '@/views/shared/basic/Button';

// Avoid
import { Message } from '../../models/types';
import { useChatStore } from '../../models/state/chatStore';
```

### Barrel Exports
```typescript
// src/models/types/index.ts
export * from './Message';
export * from './Resource';
export * from './Playbook';

// Usage
import { Message, Resource, Playbook } from '@/models/types';
```

---

## Summary

This structure provides:

✅ **Clear separation** of concerns (MVC)
✅ **Scalable architecture** (easy to add features)
✅ **Type-safe** (TypeScript throughout)
✅ **Reusable components** (shared folder)
✅ **Mock-ready** (easy to swap implementations)
✅ **Test-friendly** (isolated layers)

Ready to start building! 🚀