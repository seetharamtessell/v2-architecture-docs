# UI Rendering Engine

**Location**: Client-side (React frontend)
**Purpose**: Receives structured JSON from server-side UI Agent and dynamically renders React components
**Status**: Design Complete

---

## Overview

The **UI Rendering Engine** is a **client-side service** that acts as the bridge between the server-side **UI Agent** and the **UI Components** library. It's responsible for:

1. Receiving enhancement messages from server
2. Parsing and validating UI specifications
3. Mapping component types to React components
4. Managing dual storage (UI cache + history)
5. Rendering components dynamically

### Architecture Context

```
Server-Side UI Agent (docs/03-server/agents/ui-agent.md)
    ↓ WebSocket
    ↓ (structured JSON with UI specs)
UI Rendering Engine (THIS SERVICE)
    ↓ (maps type → component)
UI Components Package (ui-components.md)
    ↓ (renders React UI)
Chat Display
```

---

## Core Responsibilities

### 1. Message Reception
- Listen to WebSocket messages from server
- Handle two message types:
  - **Stream chunks**: Text that accumulates
  - **Enhancement messages**: Rich UI specifications

### 2. Response Parsing
- Parse enhancement JSON structure
- Extract UI mode (full component data)
- Extract history mode (compact digest)
- Validate against expected schema

### 3. Component Mapping
- Look up component type in registry
- Map server type string to React component
- Handle unknown types with fallback
- Pass props and data to components

### 4. Dual Storage Management
**Critical**: Enhancement must be split into TWO separate buckets

**Bucket 1 - UI Cache (IndexedDB)**:
- Store: `enhancement.ui` (full component data, 2-10 KB)
- Purpose: Display rich UI components
- Usage: Rendering only
- **NEVER send back to server**

**Bucket 2 - Conversation History (Memory)**:
- Store: `enhancement.history` (compact digest, 200-500 bytes)
- Purpose: LLM context for follow-up queries
- Usage: Send with every request
- **NEVER include enhancement.ui** (20x too large!)

### 5. Dynamic Rendering
- Instantiate React components from registry
- Pass props, data, and callbacks
- Handle component lifecycle
- Manage error boundaries

---

## Two-Phase Response Pattern

### Phase 1: Streaming (Immediate Display)

```typescript
// Server streams text
websocket.on('message', (chunk) => {
  if (chunk.type === 'stream') {
    // APPEND chunk to message content
    const message = messages.find(m => m.id === chunk.message_id);
    message.content += chunk.content;

    // User sees text immediately (0ms wait)
    setMessages([...messages]);
  }
});
```

### Phase 2: Enhancement (Background Processing)

```typescript
// Server sends enhancement (500ms-2s later)
websocket.on('message', (enhancement) => {
  if (enhancement.type === 'enhance') {

    // SPLIT into two buckets immediately

    // Bucket 1: UI Cache (for display only)
    uiCache.set(enhancement.message_id, enhancement.ui);

    // Bucket 2: History (for LLM context)
    conversationHistory.push(enhancement.history);  // Digest only!

    // Update message to enhanced mode
    const message = messages.find(m => m.id === enhancement.message_id);
    message.type = 'enhanced';
    message.ui = enhancement.ui;

    // Smooth fade transition to rich UI
    setMessages([...messages]);
  }
});
```

---

## Component Registry

### Registry Structure

```typescript
interface ComponentRegistry {
  // Map type strings to React components
  components: Map<string, React.ComponentType>;

  // Register component
  register(type: string, component: React.ComponentType): void;

  // Get component (with fallback)
  get(type: string): React.ComponentType;

  // Check if type exists
  has(type: string): boolean;
}
```

### Default Registry (30+ Components)

```typescript
const componentRegistry = new Map<string, React.ComponentType>([
  // Data Display
  ['metric', MetricComponent],
  ['stat_grid', StatGridComponent],
  ['chart', ChartComponent],
  ['table', TableComponent],
  ['list', ListComponent],

  // Operation Components
  ['confirmation_card', ConfirmationCardComponent],
  ['execution_plan', ExecutionPlanComponent],
  ['progress_tracker', ProgressTrackerComponent],
  ['execution_result', ExecutionResultComponent],

  // Resource Components
  ['resource_card', ResourceCardComponent],
  ['resource_table', ResourceTableComponent],
  ['resource_detail', ResourceDetailComponent],

  // Interactive Components
  ['button_group', ButtonGroupComponent],
  ['form', FormComponent],
  ['filter_panel', FilterPanelComponent],

  // Alert Components
  ['alert_card', AlertCardComponent],
  ['error_card', ErrorCardComponent],
  ['warning_card', WarningCardComponent],
  ['success_card', SuccessCardComponent],

  // Visualization Components
  ['timeline', TimelineComponent],
  ['heatmap', HeatmapComponent],
  ['network_diagram', NetworkDiagramComponent],

  // ... 30+ total components
]);
```

---

## Storage Pattern

### Why Two Separate Buckets?

**Problem**: If we store full UI data in conversation history, context explodes

```
WITHOUT SEPARATION:
Turn 1: 5 KB (UI data in history)
Turn 2: 5 KB
Turn 3: 5 KB
After 50 turns: 250 KB → Expensive, slow, token limit hit
```

```
WITH SEPARATION:
Turn 1: 300 bytes (digest in history)
Turn 2: 300 bytes
Turn 3: 300 bytes
After 50 turns: 15 KB → Fast, cheap, efficient
```

### Storage Implementation

```typescript
// ✅ CORRECT PATTERN

// 1. UI Cache (IndexedDB) - Display only
interface UICache {
  messageId: string;
  ui: UIMode;  // Full component data (5 KB)
  timestamp: number;
}

const uiCache = new Map<string, UIMode>();

// Store enhancement UI
function storeEnhancementUI(messageId: string, ui: UIMode) {
  uiCache.set(messageId, ui);
  // Also persist to IndexedDB for offline access
  await indexedDB.put('ui_cache', { messageId, ui });
}

// 2. Conversation History (Memory) - LLM context
interface HistoryEntry {
  role: 'user' | 'assistant';
  content: string;  // Compact digest (300 bytes)
  metadata?: {
    entities: string[];
    action: string;
    values?: Record<string, any>;
  };
}

const conversationHistory: HistoryEntry[] = [];

// Store enhancement history (digest only!)
function storeEnhancementHistory(history: HistoryEntry) {
  conversationHistory.push(history);  // Append to array

  // Keep last 50 entries
  if (conversationHistory.length > 50) {
    conversationHistory = conversationHistory.slice(-50);
  }
}

// When sending next request
function sendMessage(content: string) {
  websocket.send({
    content: content,
    context: {
      history: conversationHistory  // Send digests only!
    }
  });
}
```

---

## Rendering Flow

### Complete Flow Example

```typescript
// UI Rendering Engine Service
class UIRenderingEngine {
  private componentRegistry: ComponentRegistry;
  private uiCache: Map<string, UIMode>;
  private conversationHistory: HistoryEntry[];

  // Handle enhancement message
  async handleEnhancement(enhancement: Enhancement): Promise<void> {
    try {
      // 1. Parse and validate
      this.validateEnhancement(enhancement);

      // 2. Split into two buckets
      await this.storeUI(enhancement.message_id, enhancement.ui);
      this.storeHistory(enhancement.history);

      // 3. Update message display
      this.updateMessageToEnhanced(enhancement.message_id, enhancement.ui);

    } catch (error) {
      console.error('Enhancement rendering failed:', error);
      this.showFallback(enhancement.message_id, error);
    }
  }

  // Store UI in cache (display only)
  private async storeUI(messageId: string, ui: UIMode): Promise<void> {
    this.uiCache.set(messageId, ui);
    await indexedDB.put('ui_cache', { messageId, ui, timestamp: Date.now() });
  }

  // Store history digest (for LLM)
  private storeHistory(history: HistoryEntry): void {
    this.conversationHistory.push(history);

    // Keep last 50 entries
    if (this.conversationHistory.length > 50) {
      this.conversationHistory = this.conversationHistory.slice(-50);
    }
  }

  // Render component dynamically
  renderComponent(spec: ComponentSpec): ReactElement {
    // Look up component in registry
    const Component = this.componentRegistry.get(spec.type);

    if (!Component) {
      // Fallback for unknown types
      return <FallbackComponent spec={spec} />;
    }

    // Render with props and data
    return (
      <ErrorBoundary>
        <Component
          {...spec.props}
          data={spec.data}
          onAction={(action) => this.handleAction(spec.id, action)}
        />
      </ErrorBoundary>
    );
  }
}
```

---

## Message State Management

### Message Structure

```typescript
interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;  // Streamed text
  type: 'text' | 'enhanced';
  ui?: UIMode;  // Only present for enhanced messages
  timestamp: number;
}
```

### State Updates

```typescript
// 1. User message
const userMessage: Message = {
  id: generateId(),
  role: 'user',
  content: 'Show me December costs',
  type: 'text',
  timestamp: Date.now()
};
messages.push(userMessage);  // APPEND

// 2. Assistant message (streaming)
const assistantMessage: Message = {
  id: generateId(),
  role: 'assistant',
  content: '',  // Empty initially
  type: 'text',
  timestamp: Date.now()
};
messages.push(assistantMessage);  // APPEND

// 3. Stream chunks (APPEND to content)
onStreamChunk(chunk => {
  assistantMessage.content += chunk.content;  // Accumulate
  setMessages([...messages]);  // Re-render
});

// 4. Enhancement arrives (REPLACE type, ADD ui)
onEnhancement(enhancement => {
  assistantMessage.type = 'enhanced';  // Change type
  assistantMessage.ui = enhancement.ui;  // Add UI data

  // Store in cache
  uiCache.set(assistantMessage.id, enhancement.ui);

  // Store in history (digest only!)
  conversationHistory.push(enhancement.history);

  setMessages([...messages]);  // Re-render with rich UI
});
```

---

## Component Rendering

### Dynamic Renderer

```typescript
interface DynamicRendererProps {
  components: ComponentSpec[];
  messageId: string;
  onAction: (componentId: string, action: Action) => void;
}

const DynamicRenderer: React.FC<DynamicRendererProps> = ({
  components,
  messageId,
  onAction
}) => {
  return (
    <div className="enhanced-message">
      {components.map((spec) => (
        <ErrorBoundary key={spec.id}>
          {renderComponent(spec, messageId, onAction)}
        </ErrorBoundary>
      ))}
    </div>
  );
};

function renderComponent(
  spec: ComponentSpec,
  messageId: string,
  onAction: (componentId: string, action: Action) => void
): ReactElement {
  // Get component from registry
  const Component = componentRegistry.get(spec.type);

  if (!Component) {
    // Fallback for unknown types
    return (
      <div className="fallback-component">
        <h4>Unknown component type: {spec.type}</h4>
        <pre>{JSON.stringify(spec, null, 2)}</pre>
      </div>
    );
  }

  // Render component
  return (
    <Component
      {...spec.props}
      data={spec.data}
      layout={spec.layout}
      onAction={(action) => onAction(spec.id, action)}
    />
  );
}
```

---

## Error Handling

### Error Boundaries

```typescript
class EnhancementErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Component rendering error:', error, errorInfo);
    // Log to analytics/monitoring
  }

  render() {
    if (this.state.hasError) {
      // Fallback UI
      return (
        <div className="render-error">
          <h4>Failed to render component</h4>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false })}>
            Retry
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Validation

```typescript
function validateEnhancement(enhancement: Enhancement): void {
  // Check required fields
  if (!enhancement.type || enhancement.type !== 'enhance') {
    throw new Error('Invalid enhancement type');
  }

  if (!enhancement.message_id) {
    throw new Error('Missing message_id');
  }

  if (!enhancement.ui || !enhancement.ui.components) {
    throw new Error('Missing UI components');
  }

  if (!enhancement.history || !enhancement.history.content) {
    throw new Error('Missing history digest');
  }

  // Validate each component
  enhancement.ui.components.forEach((component, index) => {
    if (!component.type) {
      throw new Error(`Component ${index} missing type`);
    }

    if (!component.data) {
      throw new Error(`Component ${index} missing data`);
    }
  });
}
```

---

## Performance Optimization

### 1. Component Memoization

```typescript
const MemoizedComponent = React.memo(
  Component,
  (prevProps, nextProps) => {
    // Only re-render if data actually changed
    return prevProps.data === nextProps.data &&
           prevProps.props === nextProps.props;
  }
);
```

### 2. Lazy Loading

```typescript
// Lazy load heavy components
const ChartComponent = React.lazy(() => import('./components/ChartComponent'));
const NetworkDiagram = React.lazy(() => import('./components/NetworkDiagram'));

// Render with Suspense
<Suspense fallback={<Spinner />}>
  <ChartComponent data={chartData} />
</Suspense>
```

### 3. Virtual Scrolling

```typescript
// For large message lists
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={messages.length}
  itemSize={100}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      <MessageItem message={messages[index]} />
    </div>
  )}
</FixedSizeList>
```

---

## Integration with Chat UI

### Message Item Component

```typescript
interface MessageItemProps {
  message: Message;
  onAction: (messageId: string, action: Action) => void;
}

const MessageItem: React.FC<MessageItemProps> = ({ message, onAction }) => {
  // Text message (streaming or complete)
  if (message.type === 'text') {
    return (
      <div className="message-text">
        <Markdown content={message.content} />
      </div>
    );
  }

  // Enhanced message (rich UI)
  if (message.type === 'enhanced' && message.ui) {
    return (
      <div className="message-enhanced">
        {/* Show original text (collapsible) */}
        <details>
          <summary>Show text response</summary>
          <Markdown content={message.content} />
        </details>

        {/* Render rich UI components */}
        <DynamicRenderer
          components={message.ui.components}
          messageId={message.id}
          onAction={(componentId, action) =>
            onAction(message.id, { componentId, ...action })
          }
        />
      </div>
    );
  }

  return null;
};
```

---

## Summary

The **UI Rendering Engine** is the **client-side orchestrator** that:

✅ **Receives** structured JSON from server-side UI Agent
✅ **Parses** enhancement messages (UI mode + history digest)
✅ **Splits** into two buckets (UI cache for display, history for LLM)
✅ **Maps** component types to React components (30+ components)
✅ **Renders** dynamic UI with error boundaries
✅ **Manages** conversation context efficiently (20x smaller digests)

**Key Principle**: Keep it simple! The intelligence is in the server-side UI Agent. This is just a lightweight mapper that connects server specs to React components.

**Result**: Truly dynamic, scalable UI where server controls presentation and frontend remains clean and maintainable.

---

**Related Documentation**:
- [Server-Side UI Agent](../../03-server/agents/ui-agent.md) - Generates UI specifications
- [UI Components Package](./ui-components.md) - 30+ React presentation components
- [User Flows](./user-flows.md) - End-to-end user interaction flows