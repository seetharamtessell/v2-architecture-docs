# Server Architecture Overview

## Purpose
The server is a stateless, multi-agent system that provides operations intelligence without storing AWS credentials or estate data.

## Key Responsibilities
1. **Intent Classification**: Understand user requests and route to appropriate agents
2. **Operations Intelligence**: Generate precise execution scripts using playbook knowledge
3. **Risk Assessment**: Analyze and score operation risks
4. **Workflow Orchestration**: Coordinate multi-step operations
5. **Playbook Management**: Store and retrieve operational playbooks

## Server Architecture

```
┌──────────────────────────────────────────────────┐
│              API Gateway                         │
│         (Entry point for all requests)           │
└──────────────────────────────────────────────────┘
                      ↕
┌──────────────────────────────────────────────────┐
│              Agent System                        │
│  • Classification Agent                          │
│  • Operations Agent                              │
│  • Validation Agent                              │
│  • Risk Assessment Agent                         │
└──────────────────────────────────────────────────┘
                      ↕
┌──────────────────────────────────────────────────┐
│            Microservices                         │
│  • Playbook Service                              │
│  • RAG Service                                   │
│  • Script Generator                              │
│  • Workflow Engine                               │
│  • Auth Service                                  │
│  • Notification Service                          │
└──────────────────────────────────────────────────┘
                      ↕
┌──────────────────────────────────────────────────┐
│              Data Layer                          │
│  • Redis (metadata)                              │
│  • Qdrant (playbook vectors)                     │
│  • Git (playbook repository)                     │
│  • Audit Logs                                    │
└──────────────────────────────────────────────────┘
```

## Design Principles
- **Stateless**: No AWS credentials or estate data stored
- **Modular**: Microservices for independent scaling
- **Intelligent**: Multi-agent system for complex operations
- **Scalable**: Horizontal scaling of services
- **Observable**: Comprehensive logging and metrics

## Subdirectories
- [agents/](agents/) - AI agent system
- [microservices/](microservices/) - Service architecture
- [data/](data/) - Data storage and management
- [infrastructure/](infrastructure/) - Deployment and scaling
- [integration/](integration/) - API and external integrations

---
*See [architecture.md](../../architecture.md) for complete system context.*