# Agent System Overview

## Purpose
Multi-agent architecture for intelligent cloud operations processing.

## System Flow

```
User Prompt
    ↓
Master Agent (Router)
    ↓                      ↓
OPERATIONAL          NON-OPERATIONAL
    ↓                      ↓
Playbook Agent       Other Agents (Cost, Analysis, etc.)
    ↓                      ↓
Direct to UI         UI Agent (formats for presentation)
    ↓                      ↓
    └──────────┬───────────┘
               ↓
        Structured Response
               ↓
           Client UI
```

## Agent Types

### 1. Master Agent (Router)
- **Role**: Route requests to appropriate agent type
- **Input**: User prompt
- **Output**: Operational vs Non-operational classification
- **Routing**:
  - **Operational** → Playbook Agent (execution plans)
  - **Non-operational** → Other Agents → UI Agent (presentation)

### 2. Playbook Agent ✅ Complete
- **Role**: LLM + RAG intelligence for playbook search and recommendation
- **Input**: User intent + cloud estate context
- **Output**: Execution plans with risk assessment
- **Details**: **[playbook-agent.md](playbook-agent.md)** (2,227 lines)
- **Key Features**:
  - 4-step intelligence flow (Intent → RAG Search → LLM Ranking → Return)
  - 10 playbook lifecycle states
  - Searches escher_library + tenant_playbooks collections

### 3. Classification Agent
- **Role**: Intent recognition and request routing
- **Input**: User prompt + enriched resource context
- **Output**: Intent classification, confidence score, routing decision

### 4. Operations Agent
- **Role**: Generate execution scripts using playbook knowledge
- **Input**: Classified intent + complete resource context
- **Output**: Ready-to-execute scripts (AWS/Azure/GCP)

### 5. Validation Agent
- **Role**: Validate requests for feasibility and safety
- **Input**: User request + resource constraints
- **Output**: Validation result, blocking issues

### 6. Risk Assessment Agent
- **Role**: Analyze and score operation risks
- **Input**: Operation details + resource metadata
- **Output**: Risk level, impact assessment, approval requirements

### 7. UI Agent ✅ Complete (Server-Side)
- **Role**: Transform backend responses into structured UI specifications
- **Input**: Raw agent responses (Cost Agent, Analysis Agent, etc.)
- **Output**: Structured JSON with UI markers + dual outputs (UI mode + history digest)
- **Details**: **[ui-agent.md](ui-agent.md)** (comprehensive design)
- **Key Features**:
  - Template (20%, <200ms) vs Dynamic (80%, <1.5s) routing
  - 30+ component vocabulary
  - Dual outputs: UI mode (5 KB) + history digest (300 bytes) = 20x efficiency
  - Makes frontend infinitely scalable

## Agent Orchestration
Agents collaborate through a coordination layer that manages:
- Agent invocation sequence
- Data passing between agents
- Error handling and fallback
- Performance optimization
- Two-phase response (streaming + enhancement)

## Details
- **[playbook-agent.md](playbook-agent.md)** - ✅ Complete (2,227 lines)
- **[ui-agent.md](ui-agent.md)** - ✅ Complete (server-side presentation intelligence)
- [classification-agent.md](classification-agent.md) - 🔄 Planned
- [operations-agent.md](operations-agent.md) - 🔄 Planned
- [validation-agent.md](validation-agent.md) - 🔄 Planned
- [risk-assessment-agent.md](risk-assessment-agent.md) - 🔄 Planned
- [agent-orchestration.md](agent-orchestration.md) - 🔄 Planned