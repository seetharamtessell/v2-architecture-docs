# Escher - System Overview

## Purpose
High-level overview of the Escher Multi-Cloud Operations Management Platform.

## What is Escher?

**Escher** is a Multi-Cloud Operations AI Platform that enables users to manage cloud operations across **AWS, Azure, and GCP** through a unified conversational interface.

## Architecture at a Glance

```
┌──────────────────────────────────────────────────────────────┐
│            USER'S ENVIRONMENT (100% Private)                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Physical Laptop (Tauri App)                                │
│  ├─ React Frontend (Multi-cloud UI)                         │
│  ├─ Rust Backend (Execution Engine)                         │
│  ├─ Local Vector Store (5 RAG Collections)                  │
│  └─ Secure Credentials Store (AWS/Azure/GCP)                │
│                                                              │
│  Extended Runtime (Optional - User's Cloud)                 │
│  ├─ Cloud Scheduler (EventBridge/Logic Apps/Cloud Scheduler)│
│  ├─ Container Execution (Fargate/Container Inst/Cloud Run)  │
│  ├─ Vector Store in S3/Blob/GCS                             │
│  └─ Credentials in SSM/Key Vault/Secret Manager             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                            ↕
                      HTTPS/REST
                            ↕
┌──────────────────────────────────────────────────────────────┐
│           ESCHER AI SERVER (100% Stateless)                  │
├──────────────────────────────────────────────────────────────┤
│  • Natural language understanding                            │
│  • Multi-cloud operations intelligence (AWS/Azure/GCP)       │
│  • Playbook generation and execution planning                │
│  • Global RAG (Cloud best practices, CLI commands)           │
│  • NO user data storage                                      │
│  • NO cloud credentials                                      │
│  • NO cloud estate data                                      │
└──────────────────────────────────────────────────────────────┘
```

## Key Components

### **Client (Tauri App)**
- **Local Vector Store**: 5 RAG collections for multi-cloud estate management
- **Execution Engine**: Rust-based multi-cloud CLI/SDK executor
- **Semantic Search**: Fast natural language resource lookup
- **Privacy-First**: Your data never leaves YOUR control

### **Escher AI Server (Stateless Brain)**
- **Multi-Agent System**: Specialized agents for operations intelligence
- **Global Knowledge**: Cloud best practices, playbooks, CLI commands
- **Stateless Design**: Processes requests, returns responses, forgets everything
- **Multi-Cloud Expertise**: AWS, Azure, and GCP unified operations

### **Extended Runtime (Optional)**
- **24/7 Automation**: Runs in YOUR cloud account (not ours)
- **Event-Driven**: Starts on demand, stops when idle (cost optimization)
- **Scheduled Operations**: Reliable execution without laptop online
- **Real-Time Alerts**: Cloud-native monitoring and auto-remediation

## Design Philosophy

1. **Client-Side Data Ownership**
   - Your cloud estate, credentials, and chat history stay with YOU
   - Vector store lives on your laptop or YOUR cloud (S3/Blob/GCS)
   - Zero data sent to Escher servers for storage

2. **Server-Side Operations Intelligence**
   - Stateless AI processing with global cloud knowledge
   - Playbook library, best practices, multi-cloud equivalents
   - Processes your queries with context, returns structured responses

3. **Privacy-First Architecture**
   - Escher AI Server stores NOTHING about your infrastructure
   - Your credentials NEVER leave your environment
   - Zero trust model - you own and control all sensitive data

4. **Multi-Cloud Unified Experience**
   - Single interface for AWS, Azure, and GCP
   - Cloud-agnostic operations with consistent UX
   - Unified vector store for cross-cloud resource management

5. **Flexible Deployment**
   - **Run on Your Laptop**: Simple setup, zero cloud compute costs
   - **Extend to Your Cloud**: 24/7 operations, scheduled jobs, real-time alerts
   - User choice - start simple, extend when needed

## Core Features

### **Conversational Operations**
- Natural language queries across all clouds
- "Show me all idle VMs in Azure West US"
- "Stop production database in GCP project-x"
- Context-aware responses with safety checks

### **Alert & Event System**
- **Real-Time Alerts**: Critical events with auto-remediation
- **Morning Report**: Daily proactive scan with AI-powered Q&A
- Unified event schema across AWS/Azure/GCP
- Customizable severity thresholds and alert rules

### **Cost Management**
- Real-time spend tracking across all clouds
- Anomaly detection and forecasting
- Optimization recommendations with 1-click fixes
- Historical cost analysis without repeated API calls

### **Multi-Cloud Semantic Search**
- Fast resource lookup: "production postgres database"
- Fuzzy matching: handles typos, abbreviations
- Tag-based filtering: "env=prod AND app=main"
- Unified search across AWS, Azure, and GCP

### **Playbook Automation**
- AI-generated playbooks for complex operations
- Pre-approved execution with safety checks
- Scheduled playbooks for routine maintenance
- Multi-cloud workflows (e.g., AWS → Azure migration)

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Desktop App** | Tauri 2.0 + React | Native multi-platform client |
| **Execution Engine** | Rust | Fast, safe CLI/SDK execution |
| **Vector Store** | Qdrant | Semantic search over cloud resources |
| **AI Server** | Multi-agent system | Stateless operations intelligence |
| **Cloud Schedulers** | EventBridge/Logic Apps/Cloud Scheduler | 24/7 automation |
| **State Storage** | S3/Blob Storage/Cloud Storage | Durable vector store |
| **Credentials** | SSM/Key Vault/Secret Manager | Secure credential management |

---

## Getting Started

1. **Install Escher Desktop App** (Tauri-based, ~5MB)
2. **Connect Cloud Accounts** (AWS, Azure, GCP)
3. **Sync Cloud Estate** (Build local vector store)
4. **Start Chatting** (Natural language operations)
5. **[Optional] Extend to Cloud** (Enable 24/7 automation)

---

*For detailed architecture documentation, see [PRODUCT-VISION.md](PRODUCT-VISION.md)*