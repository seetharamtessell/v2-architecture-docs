# UI Agent Components Package

**Package Name**: `@escher/ui-agent-components`
**Purpose**: Standalone npm package for server-driven dynamic UI rendering
**Status**: Design Complete

---

## Overview

The UI Agent Components package is a **standalone, reusable library** that provides dynamic UI components based on server specifications. It enables server-driven UI architecture where the server decides what to display and the frontend has all components ready to render.

---

## Core Concept

### Server-Driven UI

```
Server sends specification:
  type: 'chart'
  props: {chartType: 'pie', title: 'Cost by Service'}
  data: {labels: [...], datasets: [...]}

Frontend:
  1. Looks up 'chart' in ComponentRegistry
  2. Maps to ChartComponent
  3. Renders with provided props + data
```

---

## Package Structure

```
@escher/ui-agent-components/
├── core/
│   ├── registry/
│   │   └── ComponentRegistry.ts    # Type string → React component mapping
│   └── renderer/
│       └── DynamicRenderer.tsx     # Core rendering engine
│
├── types/
│   ├── ui-component.ts             # UIComponent, UIMode interfaces
│   ├── props.ts                    # Component prop interfaces
│   └── index.ts                    # Export all types
│
├── templates/                      # Pre-composed (20% use cases)
│   ├── ResourceActionConfirmation/
│   ├── CostBreakdownDashboard/
│   ├── ResourceListView/
│   ├── StatusMonitor/
│   └── ErrorDisplay/
│
├── generic/                        # Building blocks (80% use cases)
│   ├── Chart/
│   ├── Table/
│   ├── Form/
│   ├── Button/
│   ├── Card/
│   ├── List/
│   ├── Metric/
│   ├── Alert/
│   └── Markdown/
│
├── domain/                         # Domain-specific components
│   └── aws/
│       ├── ResourceList/
│       ├── StatusIndicator/
│       ├── CostBreakdown/
│       ├── Playbook/
│       └── Recommendation/
│
└── layouts/                        # Layout wrappers
    ├── GridLayout.tsx
    ├── ColumnLayout.tsx
    ├── TabLayout.tsx
    └── AccordionLayout.tsx
```

---

## Design Principles

### 1. Framework Agnostic
- Pure React components
- No business logic
- No Tauri/WebSocket dependencies
- Can be used in ANY React application

### 2. Server-Driven
- Server sends: type + props + data
- Frontend maps type → component
- Component renders with data

### 3. Extensible
- Plugin system for custom components
- Override default components
- Custom theme support

### 4. Type-Safe
- Full TypeScript support
- Exported type definitions
- Auto-complete in IDEs

### 5. Tree-Shakeable
- Import only what you need
- Named exports
- No bundle bloat

---

## Core Components

### ComponentRegistry

**Purpose**: Central registry mapping type strings to React components

**Design**:
```
Registry:
  'chart' → ChartComponent
  'table' → TableComponent
  'playbook' → PlaybookComponent
  ... (30+ components)

Methods:
  - register(type, component): Add custom component
  - get(type): Retrieve component
  - has(type): Check if exists
```

**Default Registry**:
Pre-loaded with all generic, AWS-specific, and template components.

**Custom Registration**:
Developers can register custom components without modifying source.

---

### DynamicRenderer

**Purpose**: Render any UIComponent based on type string

**Input**:
```
component: UIComponent = {
  id: string
  type: string
  props: Record<string, any>
  data: Record<string, any>
  layout?: LayoutConfig
}
```

**Logic**:
1. Look up component.type in ComponentRegistry
2. If found → render with props + data
3. If not found → render fallback (error + raw data)

**Output**:
Renders the selected React component

---

## Component Categories

### Templates (Pre-composed - 20%)

**Purpose**: Common patterns, fast rendering

**Examples**:
- **ResourceActionConfirmation**: Stop/Start/Delete confirmations
- **CostBreakdownDashboard**: Pie chart + table + metrics
- **ResourceListView**: Searchable/sortable resource grid
- **StatusMonitor**: Real-time metrics with progress bars
- **ErrorDisplay**: User-friendly error messages

**Usage**:
- Server sends template_id + minimal data
- No LLM needed
- Sub-200ms rendering

---

### Generic Components (Building blocks - 80%)

**Purpose**: Flexible building blocks for dynamic composition

**Chart Component**:
- Types: line, pie, bar, area, scatter
- Props: title, legend, grid, colors
- Data: labels, datasets
- Library: Recharts

**Table Component**:
- Props: sortable, searchable, paginated
- Data: columns, rows
- Features: filtering, pagination, selection

**Form Component**:
- Props: fields, validation rules
- Data: initial values
- Features: validation, submit handling

**Button Component**:
- Props: label, variant, icon
- Data: action payload
- Features: click handling, states

**Card Component**:
- Props: title, footer
- Data: content
- Features: collapsible, closable

**Others**:
- List, Metric displays, Alert banners, Markdown renderer

---

### AWS-Specific Components

**Purpose**: Domain-specific AWS components

**ResourceList Component**:
- Display AWS resources
- Props: searchable, filterable, actions
- Data: resources array
- Features: search, filter by tags/regions

**StatusIndicator Component**:
- Show resource status
- Props: size, showLabel
- Data: status (running/stopped/error)
- Features: color-coded badges

**CostBreakdown Component**:
- Display cost analysis
- Props: period, groupBy
- Data: cost data by service/region
- Features: charts + tables

**Playbook Component**:
- Display execution plan
- Props: title, riskLevel
- Data: steps, affected resources
- Features: explain, execute actions

**Recommendation Component**:
- Display optimization suggestions
- Props: category, severity
- Data: details, savings, resources
- Features: apply, postpone, dismiss actions

---

## Type Definitions

### UIComponent Interface

```
interface UIComponent {
  id: string;                    // Unique identifier
  type: string;                  // Component type (e.g., 'chart')
  props: ComponentProps;         // Configuration
  data: ComponentData;           // Display data
  layout?: LayoutConfig;         // Optional layout hints
}
```

### Component Contract

Every component receives:
- **props**: Configuration (colors, titles, options)
- **data**: Actual data to display
- **layout**: Optional layout hints (width, height, span)
- **onAction**: Callback for user interactions

Every component provides:
- Pure presentation rendering
- User interaction events (not handles them)
- Error handling for invalid data
- Responsive design

---

## Usage in Escher Frontend

### Installation (once published)

```
npm install @escher/ui-agent-components
```

### Import

```typescript
import {
  DynamicRenderer,
  ChartComponent,
  TableComponent,
  PlaybookComponent,
  type UIComponent,
} from '@escher/ui-agent-components';
```

### Usage in MessageItem

```typescript
const MessageItem = ({ message }) => {
  if (message.type === 'enhanced' && message.ui) {
    return (
      <div>
        {message.ui.components.map((component: UIComponent) => (
          <DynamicRenderer
            key={component.id}
            component={component}
            onAction={(action) => handleAction(message.id, action)}
          />
        ))}
      </div>
    );
  }

  return <div>{message.content}</div>;
};
```

---

## Server Integration

### Server sends Enhancement

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
        "props": {
          "chartType": "pie",
          "title": "Cost by Service",
          "showLegend": true
        },
        "data": {
          "labels": ["EC2", "RDS", "S3"],
          "datasets": [{
            "data": [567, 385, 295]
          }]
        }
      },
      {
        "id": "table_1",
        "type": "table",
        "props": {
          "title": "Top Resources",
          "sortable": true
        },
        "data": {
          "columns": ["Name", "Type", "Cost"],
          "rows": [
            ["i-12345", "EC2", "$145"],
            ["db-prod", "RDS", "$120"]
          ]
        }
      }
    ]
  }
}
```

### Frontend renders

```
DynamicRenderer receives components array
    ↓
For each component:
    Look up type in Registry
    ↓
    Render component with props + data
    ↓
User sees chart + table in chat
```

---

## Template vs Dynamic

### Templates (Fast Path)
- Pre-designed compositions
- Server sends: template_id + data
- No LLM needed
- <200ms rendering
- 20% of use cases

**Example**:
```json
{
  "mode": "template",
  "template_id": "cost_breakdown_dashboard",
  "inputs": {
    "period": "December 2024",
    "total": 1247.89,
    "breakdown": [...]
  }
}
```

### Dynamic (Flexible Path)
- LLM-generated compositions
- Server sends: full components array
- Novel UI patterns
- 1-2s rendering (includes LLM time)
- 80% of use cases

**Example**:
```json
{
  "mode": "dynamic",
  "components": [
    {type: "chart", ...},
    {type: "table", ...},
    {type: "card", ...}
  ]
}
```

---

## Extensibility

### Custom Component Registration

```typescript
import { registerComponent } from '@escher/ui-agent-components';

// Custom component
const MyCustomComponent = ({ props, data }) => {
  return <div>Custom UI</div>;
};

// Register it
registerComponent('my_custom', MyCustomComponent);

// Now server can send type='my_custom'
```

### Override Default Component

```typescript
import { registerComponent, ChartComponent } from '@escher/ui-agent-components';

// Custom chart with different library
const MyChartComponent = ({ props, data }) => {
  // Use Chart.js instead of Recharts
  return <canvas>...</canvas>;
};

// Override default
registerComponent('chart', MyChartComponent);
```

---

## Deployment Options

### Option 1: Monorepo Workspace
- Keep in same repo as frontend
- Use workspace protocol
- No publishing needed

### Option 2: Private npm Registry
- Publish to private registry
- Version control
- Organization-only access

### Option 3: Git Dependency
- Reference from Git repo
- Version via tags
- No npm setup

### Option 4: Public npm
- Open source
- Community contributions
- Public availability

---

## Benefits

### For Development
- **Separation of concerns**: UI components isolated
- **Independent testing**: Test components alone
- **Independent versioning**: Update without touching main app
- **Clear contracts**: Well-defined interfaces

### For Reusability
- **Multi-platform**: Web, desktop, mobile
- **Multi-project**: Other teams can use
- **Third-party**: External developers can extend
- **Open source potential**: Community project

### For Maintenance
- **Single source of truth**: All UI components in one place
- **Consistent updates**: Update once, use everywhere
- **Documentation**: Component docs with components
- **Storybook**: Visual component library

---

## Component Examples

### Chart Component

**Server sends**:
```json
{
  "type": "chart",
  "props": {
    "chartType": "line",
    "title": "Daily Costs",
    "showGrid": true
  },
  "data": {
    "labels": ["Mon", "Tue", "Wed"],
    "datasets": [{
      "label": "Cost",
      "data": [42, 38, 45]
    }]
  }
}
```

**Component renders**: Line chart with grid and title

---

### Playbook Component

**Server sends**:
```json
{
  "type": "playbook",
  "props": {
    "title": "Stop RDS Instance",
    "riskLevel": "low"
  },
  "data": {
    "steps": [
      {
        "id": "step_1",
        "description": "Verify instance status",
        "status": "pending"
      },
      {
        "id": "step_2",
        "description": "Stop instance",
        "status": "pending"
      }
    ]
  }
}
```

**Component renders**: Playbook card with steps and action buttons

User clicks "Execute" → Component emits action → Controller handles

---

### Table Component

**Server sends**:
```json
{
  "type": "table",
  "props": {
    "sortable": true,
    "searchable": true
  },
  "data": {
    "columns": [
      {"key": "name", "label": "Name"},
      {"key": "type", "label": "Type"},
      {"key": "cost", "label": "Cost"}
    ],
    "rows": [
      {"name": "i-12345", "type": "EC2", "cost": "$145"},
      {"name": "db-prod", "type": "RDS", "cost": "$120"}
    ]
  }
}
```

**Component renders**: Sortable, searchable table

---

## Action Handling

### Component emits action
```
PlaybookComponent: User clicks "Execute"
    ↓
Emits: onAction({type: 'execute', componentId: 'playbook_1'})
```

### View passes to Controller
```
MessageItem receives action
    ↓
Passes to: onMessageAction(messageId, action)
    ↓
OpsChatView receives
    ↓
Calls: PlaybookController.handleAction(action)
```

### Controller handles
```
PlaybookController.handleAction()
    ↓
Executes playbook via Rust Execution Engine
    ↓
Updates UI state
    ↓
View re-renders with execution progress
```

---

## Future Extensions

### Add New Domains
- `domain/gcp/`: Google Cloud components
- `domain/azure/`: Azure components
- Same patterns, different providers

### Add New Component Types
- Register via plugin system
- Extend without modifying core
- Community contributions

### Multi-Framework Support
- Vue/Angular adapters
- Same component logic
- Different framework wrappers

---

## Key Takeaway

The UI Agent Components package provides a **complete library of dynamic UI components** that can render any server-specified UI. It enables true server-driven architecture where:

- Server decides what to display
- Frontend has all components ready
- No frontend code changes needed for new UIs
- Clean separation between UI and business logic

This creates a powerful, flexible, and maintainable UI system.