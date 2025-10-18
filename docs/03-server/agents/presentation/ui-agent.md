# UI Agent - Working Design Document

**Status**: 🔄 Active Design
**Date**: October 15, 2025
**Purpose**: Server-side intelligent agent that transforms raw agent responses into structured UI specifications for dynamic frontend rendering

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Responsibilities](#core-responsibilities)
4. [Two-Phase Response Pattern](#two-phase-response-pattern)
5. [UI Agent Processing Pipeline](#ui-agent-processing-pipeline)
6. [Template vs Dynamic Routing](#template-vs-dynamic-routing)
7. [Component Vocabulary](#component-vocabulary)
8. [Dual Output System](#dual-output-system)
9. [Frontend Integration](#frontend-integration)
10. [Implementation Plan](#implementation-plan)

---

## Overview

### What is the UI Agent?

The **UI Agent** is a **SERVER-SIDE intelligent agent** that sits between backend agents (Playbook Agent, Cost Agent, Analysis Agent, etc.) and the frontend. Its job is to:

1. **Understand** raw responses from other agents
2. **Transform** data into structured UI specifications
3. **Add** UI markers, component types, and layout hints
4. **Generate** dual outputs (UI mode + History mode)
5. **Return** JSON that frontend can dynamically render

### Why Do We Need This?

```
WITHOUT UI AGENT:
Backend → Raw JSON → Frontend (hard-coded mapping) → Limited UI

WITH UI AGENT:
Backend → Raw JSON → UI Agent (intelligent transform) → Structured UI JSON → Frontend (dynamic render) → Infinite scalability
```

**Key Benefits**:
- ✅ **Frontend scalability** - Add new visualizations without touching backend
- ✅ **Consistent UX** - Centralized UI decisions
- ✅ **Intelligent presentation** - AI-powered component selection
- ✅ **Dynamic adaptation** - Novel queries get optimal UI automatically
- ✅ **Efficient context** - Compact history summaries (20x smaller)

---

## Architecture

### Complete System Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    ESCHER AI SERVER                         │
├─────────────────────────────────────────────────────────────┤
│  User Prompt                                                │
│      ↓                                                       │
│  Master Agent (Router)                                      │
│      ↓                      ↓                                │
│  OPERATIONAL          NON-OPERATIONAL                       │
│      ↓                      ↓                                │
│  Playbook Agent       Other Agents                          │
│  (Execution plan)     (Analysis, Cost, etc.)                │
│      ↓                      ↓                                │
│      |                      |                                │
│      |   ┌──────────────────┴──────────────────┐           │
│      |   │      UI AGENT (Server-Side)         │           │
│      |   │  • Analyzes response structure      │           │
│      |   │  • Selects optimal components       │           │
│      |   │  • Adds UI markers & layout hints   │           │
│      |   │  • Generates dual outputs           │           │
│      |   │    - UI mode (full components)      │           │
│      |   │    - History mode (compact digest)  │           │
│      |   └─────────────────────────────────────┘           │
│      |                      ↓                                │
│      └──────────────────────┴────────────────────┐          │
│                                                   ↓          │
│                    Structured JSON with UI specs            │
└─────────────────────────────────────────────────────────────┘
                              ↓
                         WebSocket
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      FRONTEND (Client)                      │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  UI Rendering Engine (Client-Side)                    │ │
│  │  • Receives structured JSON                           │ │
│  │  • Maps component types to React components           │ │
│  │  • Renders dynamically based on markers               │ │
│  │  • Stores UI in IndexedDB (display only)              │ │
│  │  • Stores history digest (send to server)             │ │
│  └───────────────────────────────────────────────────────┘ │
│                          ↓                                  │
│                  React UI Components                        │
│  (Charts, Tables, Cards, Confirmation dialogs, etc.)       │
└─────────────────────────────────────────────────────────────┘
```

### Key Architectural Principles

1. **UI Agent is SERVER-SIDE** - Runs on Escher AI Server, not client
2. **NO rendering** - Only generates structured specifications
3. **NO React knowledge** - Outputs generic UI markers
4. **Frontend agnostic** - Could work with React, Vue, Angular
5. **Streaming first** - Text streams immediately, enhancement follows

---

## Core Responsibilities

### 1. Response Analysis
**Input**: Raw JSON from backend agents (Cost Agent, Analysis Agent, etc.)

**Process**:
- Detect data structure (list, timeseries, metric, hierarchy)
- Identify intent (display, confirmation, form, alert)
- Extract entities (resource IDs, names, values)
- Analyze data volume (10 items vs 1000 items)

**Output**: Structured analysis object

### 2. Component Selection
**Input**: Analysis object + Component vocabulary

**Process**:
- Match data structure to optimal components
- Choose between template (fast) vs dynamic (flexible)
- Select visualization types (chart, table, card)
- Determine layout strategy (single, grid, tabs)

**Output**: Component specifications

### 3. UI Structure Generation
**Input**: Component specs + Raw data

**Process**:
- Add UI markers (`type`, `layout`, `priority`)
- Map data fields to component props
- Define interactions (buttons, forms, confirmations)
- Structure multi-component layouts

**Output**: Structured UI JSON

### 4. Dual Output Creation
**Input**: UI JSON + Original response

**Process**:
- Create **UI mode**: Full components with all data (2-10 KB)
- Create **History mode**: Compact digest for LLM context (200-500 bytes)
- Ensure history has essential context only

**Output**: Enhancement message with both modes

---

## Two-Phase Response Pattern

### Phase 1: Streaming (Immediate)

```
Backend Agent Response
    ↓
Streaming Coordinator (pass-through)
    ↓ (0ms wait)
Text chunks → Client (WebSocket)
    ↓
User sees streaming text immediately
```

**Timing**: 0ms - User never waits

### Phase 2: Enhancement (Background)

```
Backend Agent Response
    ↓
Streaming Coordinator (buffer complete response)
    ↓ (on stream completion)
Trigger UI Agent
    ↓
UI Agent Processing (500ms-2s)
    ↓
Enhancement → Client (WebSocket)
    ↓
Text smoothly transitions to rich UI
```

**Timing**: 500ms-2s - Happens in background while user reads text

### Example Flow

```
User: "Show me December costs"

[Phase 1 - Immediate]
0ms:   "I'll show you the cost breakdown for December..."
100ms: "Your total spending was $1,247.89..."
500ms: "The breakdown by service shows EC2 at..."

[Phase 2 - Background]
1000ms: UI Agent processing...
1500ms: Enhancement ready (pie chart + table + metrics)
        → Smooth fade transition to rich UI
```

---

## UI Agent Processing Pipeline

### Pipeline Overview

```
Raw Response → Analyzer → Decision Engine → Builder → Output Generator
```

### Step 1: Analyzer (Rule-Based)

**Purpose**: Fast intent and structure detection

**Logic** (NO LLM):
```go
func Analyze(response Response) Analysis {
    // Detect intent
    intent := detectIntent(response)  // "display", "confirmation", "form"

    // Analyze data structure
    structure := analyzeData(response.data)  // "list", "timeseries", "metric"

    // Identify interaction type
    interaction := detectInteraction(response)  // "read-only", "confirmation", "input"

    // Extract entities
    entities := extractEntities(response)  // resource IDs, names, values

    return Analysis{
        intent: intent,
        structure: structure,
        interaction: interaction,
        entities: entities,
        dataVolume: len(response.data),
        confidence: calculateConfidence()
    }
}
```

**Output**:
```json
{
  "intent": "display_costs",
  "structure": "timeseries_with_breakdown",
  "interaction": "read_only",
  "entities": ["December", "costs", "EC2", "RDS"],
  "dataVolume": 150,
  "confidence": 0.92
}
```

### Step 2: Decision Engine

**Purpose**: Route to template (20%) vs dynamic (80%)

**Logic**:
```go
func Decide(analysis Analysis) Decision {
    // Try to match templates
    for _, template := range templateRegistry {
        if matches, confidence := template.Match(analysis); matches {
            if confidence > 0.8 {
                return Decision{
                    mode: "template",
                    templateID: template.id,
                    confidence: confidence
                }
            }
        }
    }

    // Fallback to dynamic
    return Decision{
        mode: "dynamic",
        confidence: 0.7
    }
}
```

**Routes**:
- **Template** (20%): Known patterns, fast (<200ms), no LLM
- **Dynamic** (80%): Novel queries, flexible (<1.5s), LLM-powered

### Step 3: Component Builder

#### 3a. Template Builder (Fast Path)

**Input**: Template ID + Data

**Process**:
```go
func BuildFromTemplate(templateID string, data interface{}) UISpec {
    template := templateRegistry.Get(templateID)

    // Direct field mapping (no LLM)
    return UISpec{
        mode: "template",
        components: template.FillData(data)  // Simple data mapping
    }
}
```

**Example - Cost Breakdown Template**:
```json
{
  "template_id": "cost_breakdown_dashboard",
  "components": [
    {
      "id": "metric_1",
      "type": "metric",
      "props": {"label": "Total Cost", "format": "currency"},
      "data_mapping": {"value": "$.total_cost"}
    },
    {
      "id": "chart_1",
      "type": "chart",
      "props": {"chartType": "pie", "title": "Cost by Service"},
      "data_mapping": {
        "labels": "$.services[*].name",
        "values": "$.services[*].cost"
      }
    },
    {
      "id": "table_1",
      "type": "table",
      "props": {"sortable": true},
      "data_mapping": {
        "columns": ["Service", "Cost", "Change"],
        "rows": "$.services"
      }
    }
  ]
}
```

**Timing**: <200ms (deterministic mapping)

#### 3b. Dynamic Builder (Flexible Path)

**Input**: Analysis + Data + Component vocabulary

**Process**:
```go
func BuildDynamic(analysis Analysis, data interface{}) UISpec {
    // LLM prompt with analysis + vocabulary
    prompt := fmt.Sprintf(`
        Data structure: %s
        Intent: %s
        Available components: %s

        Generate optimal UI components.
    `, analysis.structure, analysis.intent, componentVocabulary)

    // Call LLM (GPT-4/Claude)
    response := llmClient.Generate(prompt, temperature: 0.3)

    // Parse and validate
    return parseUISpec(response)
}
```

**LLM Output** (JSON):
```json
{
  "components": [
    {
      "type": "chart",
      "chartType": "line",
      "props": {"title": "Daily Costs", "showGrid": true},
      "data": {"labels": [...], "datasets": [...]}
    },
    {
      "type": "table",
      "props": {"sortable": true, "searchable": true},
      "data": {"columns": [...], "rows": [...]}
    }
  ],
  "layout": "vertical"
}
```

**Timing**: <1.5s (includes LLM call)

### Step 4: Output Generator

**Purpose**: Create dual outputs (UI + History)

**Process**:
```go
func GenerateOutput(uiSpec UISpec, originalResponse Response) Enhancement {
    // 1. UI Mode (full data)
    uiMode := UIMode{
        mode: uiSpec.mode,
        components: uiSpec.components,
        layout: uiSpec.layout
    }

    // 2. History Mode (compact digest)
    historyDigest := generateDigest(originalResponse, uiSpec)

    return Enhancement{
        type: "enhance",
        messageID: originalResponse.messageID,
        ui: uiMode,          // 2-10 KB
        history: historyDigest  // 200-500 bytes
    }
}

func generateDigest(response Response, uiSpec UISpec) HistoryEntry {
    // LLM generates compact summary
    prompt := fmt.Sprintf(`
        Summarize this response in 1-2 sentences for conversation context:
        %s

        Include: action taken, key entities, important values.
        Exclude: UI details, verbose explanations.
    `, response.content)

    summary := llmClient.Generate(prompt, maxTokens: 50)

    return HistoryEntry{
        role: "assistant",
        content: summary,
        metadata: {
            entities: extractEntities(response),
            action: uiSpec.intent
        }
    }
}
```

**Output**:
```json
{
  "type": "enhance",
  "message_id": "msg_123",
  "ui": {
    "mode": "dynamic",
    "components": [
      {
        "id": "chart_1",
        "type": "chart",
        "props": {"chartType": "pie", "title": "Cost by Service"},
        "data": {"labels": ["EC2", "RDS", "S3"], "datasets": [...]}
      }
    ]
  },
  "history": {
    "role": "assistant",
    "content": "Displayed cost breakdown for December. Total: $1,247.89 across EC2 ($567), RDS ($385), S3 ($295).",
    "metadata": {
      "entities": ["December", "costs"],
      "action": "displayed_dashboard"
    }
  }
}
```

---

## Template vs Dynamic Routing

### 5 Predefined Templates (20% of queries)

#### 1. **resource_action_confirmation**
**Trigger**: Operational intents (stop, start, delete resources)

**Structure**:
```json
{
  "template_id": "resource_action_confirmation",
  "components": [
    {
      "type": "confirmation_card",
      "props": {"actionType": "stop", "riskLevel": "medium"},
      "data": {
        "operation": "Stop EC2 Instances",
        "resources": ["i-abc123", "i-def456"],
        "warnings": ["Production instances detected"],
        "buttons": ["Confirm", "Cancel"]
      }
    }
  ]
}
```

#### 2. **cost_breakdown_dashboard**
**Trigger**: Cost analysis queries

**Structure**: Metric + Pie Chart + Table

#### 3. **resource_list_view**
**Trigger**: List/describe resource queries

**Structure**: Searchable table with filters

#### 4. **status_monitor**
**Trigger**: Real-time status queries

**Structure**: Progress bars + metrics + timeline

#### 5. **error_display**
**Trigger**: Error responses

**Structure**: Alert card + error details + suggested actions

### Dynamic Builder (80% of queries)

**When**: Novel queries, complex data, no template match

**Examples**:
- "Compare costs between AWS and Azure"
- "Show me network topology"
- "What are the security risks in my setup?"

**Process**: LLM selects optimal components from vocabulary

---

## Component Vocabulary

### 30+ Available Components

#### 1. **Data Display**
- `metric` - Single value with label
- `stat_grid` - Multiple metrics in grid
- `chart` - Line, bar, pie, area charts
- `table` - Sortable, searchable, paginated tables
- `list` - Simple item list

#### 2. **Operation Components**
- `confirmation_card` - Action approval prompts
- `execution_plan` - Multi-step operation display
- `progress_tracker` - Real-time progress
- `execution_result` - Operation results

#### 3. **Resource Components**
- `resource_card` - Single resource display
- `resource_table` - Resource list with actions
- `resource_detail` - Detailed resource view
- `resource_comparison` - Compare resources

#### 4. **Interactive Components**
- `button_group` - Action buttons
- `form` - Input forms with validation
- `filter_panel` - Filter controls
- `search_bar` - Search input

#### 5. **Alert Components**
- `alert_card` - Notifications
- `error_card` - Error display
- `warning_card` - Warning messages
- `success_card` - Success confirmations

#### 6. **Visualization Components**
- `timeline` - Event timeline
- `heatmap` - Resource utilization heatmap
- `network_diagram` - Network topology
- `dependency_graph` - Resource dependencies

#### 7. **Report Components**
- `daily_report` - Morning report summary
- `compliance_report` - Compliance status
- `optimization_report` - Cost optimization suggestions

#### 8. **Layout Components**
- `tabs` - Tab navigation
- `accordion` - Collapsible sections
- `grid` - Grid layout
- `columns` - Column layout

---

## Dual Output System

### Why Two Outputs?

**Problem**: If we store full UI data in conversation history, LLM context explodes

```
WITHOUT DIGEST:
Turn 1: 5 KB (UI data)
Turn 2: 5 KB
Turn 3: 5 KB
After 50 turns: 250 KB → Expensive, slow, hits token limits
```

```
WITH DIGEST:
Turn 1: 300 bytes (digest only)
Turn 2: 300 bytes
Turn 3: 300 bytes
After 50 turns: 15 KB → Fast, cheap, efficient
```

### Output Structure

```json
{
  "type": "enhance",
  "message_id": "msg_123",

  "ui": {
    "mode": "dynamic",
    "components": [
      {
        "id": "chart_1",
        "type": "chart",
        "props": {"chartType": "pie", "title": "Cost by Service"},
        "data": {
          "labels": ["EC2", "RDS", "S3"],
          "datasets": [{
            "label": "Cost",
            "data": [567.32, 385.21, 295.36]
          }]
        }
      },
      {
        "id": "table_1",
        "type": "table",
        "props": {"sortable": true, "searchable": true},
        "data": {
          "columns": ["Service", "Cost", "Change"],
          "rows": [
            {"service": "EC2", "cost": "$567.32", "change": "+12%"},
            {"service": "RDS", "cost": "$385.21", "change": "-5%"},
            {"service": "S3", "cost": "$295.36", "change": "+3%"}
          ]
        }
      }
    ],
    "layout": "vertical"
  },

  "history": {
    "role": "assistant",
    "content": "Displayed cost breakdown for December. Total: $1,247.89 across 3 services (EC2, RDS, S3).",
    "metadata": {
      "entities": ["December", "costs", "EC2", "RDS", "S3"],
      "values": {"total": 1247.89},
      "action": "displayed_dashboard",
      "context_type": "cost_analysis"
    }
  }
}
```

### Size Comparison

```
UI Mode: ~5 KB (full component data with props, data, layout)
History Mode: ~300 bytes (compact summary with key entities)

Ratio: 20x smaller
```

---

## Frontend Integration

### Client-Side Flow

```
1. Server streams text
   → Frontend appends chunks to message content

2. User reads streaming text (0 wait time)

3. Server sends enhancement (500ms-2s later)
   → Frontend receives dual outputs

4. Frontend splits enhancement:

   Bucket 1 (UI Cache - IndexedDB):
   ✅ Store enhancement.ui
   ✅ Use for rendering rich UI
   ❌ Never send back to server

   Bucket 2 (Conversation History - Memory):
   ✅ Append enhancement.history to history array
   ✅ Send full history with next request
   ❌ Never store enhancement.ui here (20x too large!)

5. Frontend renders components
   → Maps component types to React components
   → Displays rich UI
```

### Frontend Storage Pattern

```typescript
// ✅ CORRECT PATTERN
interface Message {
  id: string;
  role: "user" | "assistant";
  content: string;  // Streamed text
  type: "text" | "enhanced";
  ui?: UIMode;  // Only for rendering, not for history
}

// Two separate stores:

// 1. UI Cache (IndexedDB) - For display only
const uiCache = new Map<string, UIMode>();
uiCache.set(messageId, enhancement.ui);  // Store full UI

// 2. Conversation History (Memory) - For LLM context
const conversationHistory: HistoryEntry[] = [];
conversationHistory.push(enhancement.history);  // Store digest only!

// When sending next request
websocket.send({
  content: userMessage,
  context: {
    history: conversationHistory  // Send digests only (20x smaller!)
  }
});
```

### Frontend Rendering Engine

The frontend has a simple **UI Rendering Engine** that:

1. **Receives** structured JSON from UI Agent
2. **Maps** component types to React components
3. **Renders** components dynamically

```typescript
// Component Registry
const componentRegistry = {
  "chart": ChartComponent,
  "table": TableComponent,
  "metric": MetricComponent,
  "confirmation_card": ConfirmationCardComponent,
  "resource_table": ResourceTableComponent,
  // ... 30+ components
};

// Dynamic Renderer
function renderComponent(spec: ComponentSpec) {
  const Component = componentRegistry[spec.type];

  if (!Component) {
    return <FallbackComponent spec={spec} />;
  }

  return <Component {...spec.props} data={spec.data} />;
}

// Render enhancement
function renderEnhancement(enhancement: Enhancement) {
  return enhancement.ui.components.map(spec =>
    renderComponent(spec)
  );
}
```

---

## Implementation Plan

### Phase 1: Foundation (Week 1-2)
- [ ] Streaming Coordinator setup
- [ ] Basic analyzer (rule-based intent detection)
- [ ] LLM client integration (OpenAI/Anthropic)
- [ ] Component vocabulary definition

### Phase 2: Template System (Week 3-4)
- [ ] Template registry
- [ ] 5 AWS templates implementation
- [ ] Template matching logic
- [ ] Fast template builder

### Phase 3: Dynamic Builder (Week 5-6)
- [ ] Dynamic component selection (LLM-powered)
- [ ] Component composition logic
- [ ] Layout optimization
- [ ] Fallback strategies

### Phase 4: Output Generation (Week 7)
- [ ] Dual output generator
- [ ] History digest generation (LLM)
- [ ] Metadata extraction
- [ ] Output validation

### Phase 5: Frontend Integration (Week 8)
- [ ] UI rendering engine (client-side)
- [ ] Component registry
- [ ] Storage pattern (UI cache + history)
- [ ] Dynamic component mapper

### Phase 6: Testing & Optimization (Week 9-10)
- [ ] End-to-end testing
- [ ] Performance optimization
- [ ] Error handling
- [ ] Monitoring & logging

### Phase 7: Production (Week 11-12)
- [ ] Security hardening
- [ ] Rate limiting
- [ ] Caching strategies
- [ ] Deployment

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| First chunk latency | <100ms | Time to first text |
| Stream complete | <3s | Full text visible |
| Template processing | <200ms | No LLM needed |
| Dynamic processing | <1.5s | With LLM call |
| Total user wait | **0s** | Stream shows immediately |
| UI mode size | 2-10 KB | Full component data |
| History mode size | 200-500 bytes | Compact digest (20x smaller) |

---

## Key Benefits

### 1. Frontend Scalability
**Before**: Adding new visualization requires backend changes
**After**: UI Agent can generate any component from vocabulary

### 2. Consistent UX
All UI decisions centralized in UI Agent, not scattered across frontend

### 3. Zero Wait Experience
Streaming text shows immediately while enhancement processes in background

### 4. Context Efficiency
History digests are 20x smaller → Send full conversation history without token bloat

### 5. Dynamic Adaptation
Novel queries automatically get optimal UI through LLM-powered selection

---

## Summary

**UI Agent** is a **server-side intelligent agent** that:

✅ Transforms raw backend responses into structured UI specifications
✅ Selects optimal components from vocabulary (30+ components)
✅ Routes between templates (20%, fast) vs dynamic (80%, flexible)
✅ Generates dual outputs (UI mode for display, history mode for context)
✅ Enables infinite frontend scalability without backend changes

**Frontend Rendering Engine** (client-side) simply:
- Receives structured JSON
- Maps types to React components
- Renders dynamically

**Result**: Truly scalable, dynamic, AI-powered UI system where server controls presentation and frontend remains simple and flexible.

---

**Status**: Ready for implementation
**Next**: Begin Phase 1 - Foundation (Streaming Coordinator + Analyzer)
**Owner**: Backend Team (Go) + Frontend Team (React)