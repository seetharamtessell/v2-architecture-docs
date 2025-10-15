# Escher - Multi-Cloud Operations Architecture

## **Updated Architecture Overview**

```
┌──────────────────────────────────────────────────────────────────────┐
│                  CLIENT (Tauri Desktop App)                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Frontend (React + TypeScript)                             │    │
│  │  • Multi-Cloud Chat UI (AWS/Azure/GCP)                     │    │
│  │  • Resource Explorer (unified view)                        │    │
│  │  • Execution Monitor                                       │    │
│  │  • Alert Dashboard                                         │    │
│  │  • Cost Dashboard                                          │    │
│  │  • Morning Report                                          │    │
│  └────────────────────────────────────────────────────────────┘    │
│                          ↕                                          │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Rust Backend (Tauri Core)                                │    │
│  │  • Request Builder                                         │    │
│  │  • Resource Lookup Engine                                 │    │
│  │  • Multi-Cloud CLI/SDK Executor                           │    │
│  │    - AWS CLI/SDK                                          │    │
│  │    - Azure CLI/SDK                                        │    │
│  │    - GCP gcloud/SDK                                       │    │
│  │  • Workflow & Playbook Engine                             │    │
│  └────────────────────────────────────────────────────────────┘    │
│                          ↕                                          │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  LOCAL STORAGE LAYER (Vector Store)                       │    │
│  │                                                            │    │
│  │  ┌──────────────────┐  ┌──────────────────────────────┐  │    │
│  │  │ Qdrant           │  │ 6 RAG Collections:           │  │    │
│  │  │ (Vector DB)      │  │                              │  │    │
│  │  │                  │  │ 1. Cloud Estate Inventory    │  │    │
│  │  │ Multi-Cloud      │  │    AWS, Azure, GCP resources │  │    │
│  │  │ Semantic Search  │  │                              │  │    │
│  │  │                  │  │ 2. Chat History              │  │    │
│  │  │ • Resource       │  │    Conversational history    │  │    │
│  │  │   embeddings     │  │                              │  │    │
│  │  │ • Fast lookup    │  │ 3. Executed Operations       │  │    │
│  │  │ • Fuzzy matching │  │    History of operations     │  │    │
│  │  │                  │  │                              │  │    │
│  │  │ Unified Index:   │  │ 4. Immutable Reports         │  │    │
│  │  │ • Names, Tags    │  │    Cost, audit, compliance   │  │    │
│  │  │ • Cloud Provider │  │                              │  │    │
│  │  │ • Resource Type  │  │ 5. Alerts & Events           │  │    │
│  │  │ • IDs, ARNs      │  │    Alert rules, history      │  │    │
│  │  │                  │  │                              │  │    │
│  │  │                  │  │ 6. User Playbooks            │  │    │
│  │  │                  │  │    Custom playbooks          │  │    │
│  │  └──────────────────┘  └──────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────┘    │
│                          ↕                                          │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Multi-Cloud Credentials (Secure Storage)                 │    │
│  │  • OS Keychain (macOS/Windows/Linux)                      │    │
│  │  • AWS: Access Keys, Session Tokens                       │    │
│  │  • Azure: Service Principals, Managed Identity            │    │
│  │  • GCP: Service Account Keys                              │    │
│  │  • Never sent to server                                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
                              ↕
                         HTTPS/REST
                              ↕
┌──────────────────────────────────────────────────────────────────────┐
│                    ESCHER AI SERVER (Stateless)                      │
├──────────────────────────────────────────────────────────────────────┤
│  • Multi-Agent System                                                │
│    - Classification Agent                                            │
│    - Multi-Cloud Operations Agent (AWS/Azure/GCP)                    │
│    - Risk Assessment Agent                                           │
│  • Global RAG Knowledge Base                                         │
│    - Playbook Library (AWS, Azure, GCP operations)                   │
│    - CLI Command Database (complete multi-cloud reference)           │
│    - Best Practices (architecture, security, cost)                   │
│    - Multi-Cloud Equivalents (AWS ↔ Azure ↔ GCP mappings)           │
│  • Stateless Processing                                              │
│    - NO user data storage                                            │
│    - NO cloud credentials                                            │
│    - NO cloud estate data                                            │
│    - Processes requests → Returns responses → Forgets everything     │
└──────────────────────────────────────────────────────────────────────┘
                              ↕
                     (Optional - User's Cloud)
                              ↕
┌──────────────────────────────────────────────────────────────────────┐
│              EXTENDED RUNTIME (User's Cloud Account)                 │
├──────────────────────────────────────────────────────────────────────┤
│  • Cloud Scheduler (EventBridge/Logic Apps/Cloud Scheduler)          │
│  • Container Execution (Fargate/Container Instances/Cloud Run)       │
│  • Vector Store in S3/Blob Storage/Cloud Storage                     │
│  • Credentials in SSM Parameter Store/Key Vault/Secret Manager       │
│  • Event-Based Lifecycle (starts on-demand, stops when idle)         │
│  • Manages ALL clouds from single location (AWS + Azure + GCP)       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## **Key Architecture Principles**

### **1. Client-Side Data Ownership**

**Philosophy**: User's cloud estate, credentials, and operational history stay in USER'S control.

| What Stays Local/User Cloud | What Goes to Escher Server |
|---|---|
| ✅ Cloud Estate Inventory | ✅ Natural language queries |
| ✅ Cloud Credentials | ✅ Context for AI processing |
| ✅ Chat History | ❌ No user data storage |
| ✅ Executed Operations | ❌ No credentials |
| ✅ Cost Reports | ❌ No estate data |
| ✅ Alerts & Events | ❌ Stateless - forgets after response |

**Benefits**:
- **Privacy**: Your infrastructure details never stored externally
- **Security**: Credentials never leave your environment
- **Control**: You own all operational data
- **Compliance**: Meets strictest data residency requirements

---

### **2. Stateless AI Server**

**Critical Design Decision**: Escher AI Server stores NOTHING about users.

```
Request Flow:
┌────────────────────────────────────────────────────────────┐
│ Client Query + Context → AI Server                         │
│   ↓                                                         │
│ AI Server processes with Global RAG                        │
│   ↓                                                         │
│ AI Server returns structured response                      │
│   ↓                                                         │
│ AI Server FORGETS EVERYTHING                               │
└────────────────────────────────────────────────────────────┘
```

**Why Stateless?**
- **Privacy Guarantee**: No user data persistence
- **Scalability**: Horizontal scaling without state management
- **Security**: Zero attack surface for user data theft
- **Simplicity**: No database, no backups, no state synchronization

---

### **3. Multi-Cloud Unified Architecture**

**Single Interface for AWS, Azure, and GCP**

```
User Query: "Show me all running VMs"
    ↓
Escher Client searches unified cloud_estate collection
    ↓
Results include:
  - AWS EC2 instances
  - Azure Virtual Machines
  - GCP Compute Engine instances
    ↓
Display in unified table with cloud provider column
```

**Multi-Cloud Resource Normalization**:

| Cloud Provider | Resource Type | Normalized Name |
|---|---|---|
| AWS | EC2 Instance | compute_instance |
| Azure | Virtual Machine | compute_instance |
| GCP | Compute Engine VM | compute_instance |
| AWS | RDS Database | database_instance |
| Azure | SQL Database | database_instance |
| GCP | Cloud SQL | database_instance |
| AWS | S3 Bucket | object_storage |
| Azure | Blob Storage | object_storage |
| GCP | Cloud Storage | object_storage |

---

### **4. Vector Store Architecture (6 Collections)**

**Purpose**: Fast semantic search across multi-cloud resources

#### **Collection 1: Cloud Estate Inventory**
```typescript
{
  id: "aws-123456789012-us-west-2-i-abc123",
  vector: [0.123, 0.456, ...], // Embedding
  payload: {
    cloud_provider: "aws" | "azure" | "gcp",
    resource_type: "compute_instance",
    account_id: "123456789012",
    region: "us-west-2",
    state: "running",
    tags: { env: "production", app: "main" },
    encrypted_data: "..." // Full metadata + permissions
  }
}
```

**Enables**:
- "show production database" → searches across AWS RDS, Azure SQL, GCP Cloud SQL
- "idle VMs in dev" → finds underutilized compute across all clouds
- "unencrypted storage" → searches S3, Blob Storage, Cloud Storage

#### **Collection 2: Chat History**
- Conversational history with AI
- Context for follow-up questions
- Append-only, no embeddings (dummy vectors)

#### **Collection 3: Executed Operations**
- History of all operations executed
- Audit trail for compliance
- Playbook execution logs

#### **Collection 4: Immutable Reports**
- Cost reports (daily snapshots from AWS Cost Explorer, Azure Cost Management, GCP Billing)
- Audit logs (immutable, cannot be modified)
- Compliance reports (CIS benchmarks, policy violations)
- **Purpose**: Avoid repeated API calls, enable fast historical analysis

#### **Collection 5: Alerts & Events**
- Alert rules and thresholds
- Alert history (real-time + scheduled scan alerts)
- Morning report data
- Auto-remediation settings

#### **Collection 6: User Playbooks**
- User's custom playbooks with full scripts
- Encrypted content (AES-256-GCM)
- Storage strategies: local_only, uploaded_for_review, uploaded_trusted, using_default
- Managed by Playbook Service crate

**Storage Location**:
- **Run on Laptop**: Local Qdrant + hourly backups to YOUR S3/Blob/GCS
- **Extend to Cloud**: Primary storage in YOUR S3/Blob/GCS (cloud-native durability)

---

### **5. Extended Runtime Architecture**

**Optional**: User can extend Escher to run 24/7 in THEIR cloud account.

**Components Provisioned in User's Cloud**:

| Cloud | Scheduler | Execution | State Storage | Credentials |
|---|---|---|---|---|
| AWS | EventBridge | Fargate | S3 | SSM Parameter Store |
| Azure | Logic Apps | Container Instances | Blob Storage | Key Vault |
| GCP | Cloud Scheduler | Cloud Run | Cloud Storage | Secret Manager |

**Use Cases**:
- **Scheduled Jobs**: "Stop all dev VMs at 8pm daily"
- **Real-Time Alerts**: Cloud-native monitoring with auto-remediation
- **Long-Running Operations**: Multi-hour scans, migrations
- **Cross-Device Access**: State syncs across laptop and cloud

**Event-Based Lifecycle**:
```
Scheduler triggers → Container starts → Loads vector store from S3
→ Executes operation → Saves results → Shuts down
```

**Cost Optimization**: Only runs when needed (event-driven compute)

---

## **Key Architecture Changes from Legacy AWS-Only**

| Aspect | Old (AWS-Only) | New (Escher Multi-Cloud) |
|--------|----------------|--------------------------|
| **Product Name** | AWS CloudOps AI Agent | Escher |
| **Cloud Support** | AWS only | AWS + Azure + GCP |
| **Collections** | 2 (chat_history, cloud_estate) | 6 (estate, chat, ops, reports, alerts, playbooks) |
| **Server** | Stateless | Still stateless (unchanged) |
| **Credentials** | AWS keys | AWS + Azure + GCP |
| **Resource Naming** | aws_estate (AWS-only) | cloud_estate (Multi-cloud) |
| **Deployment** | Local only | Laptop + Optional Cloud Extension |
| **Automation** | Limited | 24/7 with Extended Runtime |

---

## **Why Tauri?**

| Aspect | Electron | Tauri | Impact |
|--------|----------|-------|--------|
| **Size** | ~100MB | ~3-5MB | 95% smaller |
| **Performance** | Node.js | Rust | 2-5x faster execution |
| **Memory** | ~100-200MB | ~30-50MB | 70% less RAM |
| **Security** | JS isolation | Rust + OS sandbox | More secure |
| **Local DB** | Better Node support | Native Rust support | Qdrant native |

**Benefits for Multi-Cloud Operations**:
- **Fast CLI Execution**: Rust calls AWS/Azure/GCP CLI directly (no Node overhead)
- **Secure Credentials**: OS keychain integration (native APIs)
- **Low Resource Usage**: Important when managing thousands of resources
- **Cross-Platform**: macOS, Windows, Linux with single codebase

---

## **Data Flow Examples**

### **Query Flow: "Show idle Azure VMs"**

```
1. User types: "Show idle Azure VMs"
   ↓
2. Client Frontend → ChatController → Request Builder
   ↓
3. Request Builder:
   - Searches cloud_estate collection (vector + filter)
   - Filter: cloud_provider=azure, resource_type=compute_instance
   - Loads last 5 chat messages for context
   ↓
4. Sends to Escher AI Server:
   {
     query: "Show idle Azure VMs",
     context: {
       recent_messages: [...],
       azure_vms: [... summary of VMs found ...]
     }
   }
   ↓
5. Escher AI Server:
   - Understands intent: List + filter by utilization
   - RAG lookup: Azure VM idle detection methods
   - Returns: Structured query for Azure Monitor metrics
   ↓
6. Client Rust Backend:
   - Executes: az monitor metrics list ...
   - Filters VMs with CPU < 5% for 7 days
   - Stores results in chat_history collection
   ↓
7. Display results in UI table with:
   - VM name, size, location, average CPU, monthly cost
   - Actions: [Stop VM] [Deallocate] [Resize]
```

### **Execution Flow: "Stop all dev GCP instances"**

```
1. User: "Stop all dev GCP instances"
   ↓
2. Client searches cloud_estate:
   - Filter: cloud_provider=gcp, tags.env=dev, resource_type=compute_instance
   - Finds: 15 GCP Compute Engine instances
   ↓
3. Sends to Escher AI Server with full context
   ↓
4. AI Server analyzes:
   - High-risk operation (stops 15 instances)
   - Generates execution plan with safety checks
   - Returns structured playbook
   ↓
5. Client displays confirmation:
   "About to stop 15 GCP instances in dev environment:
    - project-x: 8 instances
    - project-y: 7 instances

    Continue? [Yes] [No] [Show Details]"
   ↓
6. User confirms → Rust Execution Engine:
   - Executes: gcloud compute instances stop ... (parallel)
   - Logs each operation in executed_operations collection
   - Stores audit trail in immutable_reports collection
   ↓
7. Display results:
   "✅ Successfully stopped 15 GCP instances
    ⏱️  Operation completed in 12 seconds
    💰 Estimated monthly savings: $450"
```

---

## **Security Architecture**

### **Credential Management**

**Storage**:
- **Laptop**: OS Keychain (Keychain Access on macOS, Windows Credential Manager, Linux Secret Service)
- **Extended Runtime**: SSM Parameter Store (AWS), Key Vault (Azure), Secret Manager (GCP)

**Access Pattern**:
```
User grants access → Credentials stored securely
    ↓
Escher Client/Runtime retrieves credentials
    ↓
Executes cloud CLI/SDK with credentials
    ↓
Credentials NEVER sent to Escher AI Server
```

### **Encryption**

**At Rest**:
- Vector store encrypted (AES-256)
- Sensitive payload fields encrypted individually
- Credentials encrypted by OS/Cloud provider

**In Transit**:
- HTTPS/TLS 1.3 for all network communication
- Certificate pinning for Escher AI Server

---

## **See Also**

- [PRODUCT-VISION.md](PRODUCT-VISION.md) - Complete product vision with deployment models
- [system-overview.md](system-overview.md) - High-level system overview
- [key-decisions.md](key-decisions.md) - Architecture Decision Records
- [technology-stack.md](technology-stack.md) - Detailed technology choices
- [../02-client/](../02-client/) - Client implementation details
- [../03-server/](../03-server/) - Server implementation details

---

**This architecture enables Escher to be a privacy-first, multi-cloud operations platform with the flexibility to run entirely on your laptop or extend to your cloud for 24/7 automation.**