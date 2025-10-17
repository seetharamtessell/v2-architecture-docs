# User Flows

Complete documentation of all user journeys through the Escher application.

---

## Flow Overview

The frontend supports 8 complete user flows:

1. **Login & Authentication**
2. **Estate Scanning**
3. **Learning Center (Estate Status)**
4. **Permissions Management**
5. **Recommendations**
6. **Alerts & Notifications**
7. **Ops Chat with Playbook Execution**
8. **Reports Generation**

---

## Flow 1: Login & Authentication

### Purpose
Authenticate user via AWS Cognito

### Steps
1. User opens app
2. Sees login screen
3. Enters username + password
4. App authenticates via Cognito
5. Token stored securely
6. Redirected to dashboard (Learning Center)

### Components
- **View**: LoginView, LoginForm
- **Controller**: AuthController
- **Service**: CognitoAuthService
- **State**: AuthState (user session)

### Success Criteria
- Valid JWT token stored
- User session established
- Redirected to main app

---

## Flow 2: Estate Scanning

### Purpose
Discover and index cloud resources (AWS/Azure/GCP) across accounts/subscriptions/projects and regions

### Steps
1. User navigates to Estate Scanner
2. Selects cloud provider (AWS/Azure/GCP)
3. Selects accounts/subscriptions/projects, regions, services to scan
4. Clicks "Start Scan"
5. Controller triggers Rust Estate Scanner
6. Real-time progress updates displayed
7. Scan completes, resources stored in Rust Storage Service
8. Summary displayed

### Components
- **View**: EstateScanView, ScanProgressBar
- **Controller**: EstateScanController
- **Service**: EstateScannerService (Tauri)
- **Rust Module**: Estate Scanner
- **State**: EstateUIState (scan progress)

### Data Flow
```
User Action
    ↓
EstateScanController
    ↓
EstateScannerService (Tauri)
    ↓
Rust Estate Scanner
    ↓
Tauri Events (scan_progress)
    ↓
Controller updates UI State
    ↓
View re-renders with progress
```

### Success Criteria
- All selected accounts/regions scanned
- Resources stored in Rust Storage Service
- Summary displayed with counts

---

## Flow 3: Learning Center (Estate Status)

### Purpose
View current state of cloud estate (AWS/Azure/GCP)

### Steps
1. User navigates to Learning Center (default landing page)
2. Controller fetches estate summary from Rust Storage Service
3. Displays overview cards:
   - Total resources
   - Monthly cost
   - Security score
   - Compliance status
4. Shows resource breakdown by service/region
5. Displays recent changes

### Components
- **View**: LearningCenterView, EstateOverviewCard
- **Controller**: EstateScanController
- **Service**: StorageService (Tauri)
- **Rust Module**: Storage Service
- **State**: EstateUIState (filters, selections)

### Data Sources
- Rust Storage Service (estate data)
- Qdrant (semantic search)
- Real-time metrics

### Success Criteria
- Summary loaded from Rust
- Cards display current stats
- Recent changes visible

---

## Flow 4: Permissions Management

### Purpose
Configure what AI agent can/cannot do

### Steps
1. User navigates to Permissions
2. Controller fetches current policy from Rust Storage Service
3. User views/edits:
   - Allowed operations (auto-approve)
   - Blocked operations (never allow)
   - Resource constraints (tags, instance types)
   - Approval settings (auto vs manual)
4. User clicks "Save"
5. Controller validates policy
6. Saves to Rust Storage Service

### Components
- **View**: PermissionsView, PolicyEditor
- **Controller**: PermissionController
- **Service**: StorageService (Tauri)
- **Rust Module**: Storage Service
- **State**: PermissionUIState (form state)

### Policy Structure
```
AgentPolicy:
  - allowed_operations: {ec2: [stop, start], rds: [describe]}
  - blocked_operations: {ec2: [terminate], rds: [delete]}
  - resource_constraints: {tags, instance_types, cost_limits}
  - approval_required: {stop: false, delete: true}
```

### Success Criteria
- Policy loaded from Rust
- Changes validated
- Policy saved successfully

---

## Flow 5: Recommendations

### Purpose
View and manage AI-generated optimization suggestions

### Steps
1. User navigates to Recommendations
2. Controller fetches recommendations from Server
3. User sees list of recommendations:
   - Cost savings
   - Security improvements
   - Performance optimizations
   - Compliance fixes
4. User can:
   - **Apply**: Trigger playbook to implement
   - **Explain**: View detailed impact analysis
   - **Postpone**: Snooze for later
   - **Dismiss**: Mark as not applicable

### Components
- **View**: RecommendationsView, RecommendationCard
- **Controller**: RecommendationController
- **Service**: APIService (HTTP)
- **State**: RecommendationUIState (selected, dialog state)

### Actions

**Apply**:
- Generates playbook (server-side)
- Sends to chat as enhanced UI
- User follows playbook execution flow

**Postpone**:
- User selects duration (1 day/week/month)
- Recommendation hidden until date
- Server tracks postponement

**Dismiss**:
- Marked as not applicable
- Removed from active list
- Server tracks dismissal

### Success Criteria
- Recommendations loaded from server
- Actions processed successfully
- UI updated accordingly

---

## Flow 6: Alerts & Notifications

### Purpose
Real-time monitoring and notifications

### Steps
1. User has app open
2. Server sends alert via WebSocket (or periodic polling)
3. Alert appears:
   - Banner for critical alerts (top of screen)
   - Notification bell badge (header)
4. User clicks notification bell
5. Sees alert list
6. Can:
   - **Acknowledge**: Mark as seen
   - **Resolve**: Mark as resolved (with action)

### Components
- **View**: AlertsView, AlertCard, NotificationBell
- **Controller**: AlertController
- **Service**: WebSocketService or APIService
- **State**: AlertUIState (unread count, selected)

### Alert Types
- **Cost**: Cost spike detected
- **Security**: Unauthorized access attempt
- **Availability**: Service outage
- **Compliance**: Resource out of compliance

### Success Criteria
- Alerts received in real-time
- Banner displayed for critical
- Badge count updated
- Actions processed

---

## Flow 7: Ops Chat with Playbook Execution

### Purpose
Interactive chat interface for cloud operations (AWS/Azure/GCP) with full playbook execution lifecycle (Explain → Execute → Monitor)

### Overview
Ops Chat is the primary interface for cloud operations (AWS/Azure/GCP). Users can chat naturally, and when operations are needed, the server returns playbooks as enhanced UI components that can be explained, executed, and monitored—all within the chat interface.

---

### Part A: Basic Chat Interaction

#### Steps
1. User navigates to Ops Chat
2. Types message: "Stop my RDS db-prod-01"
3. Message sent to server via WebSocket
4. Server responds with:
   - **Phase 1: Stream** (text response, immediate)
   - **Phase 2: Enhancement** (rich UI, 1-2s later)
5. If playbook → displayed as enhanced UI component (continue to Part B)
6. User can continue chatting

#### Components
- **View**: OpsChatView, MessageList, MessageItem
- **Controller**: ChatController
- **Service**: WebSocketService, StorageService (Tauri)
- **State**: ChatUIState (messages, uiCache), HistoryModel (Rust)

#### Message Types
- **User message**: Plain text
- **Assistant text**: Streaming text response
- **Assistant enhanced**: Rich UI (playbook, chart, table)

#### Critical: Bucketing
When enhancement arrives:
```
Enhancement {
  ui: {...}      → BUCKET 1: ChatUIState.uiCache (transient)
  history: {...} → BUCKET 2: Rust Storage Service (persistent)
}
```

**Why**:
- UI cache: For rendering (2-10KB per message)
- History: For LLM context (200-500 bytes per message)
- Next message sends history (not UI cache)

---

### Part B: Playbook Execution Flow

When server sends a playbook as enhanced UI, the user enters the playbook execution flow:

#### Step 1: Playbook Arrives
- Server sends playbook as enhanced UI component
- Displayed in chat as PlaybookCard
- Shows: title, summary, steps, metadata, risk level

#### Step 2: User Clicks "Explain Plan"
- Opens modal with detailed explanation
- For each step:
  - Description
  - Command to be executed
  - Impact/risks
  - Estimated time
- User can review full plan
- User can close modal or proceed to execute

#### Step 3: User Clicks "Execute"
- PlaybookController triggered
- Updates PlaybookUIState (execution in progress)
- Loops through steps sequentially
- For each step:
  - Calls ExecutionEngineService (Tauri)
  - Rust Execution Engine runs cloud CLI command (AWS/Azure/GCP)
  - Real-time output streamed back via Tauri events
  - Step status updated (pending → running → completed/failed)

#### Step 4: Monitor Execution
- ExecutionProgress component shows:
  - Current step indicator
  - Real-time command output (stdout/stderr)
  - Progress indicators for each step
  - Pause/Cancel buttons
- User can:
  - Watch real-time progress
  - Pause execution
  - Cancel execution
  - View output for each step

#### Step 5: Completion
- All steps completed (or some failed)
- Summary displayed in chat
- Execution saved to history (Rust Storage Service)
- User can continue chatting

---

### Components

**Chat Components**:
- **View**: OpsChatView, MessageList, MessageItem, InputArea
- **Controller**: ChatController
- **Service**: WebSocketService, StorageService (Tauri)
- **State**: ChatUIState (messages, uiCache, input value)

**Playbook Components**:
- **View**: PlaybookCard, PlaybookStepList, ExecutionProgress, ExplainPlanModal
- **Controller**: PlaybookController
- **Service**: ExecutionEngineService (Tauri)
- **Rust Module**: Execution Engine
- **State**: PlaybookUIState (execution status, current step, modal visibility)

---

### Complete Data Flow

```
User types message
    ↓
ChatController.sendMessage()
    ↓
WebSocketService sends to server
    ↓
Server responds with stream
    ↓
ChatController.handleStreamChunk()
    ↓
View shows streaming text
    ↓
Server sends enhancement (playbook)
    ↓
ChatController.handleEnhancement()
    ├─→ BUCKET 1: Cache UI in ChatUIState
    └─→ BUCKET 2: Save history to Rust Storage
    ↓
View shows PlaybookCard
    ↓
User clicks "Explain Plan"
    ↓
PlaybookController.explainPlan()
    ↓
Modal displays detailed explanation
    ↓
User clicks "Execute"
    ↓
PlaybookController.executePlaybook()
    ↓
For each step:
    ExecutionEngineService.executeCommand()
    ↓
    Rust Execution Engine
    ↓
    Tauri event: execution_output_{id}
    ↓
    PlaybookController receives output
    ↓
    Updates step status and output
    ↓
    View re-renders ExecutionProgress
    ↓
All steps complete
    ↓
Summary displayed in chat
    ↓
User can continue chatting
```

---

### Success Criteria

**Chat**:
- Messages sent/received successfully
- Text streams immediately (<100ms first chunk)
- Enhanced UI appears 1-2s later
- Bucketing implemented correctly

**Playbook Execution**:
- Playbook displayed as enhanced UI in chat
- "Explain Plan" shows detailed breakdown
- Execution proceeds step-by-step
- Real-time output visible for each step
- All steps completed successfully (or failed gracefully)
- User can monitor progress and cancel if needed

---

## Flow 8: Reports Generation

### Purpose
Generate visual analytics reports

### Steps
1. User navigates to Ops Chat
2. Types: "Show me cost report for December"
3. Server processes request
4. Server responds with:
   - **Phase 1: Stream** (text description)
   - **Phase 2: Enhancement** (rich UI with charts/tables)
5. Enhanced UI displays:
   - Summary card (total cost, change %)
   - Pie chart (cost by service)
   - Line chart (daily costs)
   - Table (top resources by cost)
6. User can:
   - Export as PDF/CSV
   - Save report
   - Ask follow-up questions

### Components
- **View**: Same as Ops Chat (MessageItem with DynamicRenderer)
- **Controller**: ChatController
- **Service**: WebSocketService
- **UI Components**: Chart, Table, Card components from UI Components package

### Report Types
- Cost reports (monthly/quarterly/yearly)
- Resource utilization
- Security compliance
- Performance metrics
- Custom queries

### Success Criteria
- Text response streams immediately
- Rich UI appears 1-2s later
- Charts/tables render correctly
- Export functionality works

---

## Navigation Structure

```
App Layout (Sidebar + Main Content)
├── Dashboard (Learning Center)
├── Ops Chat
├── Reports (Ops Chat with report focus)
├── Estate Scanner
├── Recommendations
├── Alerts
├── Permissions
└── Settings
```

---

## Cross-Flow Patterns

### Pattern 1: Recommendation → Playbook
1. User sees recommendation
2. Clicks "Apply"
3. Server generates playbook
4. Playbook sent to Ops Chat as enhanced UI
5. User follows playbook execution flow

### Pattern 2: Alert → Ops Chat
1. User sees alert
2. Clicks "Resolve"
3. Opens Ops Chat with pre-filled context
4. User can take action via chat

### Pattern 3: Learning Center → Ops Chat
1. User sees resource in Learning Center
2. Clicks "Manage"
3. Opens Ops Chat with resource context
4. User can perform operations

---

## State Transitions

### Chat Message States
```
pending → streaming → text_complete → enhanced
```

### Playbook States
```
idle → explaining → executing → paused/completed/failed
```

### Scan States
```
idle → configuring → scanning → completed/failed
```

### Alert States
```
unread → read → acknowledged → resolved
```

---

## Error Handling

### Chat Errors
- Message send failed → Retry button
- Enhancement timeout → Keep text version
- WebSocket disconnect → Auto-reconnect

### Execution Errors
- Step failed → Show error, option to retry
- Timeout → Show partial output, mark as failed
- Cancel → Graceful shutdown

### Scan Errors
- Account access denied → Continue with other accounts
- Service unavailable → Mark as partial scan
- Timeout → Retry logic

---

All user flows are designed to be:
- **Intuitive**: Clear next steps
- **Resilient**: Graceful error handling
- **Responsive**: Real-time updates
- **Informative**: Clear feedback at each step