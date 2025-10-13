# Escher - Multi-Cloud Operations Management Platform
## Product Vision & Architecture Goals

**Last Updated**: October 2025
**Status**: Active Discussion - Defining Complete Scope

---

## **🔐 Critical Architecture Principle**

**⚠️ ESCHER AI SERVER IS 100% STATELESS - REGARDLESS OF DEPLOYMENT MODEL**

Whether user chooses **Local Only** or **Extend My Laptop** deployment:
- **Escher AI Server stores NOTHING**: No user data, no cloud estate, no credentials, no chat history, no state
- **Escher AI Server is a pure processing engine**: Receives requests → Processes with RAG → Returns responses → Forgets everything
- **All state resides with the user**: On physical laptop (Local Only) or in user's cloud (Extend My Laptop)
- **Privacy guarantee**: User's cloud estate and credentials NEVER leave user's control

---

## Product Overview

**Escher** is a Multi-Cloud Operations AI Platform that enables users to manage cloud operations across **AWS, Azure, and GCP** through a unified conversational interface.

### Core Philosophy
- **Multi-Cloud Support**: Single platform for AWS, Azure, GCP (and future cloud providers)
- **Conversational Management**: Natural language interface for all cloud operations
- **Unified Experience**: Consistent interface across all cloud providers
- **User-Controlled State**: State and execution remain in user's control (local or user's cloud)
- **Privacy First**: Escher AI Server is 100% stateless - stores nothing about users
- **AI-Powered Intelligence**: Server provides cloud-agnostic operations knowledge, recommendations, and playbook generation
- **Flexible Deployment**: User chooses local-only or extended to cloud model

---

## Deployment Architecture

Escher offers **two deployment models** to meet different user needs:

### **Model 1: Local Only** (Beta / Lightweight Users)

**Target Users**: Individual contributors, small teams, users with simple operations

**Architecture**:
- **Physical Laptop** runs Tauri application (Rust backend + React frontend)
- **Local Execution**: All operations execute from laptop using Rust execution engine
- **Local State**: Chat history, estate inventory, credentials stored on laptop
- **Requirements**: Laptop must stay online for scheduled operations

**Pros**:
- Zero cloud infrastructure costs
- Complete local control
- Simple setup

**Cons**:
- Laptop must remain online for scheduled jobs
- Limited to laptop's compute resources
- No cross-device access to state

---

### **Model 2: Extend My Laptop** (Main Release / Power Users)

**Target Users**: Teams with scheduled operations, long-running tasks, 24/7 requirements

**Architecture**:
```
Physical Laptop (Tauri App) ←→ Escher AI Server (Stateless Brain)
        ↓↑                              ↑
Extend My Laptop (User's Cloud)          |
        ↓                                |
Cloud Schedulers + Execution + State ←--┘
```

**Components in User's Cloud**:

| Cloud Provider | Scheduler | Execution | State Storage | Credentials |
|---|---|---|---|---|
| **AWS** | EventBridge | Fargate | S3 | SSM Parameter Store |
| **Azure** | Logic Apps / Functions | Container Instances | Blob Storage | Key Vault |
| **GCP** | Cloud Scheduler | Cloud Run | Cloud Storage | Secret Manager |

**Setup Process**:
1. User chooses "Extend to Cloud" from physical laptop
2. User selects cloud provider for Extend My Laptop (AWS, Azure, or GCP)
3. Physical laptop uses **user's local credentials** to provision infrastructure in **user's account**:
   - Deploys **Escher-provided container image** (Rust execution engine + cloud CLIs)
   - Creates scheduler (EventBridge/Logic Apps/Cloud Scheduler)
   - Creates state storage (S3/Blob/GCS buckets)
   - Creates credential storage (SSM/Key Vault/Secret Manager)
4. User **installs cloud credentials** in Extend My Laptop (just like local laptop - AWS CLI configure, Azure CLI login, gcloud auth)
5. Physical laptop becomes thin client, Extend My Laptop is single source of truth

**Execution Model**:
- **Physical Laptop**: Interactive operations, ad-hoc queries, real-time tasks
- **Extend My Laptop**: Scheduled operations, long-running tasks, automation, reports
- **Event-Based Lifecycle**: Extend My Laptop starts on-demand, stops when idle (cost optimization)
  - Scheduler triggers wake up Extend My Laptop for scheduled jobs
  - Long-running operations keep Extend My Laptop alive until completion
  - Auto-stops after idle period

**Data Flows**:
1. **User Query**: Physical Laptop → Escher AI Server → AI response → Physical Laptop
2. **Interactive Execution**: Physical Laptop → Extend My Laptop → Cloud APIs → Results → State Storage (S3/Blob/GCS)
3. **Scheduled Execution**: Scheduler (EventBridge/etc) → Extend My Laptop → Cloud APIs → Results → State Storage
4. **State Sync**: Physical Laptop pulls latest state from Extend My Laptop storage

**Multi-Cloud Management**:
- Extend My Laptop (e.g., on AWS) manages **all clouds** (AWS + Azure + GCP)
- User installs credentials for all clouds in Extend My Laptop's credential store
- Example: AWS Fargate Extend My Laptop with AWS credentials (SSM) + Azure Service Principal (SSM) + GCP Service Account (SSM)

**Escher AI Server Role**:
- **100% Stateless**: Stores nothing about users, no credentials, no state
- **Provides**: LLM responses, operation suggestions, playbook generation, AI intelligence
- **Communication**: Physical Laptop ↔ AI Server (for conversational AI), Extend My Laptop ↔ AI Server (for scheduled job intelligence)

**Pros**:
- 24/7 operations without laptop online
- Scheduled jobs run reliably
- Long-running operations don't block laptop
- Cross-device access to state
- Event-based compute = lower costs than always-on

**Cons**:
- Cloud infrastructure costs (EventBridge, Fargate/Container Instances, S3/Blob/GCS)
- More complex setup
- Cold start delays for event-based execution

---

### **User Choice**

Users can switch between models:
- Start with **Local Only** for simplicity
- Upgrade to **Extend My Laptop** when they need scheduling/automation
- Downgrade back to **Local Only** anytime (Extend My Laptop infrastructure can be destroyed)

---

## Escher AI Server Architecture

### **Stateless Processing Engine**

The Escher AI Server is a **pure stateless processing engine** - it receives requests, processes them using RAG (Retrieval-Augmented Generation), and returns responses without storing any user data.

### **Server Capabilities**

**Built-in RAG Knowledge Base:**
- **Playbook Library**: Comprehensive library of cloud operation playbooks (AWS, Azure, GCP)
- **CLI Command Database**: Complete reference of cloud CLI commands and their usage
- **Best Practices**: Cloud architecture patterns, security guidelines, cost optimization strategies
- **Multi-Cloud Operations**: Cross-cloud equivalents and migration patterns

**AI Processing:**
- Natural language understanding (user intent extraction)
- Context-aware response generation
- Operation planning and sequencing
- Playbook generation and customization
- Anomaly detection and recommendations

### **Data Flow Details**

#### **1. Interactive Query Flow (Physical/Extend My Laptop → AI Server)**

```
User: "Show me all running EC2 instances in us-east-1"

Physical/Extend My Laptop (Client-Side Processing):
├─ Search local RAG (vector store):
│   ├─ Cloud Estate Inventory: Check for EC2 instances in us-east-1
│   ├─ Chat History: Retrieve previous conversation context
│   └─ Executed Operations: Check recent EC2-related operations
└─ Prepare context from RAG results

Physical/Extend My Laptop → Escher AI Server:
├─ Query: "Show me all running EC2 instances in us-east-1"
└─ Context from RAG:
    ├─ Previous chat history (last 5 messages for continuity)
    ├─ Relevant estate info (EC2 instance count, regions)
    └─ Recent operations (any EC2 operations in last hour)

Escher AI Server Processing:
├─ Parse intent: List resources
├─ Identify scope: EC2, us-east-1, running state
├─ RAG lookup: EC2 list commands/APIs
├─ Analyze context: Understand user's recent activities
└─ Generate response type: Information query (not execution)

Escher AI Server → Physical/Extend My Laptop:
├─ Response Type: "information" | "execution" | "report"
├─ Operation: { type: "list_ec2", filters: { region: "us-east-1", state: "running" } }
└─ Suggested Display: Table format with instance details

Physical/Extend My Laptop:
├─ If type = "information": Query cloud APIs locally, display results
├─ If type = "execution": Execute operation with Rust execution engine
├─ If type = "report": Generate report and store locally/S3
└─ Store interaction in RAG (chat history collection)
```

**Key Points:**
- **Client searches RAG first**: Cloud estate inventory, chat history, executed operations
- **Context sent to AI Server**: Previous chat history + relevant RAG results (not full snapshot)
- **AI Server processes transiently**: Does not store context, forgets after response
- **Privacy preserved**: AI Server never stores cloud estate, credentials, or chat history

#### **2. Execution Flow (User → AI Server → Execution)**

```
User: "Stop all dev EC2 instances in us-east-1"

Physical/Extend My Laptop (Client-Side Processing):
├─ Search local RAG (vector store):
│   ├─ Cloud Estate Inventory: Find all dev EC2 instances in us-east-1
│   ├─ Chat History: Retrieve full conversation history (for LLM context)
│   └─ Executed Operations: Recent EC2-related operations
└─ Prepare context from RAG results

Physical/Extend My Laptop → Escher AI Server:
├─ Query: "Stop all dev EC2 instances in us-east-1"
└─ Context from RAG:
    ├─ Full chat history (entire conversation for LLM context)
    ├─ Dev instances found: 5 instances with tag=dev in us-east-1 (i-xxx, i-yyy, ...)
    └─ Recent operations: List of recent EC2 operations (if any)

Escher AI Server:
├─ Intent: Stop resources
├─ Scope: EC2, us-east-1, tag=dev
├─ Context understanding: Full conversation history allows LLM to understand user's intent
├─ RAG lookup: Stop EC2 playbook
├─ Safety check: High-risk operation (stops multiple instances)
└─ Generate execution plan

Escher AI Server → Physical/Extend My Laptop:
├─ Response Type: "execution"
├─ Execution Plan:
│   ├─ Step 1: List EC2 instances with tag=dev in us-east-1 (5 instances found)
│   ├─ Step 2: Confirm instances with user
│   └─ Step 3: Stop instances (AWS CLI: aws ec2 stop-instances --instance-ids i-xxx i-yyy ...)
└─ Estimated Impact: 5 instances affected

Physical/Extend My Laptop Rust Execution Engine:
├─ Display execution plan to user
├─ Request user confirmation
├─ Execute playbook steps
├─ Store results in RAG (Executed Operations collection)
└─ Store audit log in RAG (Immutable Reports collection)

**Note**: Risk levels and approval workflows are not yet designed (see Collaboration & Approval Workflows section)
```

#### **3. Scheduled Job Flow (Extend My Laptop → AI Server)**

```
Scheduled Job: "Stop all dev VMs at 8pm daily"

EventBridge/Cloud Scheduler → Extend My Laptop (Fargate/Container Instance/Cloud Run)
└─ Trigger: Scheduled job execution

Extend My Laptop (Client-Side Processing):
├─ Load RAG from S3/Blob/GCS:
│   ├─ Cloud Estate Inventory: Find all dev VMs across clouds
│   ├─ Chat History: Load schedule creation context (when Manager created this schedule)
│   └─ Executed Operations: Check previous executions of this schedule
└─ Prepare context from RAG results

Extend My Laptop → Escher AI Server:
├─ Query: "Execute scheduled job: Stop all dev VMs at 8pm"
└─ Context from RAG:
    ├─ Schedule creation chat history (for LLM context)
    ├─ Dev VMs found: 15 VMs (5 AWS EC2, 6 Azure VMs, 4 GCP Compute Engine)
    └─ Last execution: Yesterday at 8pm, 15 VMs stopped successfully

Escher AI Server:
├─ Intent: Execute scheduled operation
├─ RAG lookup: Stop VMs playbook (multi-cloud)
├─ Context understanding: Schedule created by Manager, routine daily operation
├─ Generate execution plan for all clouds
└─ Return structured operations

Escher AI Server → Extend My Laptop:
├─ Response Type: "execution"
├─ Multi-Cloud Operations:
│   ├─ AWS: aws ec2 stop-instances --instance-ids i-xxx, i-yyy
│   ├─ Azure: az vm stop --resource-group dev --name vm1, vm2
│   └─ GCP: gcloud compute instances stop vm1 vm2 --zone=us-central1-a
└─ Expected Results: 15 VMs stopped (5 AWS, 6 Azure, 4 GCP)

Extend My Laptop Rust Execution Engine:
├─ Execute multi-cloud operations in parallel
├─ Store results in RAG (Executed Operations collection)
├─ Store audit logs in RAG (Immutable Reports collection)
├─ Upload RAG to S3/Blob/GCS
└─ Shutdown (event-based lifecycle)
```

#### **4. Playbook Generation Flow**

```
User: "Create a disaster recovery playbook for my production environment"

Physical Laptop (Client-Side Processing):
├─ Search local RAG (vector store):
│   ├─ Cloud Estate Inventory: Find all production resources (RDS, EC2, S3, ALB)
│   ├─ Chat History: Retrieve full conversation history (for LLM context)
│   └─ Executed Operations: Check existing backups, snapshots, replications
└─ Prepare context from RAG results

Physical Laptop → Escher AI Server:
├─ Query: "Create a disaster recovery playbook for my production environment"
└─ Context from RAG:
    ├─ Full chat history (entire conversation for LLM context)
    ├─ Production inventory: RDS (2 instances), EC2 (12 instances), S3 (5 buckets), ALB (3 load balancers)
    └─ Existing DR measures: RDS automated backups enabled, no S3 replication, no AMI automation

Escher AI Server:
├─ Intent: Generate playbook
├─ Context understanding: User needs DR playbook, some measures exist, gaps identified
├─ RAG lookup: DR best practices, backup strategies, multi-region patterns
├─ Analyze context: Identify critical resources and missing DR components
└─ Generate custom playbook addressing gaps

Escher AI Server → Physical Laptop:
├─ Response Type: "playbook"
├─ Playbook Name: "Production DR Playbook"
├─ Steps:
│   ├─ Step 1: Enable automated RDS snapshots (daily) - ✅ Already enabled
│   ├─ Step 2: Replicate S3 buckets to backup region - ⚠️ Missing, critical
│   ├─ Step 3: Create AMIs of critical EC2 instances - ⚠️ Missing, recommended
│   ├─ Step 4: Configure cross-region ALB with health checks
│   ├─ Step 5: Set up Route53 failover routing
│   └─ Step 6: Test failover procedure monthly
├─ Estimated Cost: $X/month (based on current production size)
└─ Compliance: Meets RTO=4h, RPO=1h requirements

Physical Laptop:
├─ Display playbook to user
├─ User reviews/modifies playbook
├─ Store playbook in RAG (Executed Operations collection)
└─ User can execute playbook on-demand or schedule it
```

**Playbook Management:**
- **Escher Playbook Library**: Server provides pre-built playbooks via RAG
- **User Playbooks**: Users can create/modify playbooks and store them locally or in their cloud
- **Playbook Override**: User playbooks override Escher-provided playbooks
- **Playbook Storage**: Local (Local Only mode) or S3/Blob/GCS (Extend My Laptop mode)

### **RAG Architecture**

**Client-Side RAG (Physical/Extend My Laptop - Rust):**
- **Local Knowledge Base Collections**:
  1. **Cloud Estate Inventory**: Current resource inventory across all clouds
  2. **Chat History**: Conversational history with AI
  3. **Executed Operations**: History of operations executed
  4. **Immutable Reports**: Cost reports, audit logs, compliance reports (to avoid repeated API calls)
- **Purpose**: Provides context to AI Server queries, enables offline operation documentation
- **Storage**: Local vector store (Local Only) or S3/Blob/GCS vector store (Extend My Laptop)

**Immutable Reports Collection:**
- **Cost Reports**: Daily snapshots of AWS Cost Explorer, Azure Cost Management, GCP Billing data
  - Prevents repeated API calls (reduces cost)
  - Historical cost analysis without hitting cloud APIs
  - Daily sync scheduled (Manager persona gets updated cost data automatically)
- **Audit Logs**: Immutable log of all operations
  - Daily sync ensures Manager has complete audit trail
  - Cannot be modified after creation (compliance requirement)
  - Stored in vector store for fast retrieval and AI analysis
- **Compliance Reports**: Security scans, policy violations, CIS benchmark results
  - Generated on-demand or scheduled
  - Stored in vector store for historical comparison

**Daily Sync for Manager Persona:**
- **Scheduled Job**: Daily sync at user-configured time (default: 2am)
- **Syncs**: Cost data, audit logs, compliance reports to vector store
- **Benefit**: Manager wakes up to fresh data without manual refresh
- **Cost Optimization**: Single daily API call instead of repeated queries

**Server-Side RAG (Escher AI Server):**
- **Global Knowledge Base**: All cloud provider APIs, CLI commands, playbooks, best practices
- **Purpose**: Provides cloud operations expertise and generates intelligent responses
- **Updates**: Escher continuously updates with new cloud features, best practices, security advisories

**Combined Power:**
- Client RAG provides user-specific context (estate + immutable reports)
- Server RAG provides cloud operations expertise
- Together they enable intelligent, context-aware, multi-cloud operations

---

### **Version 2 Release: Central Immutable Reports**

**Beta/V1 Release (Current):**
- Immutable reports stored in user's control:
  - **Local Only**: Local vector store on physical laptop
  - **Extend My Laptop**: Vector store in S3/Blob/GCS (user's cloud)
- Privacy-first: No reports leave user's environment

**V2 Release (Future):**
- **Optional**: User can choose to sync immutable reports to Escher-managed central location
- **Benefits**:
  - Cross-device access to reports (access from any device)
  - Team collaboration on reports
  - Longer retention without user cloud costs
  - Advanced analytics across historical reports
- **User Choice**: Opt-in only, users control what reports are synced
- **Privacy**: Reports are encrypted, user controls access
- **Migration**: Users can migrate from V1 (local) to V2 (central) anytime

### **Privacy & Security Model**

**What AI Server Receives:**
- ✅ Natural language queries
- ✅ Cloud estate snapshots (for context - processed transiently, not stored)
- ✅ Operation results (for generating recommendations - processed transiently)

**What AI Server NEVER Receives:**
- ❌ Cloud credentials (AWS keys, Azure service principals, GCP service accounts)
- ❌ Sensitive data from cloud resources (database contents, file contents, secrets)
- ❌ User identity information

**What AI Server NEVER Stores:**
- ❌ User data
- ❌ Cloud estate information
- ❌ Chat history
- ❌ Operation history
- ❌ Any user-specific state

**Processing Model:**
```
Request arrives → Load from RAG → Process with LLM → Generate response → Return → Forget everything
```

Every request is independent. The AI Server has no memory between requests.

---

## User Personas

### 1. **Manager**
- Reviews reports and analytics
- Sets budgets and cost policies
- Approves high-risk operations
- Schedules automated operations
- Monitors team activities
- Manages compliance requirements

### 2. **Executor** (Operations Engineer)
- Runs day-to-day operations conversationally
- Follows organizational policies
- Executes pre-approved playbooks
- **"Extend Me" Pattern**: Executes operations within manager-defined boundaries

---

## Cloud Management Operations

### 1. **Resource Operations** (Day-to-day)
- **Start/Stop/Restart** resources
  - AWS: EC2, RDS, Lambda
  - Azure: VMs, SQL Database, Functions
  - GCP: Compute Engine, Cloud SQL, Cloud Functions
- **Resize/Scale** resources (instance types, storage, compute)
- **Create/Delete** resources
- **Configure** resources (firewall rules, tags, settings)
- **Backup/Restore** operations
- **Snapshot management**

**Execution**: Client-side with user credentials (cloud-specific SDKs/APIs)

---

### 2. **Cost Management**
- Real-time cost analysis (current spend, trends)
- Budget tracking and alerts
- Cost optimization recommendations (rightsizing, unused resources)
- Resource utilization tracking
- Reserved instance analysis
- Savings plan recommendations
- Waste detection (idle resources, unattached volumes)

**Implementation**:
- Historical cost data stored in immutable reports collection (vector store)
- Daily snapshots from AWS Cost Explorer, Azure Cost Management, GCP Billing APIs
- Budget alerts: Periodic (evaluated during daily sync, default 2am)

---

### 3. **Reports & Analytics**
- **Infrastructure Reports**: Inventory, configuration, topology
- **Cost Reports**: By service, account, region, tag
- **Security Reports**: Vulnerabilities, policy violations, compliance
- **Performance Reports**: Resource utilization, bottlenecks
- **Change History**: Audit trail of operations
- **Compliance Reports**: CIS benchmarks, custom policies

**Implementation**:
- **Generation**: Both on-demand (user requests) and scheduled (daily sync at 2am)
- **Storage**: Immutable reports collection in vector store (local or S3/Blob/GCS)
- **Export Formats**: PDF, CSV, Excel, JSON (AI generates in requested format)
- **Retention Policy**: User-configurable (default: 90 days for cost, 1 year for audit logs)

---

### 4. **Automation & Scheduling**
- **Scheduled Operations**: Nightly shutdowns, weekend starts, periodic tasks
- **Automated Remediation**: Auto-stop idle instances, delete old snapshots
- **Backup Schedules**: Automated backup execution
- **Compliance Enforcement**: Auto-tag resources, enforce encryption
- **Cost Optimization**: Automated cleanup of waste

**Implementation**:
- **Local Only**: Physical laptop must be online for scheduled operations (local cron/scheduler)
- **Extend My Laptop**: Cloud schedulers (EventBridge/Logic Apps/Cloud Scheduler) trigger Extend My Laptop
- **Event-Driven**: EventBridge Events, Azure Event Grid, Cloud Pub/Sub trigger automated remediation
- **No AssumeRole needed**: Extend My Laptop uses credentials installed in SSM/Key Vault/Secret Manager

---

### 5. **Security & Compliance**
- **Security Scanning**: Misconfigurations, vulnerabilities, exposed resources
- **Compliance Checks**: CIS benchmarks, SOC2, HIPAA, custom policies
- **IAM Analysis**: Overprivileged roles, unused credentials, permission boundaries
- **Encryption Validation**: S3, EBS, RDS encryption status
- **Network Security**: Open ports, public resources, security group rules
- **Continuous Monitoring**: Real-time security posture

**Key Questions**:
- [ ] Continuous monitoring or on-demand scans?
- [ ] Alert delivery: In-app, email, Slack, PagerDuty?
- [ ] Remediation: Manual or automated?

---

### 6. **Multi-Account/Subscription/Project Management**
- **Org-Level Visibility**: All cloud accounts/subscriptions/projects in unified view
  - AWS: Organizations, Accounts
  - Azure: Management Groups, Subscriptions
  - GCP: Organizations, Projects
- **Cross-Account Operations**: Batch operations across cloud boundaries
- **Consolidated Reporting**: Org-wide costs, compliance, security across all clouds
- **Centralized Policy Enforcement**: Consistent policies across all cloud providers
- **Account Governance**: Account/subscription/project creation, access management

**Key Questions**:
- [ ] How many accounts per customer typically?
- [ ] Role-based access control per account/cloud?
- [ ] Cross-account assume role patterns (AWS), Service Principals (Azure), Service Accounts (GCP)?

---

### 7. **Collaboration & Approval Workflows**
- **Operation Approval**: Manager approves before Executor runs (for high-risk ops)
- **Change Tracking**: Audit log of all operations
- **Team Permissions**: Role-based access control (RBAC)
- **Notification System**: Alert team about operations, changes, issues
- **Commenting**: Team discussion on operations/reports

**Key Questions**:
- [ ] Approval workflow: Client-side or server-side?
- [ ] Real-time collaboration needed?
- [ ] Notification channels: In-app, email, Slack?

---

### 8. **AI-Powered Operations**
- **Conversational Queries**:
  - "What's my biggest cost driver across all clouds?"
  - "Show me all public storage buckets" (S3, Blob Storage, Cloud Storage)
  - "Which VMs are underutilized?"
- **Smart Recommendations**: AI suggests cloud-specific optimizations
- **Anomaly Detection**: Unusual spending, security events, performance issues across all clouds
- **Playbook Generation**: AI creates multi-step, multi-cloud operation plans
- **Natural Language Execution**:
  - "Stop all dev VMs in Azure West US"
  - "Enable encryption on all GCP buckets in project X"
- **Context-Aware Responses**: Understands user's complete multi-cloud estate

**Current**: Server-side AI with client-side context enrichment

---

## Architecture Questions to Resolve

### ✅ **Resolved - Deployment & Execution**

#### 1. Deployment Model
- ✅ **DECIDED**: Two models - Local Only (Beta) and Extend My Laptop (Main Release)
- ✅ User chooses deployment based on their needs

#### 2. Immediate Operations (User-Initiated)
- ✅ **DECIDED**: Executed by Physical Laptop (Local Only) or Extend My Laptop (Extended mode)
- ✅ Uses user's stored credentials

#### 3. Scheduled Operations (Automated)
- ✅ **DECIDED**:
  - **Local Only**: Physical laptop must be online (local cron/scheduler)
  - **Extend My Laptop**: Cloud scheduler (EventBridge/Logic Apps/Cloud Scheduler) triggers Extend My Laptop
  - User chooses model based on requirements

#### 4. State & Credentials Storage
- ✅ **DECIDED**:
  - **Local Only**: Stored on physical laptop
  - **Extend My Laptop**: Stored in user's cloud (S3/Blob/GCS for state, SSM/Key Vault/Secret Manager for credentials)
  - **Escher AI Server**: 100% stateless, stores nothing

#### 5. Continuous Monitoring & Automated Remediation
- ✅ **DECIDED**:
  - **Local Only**: Limited to when laptop online
  - **Extend My Laptop**: Event-driven via cloud schedulers (EventBridge Events, Azure Event Grid, Cloud Pub/Sub)

---

### ✅ **Resolved - Reports & Analytics**

#### 1. Historical Data for Reports
- ✅ **DECIDED**: Stored in vector store as immutable reports collection
  - **Local Only**: Local vector store on laptop
  - **Extend My Laptop**: Vector store in S3/Blob/GCS (user's cloud)
  - **V2 Release**: Optional central immutable reports (opt-in)
  - Retention policy: User-configurable (default recommendations: 90 days for cost, 1 year for audit logs)

#### 2. Cost Data Collection
- ✅ **DECIDED**: Daily snapshots to avoid repeated API calls
  - Direct API calls to AWS Cost Explorer, Azure Cost Management, GCP Billing APIs
  - **Daily sync** scheduled (default 2am, user-configurable)
  - Stored in immutable reports vector store for historical analysis
  - **Cost optimization**: Single daily API call instead of repeated queries
  - Manager persona gets fresh data automatically every morning

#### 3. Audit Logs
- ✅ **DECIDED**: Immutable audit logs in vector store
  - Cannot be modified after creation (compliance requirement)
  - Daily sync ensures complete audit trail
  - Fast retrieval via vector store for AI analysis
  - Manager persona has complete audit trail updated daily

#### 4. Report Generation
- ✅ **DECIDED**: Both on-demand and scheduled
  - On-demand: User requests via conversational interface
  - Scheduled: Daily sync for cost/audit logs
  - Export formats: PDF, CSV, Excel, JSON (AI generates in requested format)
  - ❓ **OPEN**: Email delivery option?

---

### 🟢 **Moderate - Collaboration & RBAC**

#### 1. Multi-Account/Subscription/Project Access
- ❓ **OPEN**: How managed?
  - User installs credentials for each account/subscription/project?
  - Cross-account AssumeRole chains (AWS), Service Principals (Azure), Service Accounts (GCP)?
  - Both patterns supported?

#### 2. Team Collaboration
- ❓ **OPEN**:
  - How do Manager and Executor personas collaborate?
  - Approval workflows stored where (local vs cloud)?
  - Real-time notifications needed?

#### 3. Audit Trail
- ✅ **DECIDED**: Immutable audit logs in vector store
  - **All operations logged**: In immutable reports collection (vector store)
  - **Storage**: Local (Local Only) or S3/Blob/GCS (Extend My Laptop)
  - **Immutable**: Cannot be modified after creation (compliance requirement)
  - **Daily sync**: Ensures complete audit trail updated daily at 2am
  - **Retention**: User-configurable (default: 1 year for audit logs)

---

## "Extend Me" Pattern

**Understanding Needed**:
- Manager defines operation templates/playbooks
- Executor runs them with "extend me" command
- Pre-approved operations with variable parameters
- Reduces approval overhead for routine operations

**Questions**:
- [ ] How are templates defined?
- [ ] What parameters can Executor modify?
- [ ] Approval workflow for template creation?
- [ ] Audit trail for "extend me" executions?

---

## Next Steps - Architecture Discussion

### ✅ Phase 1: Define Execution Model (COMPLETE)
1. ✅ Decided on two deployment models (Local Only vs Extend My Laptop)
2. ✅ Defined scheduled operations execution (local scheduler vs cloud scheduler)
3. ✅ Clarified privacy model (Escher AI Server 100% stateless, state in user's control)
4. ✅ Defined Extend My Laptop provisioning (Escher provisions in user's account with user's credentials)

### ✅ Phase 2: Define Data Architecture (COMPLETE)
1. ✅ Historical data retention strategy (vector store with immutable reports collection)
2. ✅ Reports generation and storage model (on-demand + scheduled, stored in vector store)
3. ✅ Cost data collection and aggregation approach (daily snapshots to vector store)
4. ✅ Export formats (PDF, CSV, Excel, JSON via AI generation)
5. ✅ Daily sync for Manager persona (cost + audit logs at 2am)

### ⏳ Phase 3: Define Personas & RBAC (PENDING)
1. Complete persona definitions (Manager, Executor, others?)
2. Permission model per persona
3. Approval workflows (local vs cloud-based)
4. "Extend me" pattern implementation details
5. Team collaboration mechanisms

### ⏳ Phase 4: Document Complete Operations (PENDING)
1. List all supported cloud operations by category
2. Define which operations need approval
3. Risk levels per operation type
4. Automation boundaries
5. Multi-account/subscription/project patterns

---

## Alignment Check

### ✅ **Fully Aligned & Documented**
- Multi-cloud platform (AWS, Azure, GCP) with conversational interface
- Two deployment models (Local Only and Extend My Laptop) based on user needs
- State and execution in user's control (local or user's cloud)
- Escher AI Server 100% stateless (no user data, credentials, or state)
- Extend My Laptop provisioned in user's account with user's credentials
- Scheduled operations via cloud schedulers (EventBridge/Logic Apps/Cloud Scheduler)
- Multi-cloud management from single Extend My Laptop
- Rust execution engine for operations (playbooks, CLI commands, shell scripts)
- **Immutable reports in vector store** (cost, audit logs, compliance)
- **Daily sync for Manager persona** (cost + audit logs at 2am)
- **Client-side RAG** with 4 collections (estate, chat, operations, immutable reports)
- **Server-side RAG** with playbook library and cloud operations knowledge
- **V2 release plan** for optional central immutable reports

### 🟡 **Partially Defined - Need Details**
- Multi-account/subscription/project credential management
- Collaboration and approval workflows
- "Extend me" pattern implementation

### ❌ **Not Yet Documented**
- Complete list of supported cloud operations by category
- Personas & RBAC model details
- Budget management features
- Notification and alerting mechanisms (email delivery, Slack, PagerDuty)

---

**This document will be updated as we make architectural decisions.**