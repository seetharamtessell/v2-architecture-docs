# Escher - Multi-Cloud Operations Management Platform
## Product Vision & Architecture Goals

**Last Updated**: October 2025
**Status**: Active Discussion - Defining Complete Scope

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

### **Model 2: Extended Laptop** (Main Release / Power Users)

**Target Users**: Teams with scheduled operations, long-running tasks, 24/7 requirements

**Architecture**:
```
Physical Laptop (Tauri App) ←→ Escher AI Server (Stateless Brain)
        ↓↑                              ↑
Extended Laptop (User's Cloud)          |
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
2. User selects cloud provider for Extended Laptop (AWS, Azure, or GCP)
3. Physical laptop uses **user's local credentials** to provision infrastructure in **user's account**:
   - Deploys **Escher-provided container image** (Rust execution engine + cloud CLIs)
   - Creates scheduler (EventBridge/Logic Apps/Cloud Scheduler)
   - Creates state storage (S3/Blob/GCS buckets)
   - Creates credential storage (SSM/Key Vault/Secret Manager)
4. User **installs cloud credentials** in Extended Laptop (just like local laptop - AWS CLI configure, Azure CLI login, gcloud auth)
5. Physical laptop becomes thin client, Extended Laptop is single source of truth

**Execution Model**:
- **Physical Laptop**: Interactive operations, ad-hoc queries, real-time tasks
- **Extended Laptop**: Scheduled operations, long-running tasks, automation, reports
- **Event-Based Lifecycle**: Extended Laptop starts on-demand, stops when idle (cost optimization)
  - Scheduler triggers wake up Extended Laptop for scheduled jobs
  - Long-running operations keep Extended Laptop alive until completion
  - Auto-stops after idle period

**Data Flows**:
1. **User Query**: Physical Laptop → Escher AI Server → AI response → Physical Laptop
2. **Interactive Execution**: Physical Laptop → Extended Laptop → Cloud APIs → Results → State Storage (S3/Blob/GCS)
3. **Scheduled Execution**: Scheduler (EventBridge/etc) → Extended Laptop → Cloud APIs → Results → State Storage
4. **State Sync**: Physical Laptop pulls latest state from Extended Laptop storage

**Multi-Cloud Management**:
- Extended Laptop (e.g., on AWS) manages **all clouds** (AWS + Azure + GCP)
- User installs credentials for all clouds in Extended Laptop's credential store
- Example: AWS Fargate Extended Laptop with AWS credentials (SSM) + Azure Service Principal (SSM) + GCP Service Account (SSM)

**Escher AI Server Role**:
- **100% Stateless**: Stores nothing about users, no credentials, no state
- **Provides**: LLM responses, operation suggestions, playbook generation, AI intelligence
- **Communication**: Physical Laptop ↔ AI Server (for conversational AI), Extended Laptop ↔ AI Server (for scheduled job intelligence)

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
- Upgrade to **Extended Laptop** when they need scheduling/automation
- Downgrade back to **Local Only** anytime (Extended Laptop infrastructure can be destroyed)

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

**Key Questions**:
- [ ] Historical cost data: Stored on server or only client?
- [ ] Cost API calls: From client or server?
- [ ] Budget alerts: Real-time or periodic?

---

### 3. **Reports & Analytics**
- **Infrastructure Reports**: Inventory, configuration, topology
- **Cost Reports**: By service, account, region, tag
- **Security Reports**: Vulnerabilities, policy violations, compliance
- **Performance Reports**: Resource utilization, bottlenecks
- **Change History**: Audit trail of operations
- **Compliance Reports**: CIS benchmarks, custom policies

**Key Questions**:
- [ ] Generated on-demand or scheduled?
- [ ] Stored where: Client or server?
- [ ] Export formats: PDF, CSV, Excel?
- [ ] Historical data retention policy?

---

### 4. **Automation & Scheduling**
- **Scheduled Operations**: Nightly shutdowns, weekend starts, periodic tasks
- **Automated Remediation**: Auto-stop idle instances, delete old snapshots
- **Backup Schedules**: Automated backup execution
- **Compliance Enforcement**: Auto-tag resources, enforce encryption
- **Cost Optimization**: Automated cleanup of waste

**Key Questions**:
- [ ] Who executes scheduled operations: Client or server?
- [ ] What if client is offline during scheduled time?
- [ ] Does server need AssumeRole access for scheduled operations?
- [ ] Event-driven automation: How triggered?

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
- ✅ **DECIDED**: Two models - Local Only (Beta) and Extended Laptop (Main Release)
- ✅ User chooses deployment based on their needs

#### 2. Immediate Operations (User-Initiated)
- ✅ **DECIDED**: Executed by Physical Laptop (Local Only) or Extended Laptop (Extended mode)
- ✅ Uses user's stored credentials

#### 3. Scheduled Operations (Automated)
- ✅ **DECIDED**:
  - **Local Only**: Physical laptop must be online (local cron/scheduler)
  - **Extended Laptop**: Cloud scheduler (EventBridge/Logic Apps/Cloud Scheduler) triggers Extended Laptop
  - User chooses model based on requirements

#### 4. State & Credentials Storage
- ✅ **DECIDED**:
  - **Local Only**: Stored on physical laptop
  - **Extended Laptop**: Stored in user's cloud (S3/Blob/GCS for state, SSM/Key Vault/Secret Manager for credentials)
  - **Escher AI Server**: 100% stateless, stores nothing

#### 5. Continuous Monitoring & Automated Remediation
- ✅ **DECIDED**:
  - **Local Only**: Limited to when laptop online
  - **Extended Laptop**: Event-driven via cloud schedulers (EventBridge Events, Azure Event Grid, Cloud Pub/Sub)

---

### 🟡 **Important - Reports & Analytics**

#### 1. Historical Data for Reports
- ❓ **OPEN**: Where stored and for how long?
  - **Local Only**: Local database on laptop (limited retention)?
  - **Extended Laptop**: S3/Blob/GCS (longer retention, configurable by user)?
  - Retention policy (30 days, 90 days, 1 year)?

#### 2. Cost Data Collection
- ❓ **OPEN**: How accessed?
  - Direct API calls to AWS Cost Explorer, Azure Cost Management, GCP Billing APIs?
  - Real-time or periodic snapshots?
  - Stored for historical analysis?

#### 3. Report Generation
- ❓ **OPEN**:
  - On-demand or scheduled?
  - Export formats: PDF, CSV, Excel, JSON?
  - Email delivery option?

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
- ❓ **OPEN**:
  - All operations logged where?
  - Immutable audit log requirements?
  - Compliance retention policies?

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
1. ✅ Decided on two deployment models (Local Only vs Extended Laptop)
2. ✅ Defined scheduled operations execution (local scheduler vs cloud scheduler)
3. ✅ Clarified privacy model (Escher AI Server 100% stateless, state in user's control)
4. ✅ Defined Extended Laptop provisioning (Escher provisions in user's account with user's credentials)

### 🔄 Phase 2: Define Data Architecture (IN PROGRESS)
1. ❓ Historical data retention strategy
2. ❓ Reports generation and storage model
3. ❓ Cost data collection and aggregation approach
4. ❓ Export formats and delivery mechanisms

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
- Two deployment models (Local Only and Extended Laptop) based on user needs
- State and execution in user's control (local or user's cloud)
- Escher AI Server 100% stateless (no user data, credentials, or state)
- Extended Laptop provisioned in user's account with user's credentials
- Scheduled operations via cloud schedulers (EventBridge/Logic Apps/Cloud Scheduler)
- Multi-cloud management from single Extended Laptop
- Rust execution engine for operations (playbooks, CLI commands, shell scripts)

### 🟡 **Partially Defined - Need Details**
- Reports & analytics architecture (generation, storage, retention)
- Cost data collection and historical analysis
- Multi-account/subscription/project credential management
- Collaboration and approval workflows
- "Extend me" pattern implementation

### ❌ **Not Yet Documented**
- Complete list of supported cloud operations by category
- Personas & RBAC model details
- Budget management features
- Notification and alerting mechanisms
- Audit trail and compliance logging

---

**This document will be updated as we make architectural decisions.**