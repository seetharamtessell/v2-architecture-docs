# Agent System Overview

## Purpose
Multi-agent architecture for intelligent AWS operations processing.

## Agent Types

### 1. Classification Agent
- **Role**: Intent recognition and request routing
- **Input**: User prompt + enriched resource context
- **Output**: Intent classification, confidence score, routing decision

### 2. Operations Agent
- **Role**: Generate execution scripts using playbook knowledge
- **Input**: Classified intent + complete resource context
- **Output**: Ready-to-execute AWS CLI scripts

### 3. Validation Agent
- **Role**: Validate requests for feasibility and safety
- **Input**: User request + resource constraints
- **Output**: Validation result, blocking issues

### 4. Risk Assessment Agent
- **Role**: Analyze and score operation risks
- **Input**: Operation details + resource metadata
- **Output**: Risk level, impact assessment, approval requirements

## Agent Orchestration
Agents collaborate through a coordination layer that manages:
- Agent invocation sequence
- Data passing between agents
- Error handling and fallback
- Performance optimization

## Details
- [classification-agent.md](classification-agent.md)
- [operations-agent.md](operations-agent.md)
- [validation-agent.md](validation-agent.md)
- [risk-assessment-agent.md](risk-assessment-agent.md)
- [agent-orchestration.md](agent-orchestration.md)