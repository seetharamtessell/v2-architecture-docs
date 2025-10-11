# Claude Code Prompts for UI Development

Ready-to-use prompts to accelerate development with Claude Code.

**How to use**: Copy the prompt, paste into Claude Code, and let it generate the code.

---

## Table of Contents

1. [Project Setup Prompts](#project-setup-prompts)
2. [Component Generation Prompts](#component-generation-prompts)
3. [Store Creation Prompts](#store-creation-prompts)
4. [Controller Implementation Prompts](#controller-implementation-prompts)
5. [View Building Prompts](#view-building-prompts)
6. [Service Layer Prompts](#service-layer-prompts)

---

## Project Setup Prompts

### 1. Initialize Tauri Project

```
Create a new Tauri + React + TypeScript project for the Escher Client.

Setup requirements:
- Tauri 2.0
- React 18 with TypeScript
- Vite as build tool
- Tailwind CSS for styling
- Zustand for state management
- React Router for routing
- Recharts for data visualization

Configure:
- TypeScript with strict mode
- Path aliases (@/ pointing to src/)
- ESLint and Prettier
- Tailwind with custom color palette (primary blue)

Project structure should follow MVC pattern:
- src/models/ (types, state, transformers)
- src/views/ (presentation components)
- src/controllers/ (business logic)
- src/services/ (infrastructure)
```

### 2. Setup Folder Structure

```
Create the complete MVC folder structure for the Escher Client based on this layout:

src/
├── models/
│   ├── types/
│   ├── state/
│   └── transformers/
├── views/
│   ├── auth/
│   ├── chat/
│   ├── estate/
│   ├── scan/
│   ├── recommendations/
│   ├── alerts/
│   ├── permissions/
│   ├── reports/
│   ├── layouts/
│   └── shared/
│       ├── basic/
│       ├── forms/
│       ├── feedback/
│       ├── charts/
│       ├── data-display/
│       └── ui-agent/
├── controllers/
├── services/
│   ├── tauri/
│   ├── websocket/
│   ├── api/
│   └── auth/
├── routes/
├── utils/
├── hooks/
└── assets/

Create index.ts barrel exports for each major folder.
```

---

## Component Generation Prompts

### 3. Create Basic Button Component

```
Create a reusable Button component in src/views/shared/basic/Button.tsx

Requirements:
- TypeScript with proper props interface
- Tailwind CSS styling
- Support variants: primary, secondary, danger, ghost
- Support sizes: sm, md, lg
- Support disabled state
- Support loading state with spinner
- Support icon (left or right)
- Accessible (proper ARIA attributes)

Example usage:
<Button variant="primary" size="md" onClick={handleClick}>
  Click Me
</Button>

<Button variant="danger" size="lg" loading>
  Processing...
</Button>

<Button variant="ghost" icon={<TrashIcon />} iconPosition="left">
  Delete
</Button>
```

### 4. Create Card Component

```
Create a reusable Card component in src/views/shared/basic/Card.tsx

Requirements:
- TypeScript
- Tailwind CSS
- Composable sub-components: CardHeader, CardBody, CardFooter
- Support hoverable prop (lift effect on hover)
- Support clickable prop (cursor pointer)
- Support bordered prop
- Responsive padding

Example usage:
<Card hoverable>
  <CardHeader>
    <h3>Title</h3>
    <Badge>New</Badge>
  </CardHeader>
  <CardBody>
    Content goes here
  </CardBody>
  <CardFooter>
    <Button>Action</Button>
  </CardFooter>
</Card>
```

### 5. Create Modal Component

```
Create a Modal component in src/views/shared/feedback/Modal.tsx

Requirements:
- TypeScript
- Tailwind CSS
- Backdrop with blur effect
- Close on escape key
- Close on backdrop click (optional)
- Animation (fade in/out)
- Sizes: sm, md, lg, full
- Scrollable content
- Header with close button
- Footer with action buttons

Use React portals to render outside DOM hierarchy.

Example usage:
<Modal
  isOpen={isOpen}
  onClose={handleClose}
  size="md"
  title="Confirm Action"
>
  <p>Are you sure you want to proceed?</p>
  <ModalFooter>
    <Button variant="secondary" onClick={handleClose}>Cancel</Button>
    <Button variant="primary" onClick={handleConfirm}>Confirm</Button>
  </ModalFooter>
</Modal>
```

### 6. Create PlaybookCard Component

```
Create a PlaybookCard UI Agent component in src/views/shared/ui-agent/PlaybookCard.tsx

Requirements:
- TypeScript with Playbook interface
- Display playbook title, summary, risk level
- Show step count
- Risk level badge (low=green, medium=yellow, high=red)
- Two action buttons: "Explain Plan" and "Execute"
- Estimated duration display
- Expandable step list
- Tailwind CSS styling

Interface:
interface Playbook {
  id: string;
  title: string;
  summary: string;
  riskLevel: 'low' | 'medium' | 'high';
  estimatedDuration: number; // seconds
  steps: Array<{
    description: string;
    command: string;
    estimatedTime: string;
  }>;
}

Example usage:
<PlaybookCard
  playbook={playbook}
  onExplain={() => setShowExplanation(true)}
  onExecute={() => controller.executePlaybook(playbook)}
/>
```

### 7. Create LineChart Component

```
Create a LineChart component in src/views/shared/charts/LineChart.tsx using Recharts

Requirements:
- TypeScript with proper data interface
- Responsive width and height
- Customizable colors
- Tooltip on hover
- Grid lines
- X and Y axis labels
- Legend
- Support multiple data series

Example usage:
<LineChart
  data={[
    { date: '2024-01', value: 100 },
    { date: '2024-02', value: 150 },
    { date: '2024-03', value: 120 }
  ]}
  xKey="date"
  yKey="value"
  color="#3b82f6"
  title="Cost Trend"
/>
```

---

## Store Creation Prompts

### 8. Create Chat Store

```
Create a Zustand store for chat state in src/models/state/chatStore.ts

Requirements:
- TypeScript with proper interfaces
- State:
  - contextId (string)
  - messages (Message[])
  - uiCache (Map<string, UIEnhancement>)
  - isStreaming (boolean)
  - error (string | null)

- Actions:
  - setContextId(id: string)
  - addMessage(msg: Message)
  - updateMessage(id: string, updates: Partial<Message>)
  - appendToLastMessage(text: string)
  - cacheUI(msgId: string, ui: UIEnhancement)
  - clearCache()
  - setStreaming(streaming: boolean)
  - setError(error: string | null)
  - reset()

- Use persist middleware to save contextId only
- Export useChatStore hook

Message interface:
interface Message {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
  metadata?: any;
}
```

### 9. Create Estate Store

```
Create a Zustand store for estate resources in src/models/state/estateStore.ts

Requirements:
- TypeScript
- State:
  - resources (Resource[])
  - filters (ResourceFilters)
  - selectedResource (Resource | null)
  - isLoading (boolean)
  - error (string | null)

- Actions:
  - setResources(resources: Resource[])
  - addResource(resource: Resource)
  - updateResource(id: string, updates: Partial<Resource>)
  - deleteResource(id: string)
  - setFilters(filters: ResourceFilters)
  - selectResource(resource: Resource | null)
  - setLoading(loading: boolean)
  - setError(error: string | null)
  - reset()

Resource interface:
interface Resource {
  id: string;
  type: string;
  name: string;
  region: string;
  accountId: string;
  state: string;
  tags: Record<string, string>;
  metadata?: any;
}
```

### 10. Create Scan Store

```
Create a Zustand store for scanner state in src/models/state/scanStore.ts

Requirements:
- State:
  - currentScanId (string | null)
  - scanProgress (ScanProgress | null)
  - scanResults (ScanResults | null)
  - scanStatus ('idle' | 'running' | 'completed' | 'failed' | 'cancelled')
  - error (string | null)

- Actions:
  - startScan(scanId: string)
  - updateProgress(progress: ScanProgress)
  - completeScan(results: ScanResults)
  - failScan(error: string)
  - cancelScan()
  - reset()

ScanProgress interface:
interface ScanProgress {
  percentComplete: number;
  currentService: string;
  servicesCompleted: number;
  servicesTotal: number;
  resourcesDiscovered: number;
  elapsedSeconds: number;
}
```

---

## Controller Implementation Prompts

### 11. Create Chat Controller

```
Create a ChatController in src/controllers/ChatController.ts

Requirements:
- TypeScript class-based controller
- Dependencies:
  - useChatStore from '@/models/state/chatStore'
  - MockTauriService from '@/services/tauri/MockTauriService'
  - MockWebSocketService from '@/services/websocket/MockWebSocketService'

- Methods:
  - sendMessage(text: string): Promise<void>
  - handleTextChunk(data: { text: string }): void (private)
  - handleEnhancement(data: Enhancement): Promise<void> (private)
  - destroy(): void (cleanup)

Flow:
1. User sends message
2. Add user message to store
3. Add empty assistant message to store
4. Get chat history from Tauri
5. Send to server via WebSocket
6. Listen for text_chunk events → append to assistant message
7. Listen for enhancement event → bucket UI to cache, history to Rust
8. Set streaming to false

Include proper error handling and cleanup.
```

### 12. Create Estate Scan Controller

```
Create an EstateScanController in src/controllers/EstateScanController.ts

Requirements:
- TypeScript class
- Dependencies:
  - useScanStore
  - MockTauriService

- Methods:
  - startScan(config: ScanConfig): Promise<void>
  - cancelScan(): Promise<void>
  - loadResults(): Promise<void>
  - handleScanProgress(progress: ScanProgress): void (private)
  - handleScanComplete(data: any): void (private)
  - destroy(): void

Flow:
1. User submits scan config
2. Update store (status: 'running')
3. Call Tauri startScan
4. Listen to scan_progress events → update store
5. Listen to scan_complete event → load results, update store
6. Handle errors gracefully

Setup event listeners in constructor, cleanup in destroy.
```

### 13. Create Playbook Controller

```
Create a PlaybookController in src/controllers/PlaybookController.ts

Requirements:
- TypeScript class
- Dependencies:
  - usePlaybookStore (create this store first)
  - MockTauriService

- Methods:
  - executePlaybook(playbook: Playbook): Promise<void>
  - pauseExecution(): Promise<void>
  - resumeExecution(): Promise<void>
  - cancelExecution(): Promise<void>
  - handleExecutionOutput(data: ExecutionOutput): void (private)
  - handleStepComplete(data: any): void (private)
  - destroy(): void

Flow:
1. User clicks Execute on PlaybookCard
2. Update store (status: 'executing')
3. Call Tauri executePlaybook
4. Listen to execution_output events → append to store
5. Listen to playbook_step_complete events → update step status
6. Handle completion or errors

Real-time output updates should be reflected in UI.
```

---

## View Building Prompts

### 14. Create Login View

```
Create a LoginView in src/views/auth/LoginView.tsx

Requirements:
- TypeScript functional component
- Tailwind CSS styling
- Form with:
  - Email input (with validation)
  - Password input (with show/hide toggle)
  - Remember me checkbox
  - Submit button with loading state
  - Forgot password link
  - Sign up link

- Use AuthController for login logic
- Display errors inline
- Responsive design (centered on large screens, full width on mobile)
- Loading spinner on submit
- Redirect to /dashboard on success

Example layout:
- Logo at top
- "Welcome back" heading
- Form card in center
- Links at bottom
```

### 15. Create Ops Chat View

```
Create an OpsChatView in src/views/chat/OpsChatView.tsx

Requirements:
- TypeScript functional component
- Full height layout (flex column)
- Three sections:
  1. Header (title + actions)
  2. Message list (scrollable, flex-1)
  3. Input area (fixed at bottom)

- Use useChatStore for state
- Use ChatController for actions
- Components:
  - MessageList (renders all messages)
  - MessageItem (individual message with avatar, content, timestamp)
  - ChatInput (textarea with send button)

- Features:
  - Auto-scroll to bottom on new message
  - Show "typing..." indicator when streaming
  - Render enhanced UI (PlaybookCard) for playbook messages
  - Empty state when no messages

- Responsive design
```

### 16. Create Estate Scan View

```
Create an EstateScanView in src/views/scan/EstateScanView.tsx

Requirements:
- TypeScript functional component
- Three-phase UI:
  1. Configuration form (when idle)
  2. Progress display (when scanning)
  3. Results summary (when complete)

- Configuration form:
  - Account multi-select
  - Region multi-select
  - Service multi-select
  - Start scan button

- Progress display:
  - Progress bar (0-100%)
  - Current service label
  - Resources discovered count
  - Elapsed time
  - Cancel button

- Results summary:
  - Total resources found
  - Breakdown by service (chart)
  - Breakdown by region (chart)
  - View details button

- Use useScanStore and EstateScanController
```

### 17. Create Learning Center View

```
Create a LearningCenterView (Dashboard) in src/views/estate/LearningCenterView.tsx

Requirements:
- TypeScript functional component
- Dashboard layout with cards grid
- Overview cards:
  - Total Resources (with icon)
  - Monthly Cost (with trend)
  - Security Score (with gauge)
  - Open Alerts (with count)

- Charts section:
  - Cost trend (line chart)
  - Resources by service (pie chart)

- Recent activity section:
  - Last scan timestamp
  - Recent changes list

- Quick actions:
  - Start new scan button
  - View recommendations button

- Use useEstateStore for data
- Responsive grid (1 col mobile, 2-3 cols tablet, 4 cols desktop)
```

---

## Service Layer Prompts

### 18. Create Mock Tauri Service

```
Create a MockTauriService in src/services/tauri/MockTauriService.ts

Requirements:
- TypeScript class
- Implement ALL 70+ Tauri commands as async methods
- Return mock data with realistic delays (50-500ms)
- Simulate events using window.dispatchEvent
- Event listeners return cleanup functions

Categories:
1. Chat commands (getChatHistory, saveMessage, deleteConversation)
2. Estate commands (getEstateResources, semanticSearch, getResourceDetails)
3. Policy commands (getPolicy, updatePolicy)
4. Scanner commands (startScan, cancelScan, getScanResults)
5. Execution commands (executeCommand, executePlaybook, cancelExecution)
6. Auth commands (storeTokens, getAccessToken, refreshToken, clearTokens)

Event simulation:
- scan_progress (emit every 1s during mock scan)
- scan_complete (emit after mock scan completes)
- execution_output (emit during mock execution)
- execution_complete (emit after mock execution)

Include helper methods:
- delay(ms): Promise<void>
- emitEvent(name, data): void
- addEventListener(name, callback): () => void
```

### 19. Create Mock WebSocket Service

```
Create a MockWebSocketService in src/services/websocket/MockWebSocketService.ts

Requirements:
- TypeScript class
- Methods:
  - connect(url: string): void
  - disconnect(): void
  - send(message: ClientMessage): void
  - on(event: string, callback: Function): void
  - off(event: string, callback: Function): void

- Simulate connection with 100ms delay
- When message sent:
  - If type is 'chat_message', simulate streaming response
  - Emit 'text_chunk' events for each word (50ms apart)
  - Emit 'enhancement' event after streaming complete
  - Detect if playbook needed based on message content

- Support multiple event handlers per event type
- Proper cleanup on disconnect
```

### 20. Create Mock API Service

```
Create a MockAPIService in src/services/api/MockAPIService.ts

Requirements:
- TypeScript class
- HTTP methods:
  - get(endpoint: string, params?: any): Promise<T>
  - post(endpoint: string, data: any): Promise<T>
  - put(endpoint: string, data: any): Promise<T>
  - delete(endpoint: string): Promise<T>

- Implement endpoints:
  - getRecommendations(): Promise<Recommendation[]>
  - applyRecommendation(id: string): Promise<{ playbookId: string }>
  - postponeRecommendation(id: string, until: number): Promise<void>
  - dismissRecommendation(id: string, reason?: string): Promise<void>
  - getAlerts(): Promise<Alert[]>
  - acknowledgeAlert(id: string): Promise<void>
  - resolveAlert(id: string, resolution?: string): Promise<void>

- Return mock data with realistic delays (300-1000ms)
- Log all operations to console
- Simulate errors (5% chance) with proper error objects
```

---

## Advanced Prompts

### 21. Create Dynamic Component Registry

```
Create a DynamicRenderer component in src/views/shared/ui-agent/DynamicRenderer.tsx

Requirements:
- TypeScript functional component
- Component registry pattern
- Props: { type: string; data: any }

- Registry mapping:
  - 'playbook' → PlaybookCard
  - 'chart' → LineChart/BarChart/PieChart (based on data.chartType)
  - 'table' → Table
  - 'card' → Card
  - 'alert' → Alert
  - ... add more as needed

- Fallback to generic Card if type not found
- Pass data as props to rendered component

Example usage:
const enhancement = {
  type: 'playbook',
  data: { ...playbookData }
};

<DynamicRenderer {...enhancement} />
```

### 22. Create Custom Hook - useTauriEvent

```
Create a useTauriEvent hook in src/hooks/useTauriEvent.ts

Requirements:
- TypeScript custom hook
- Parameters: eventName (string), callback (function)
- Use useEffect to setup/cleanup listener
- Auto-cleanup on unmount

Example usage:
useTauriEvent<ScanProgress>('scan_progress', (progress) => {
  console.log('Scan progress:', progress);
});
```

### 23. Create Route Guards

```
Create a PrivateRoute component in src/routes/PrivateRoute.tsx

Requirements:
- TypeScript functional component
- Check if user is authenticated (useAuthStore)
- If authenticated, render children
- If not authenticated, redirect to /login
- Show loading spinner while checking auth status

Example usage:
<Route path="/dashboard" element={
  <PrivateRoute>
    <LearningCenterView />
  </PrivateRoute>
} />
```

---

## Testing Prompts

### 24. Create Component Test

```
Create a test file for the Button component using Vitest and React Testing Library

Location: src/views/shared/basic/Button.test.tsx

Requirements:
- Test all variants render correctly
- Test click handler is called
- Test disabled state prevents clicks
- Test loading state shows spinner
- Test icon positioning (left/right)
- Use proper accessibility queries
- Mock any external dependencies

Include:
- describe block for component
- individual test cases
- cleanup after each test
```

### 25. Create Controller Test

```
Create a test file for ChatController using Vitest

Location: src/controllers/ChatController.test.ts

Requirements:
- Mock useChatStore
- Mock TauriService and WebSocketService
- Test sendMessage flow:
  - Adds user message to store
  - Calls Tauri to get history
  - Sends message to WebSocket
- Test handleTextChunk updates store
- Test handleEnhancement buckets correctly
- Test error handling
- Test cleanup in destroy method

Use vi.mock() for dependencies
```

---

## Tips for Using These Prompts

1. **Be specific**: Add more details if needed for your use case
2. **Iterate**: Run the prompt, review the code, ask for refinements
3. **Test**: Always test generated code before integrating
4. **Customize**: Adapt prompts to match your team's conventions
5. **Combine**: Use multiple prompts together for complex features

**Happy coding with Claude! 🚀**