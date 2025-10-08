# Microservices Architecture Overview

## Purpose
Modular, independently scalable services for server-side operations.

## Service Catalog

### Core Services

#### API Gateway
- Entry point for all client requests
- Request routing and load balancing
- Authentication and rate limiting

#### Auth Service
- User authentication and authorization
- Token management
- Access control

#### Playbook Service
- Playbook CRUD operations
- Versioning and rollback
- Git repository integration

#### RAG Service
- Vector search over playbooks
- Embedding generation
- Semantic matching

#### Script Generator
- AWS CLI script generation
- Parameter resolution
- Multi-cloud support (future)

#### Workflow Engine
- Multi-step operation orchestration
- State management
- Retry and error handling

#### Notification Service
- Alerts and notifications
- Webhook delivery
- Event streaming

## Service Communication
- **Synchronous**: REST/gRPC for request-response
- **Asynchronous**: Message queue for events
- **Service Discovery**: TBD (Consul/etcd/K8s DNS)

## Details
- [api-gateway.md](api-gateway.md)
- [auth-service.md](auth-service.md)
- [playbook-service.md](playbook-service.md)
- [rag-service.md](rag-service.md)
- [script-generator.md](script-generator.md)
- [workflow-engine.md](workflow-engine.md)
- [notification-service.md](notification-service.md)