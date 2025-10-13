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
- **Periodic Backup**: Local database snapshot synced to S3/Blob/GCS periodically (configurable interval, default: hourly)
- **Requirements**: Laptop must stay online for scheduled operations

**Pros**:
- Zero cloud compute infrastructure costs (only storage for backups)
- Complete local control
- Simple setup
- Automatic backup to cloud for disaster recovery

**Cons**:
- Laptop must remain online for scheduled jobs
- **Laptop must remain always-on for real-time alerts** (critical limitation for monitoring)
- Limited to laptop's compute resources
- No cross-device access to state (backups for recovery only, not real-time sync)

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

1. **Interactive Query Flow** (Local Only):
   ```
   User Query → Physical Laptop searches RAG → Sends query + context to AI Server
   → AI Server processes → Returns response (information/execution/report)
   → Physical Laptop executes locally → Stores results in local RAG
   → Periodic backup to S3/Blob/GCS (hourly)
   ```

2. **Interactive Query Flow** (Extend My Laptop):
   ```
   User Query → Physical Laptop searches local RAG → Sends query + context to AI Server
   → AI Server processes → Returns response
   → Physical Laptop sends execution request to Extend My Laptop
   → Extend My Laptop executes → Stores results in S3/Blob/GCS RAG
   → Physical Laptop syncs latest state from S3/Blob/GCS
   ```

3. **Scheduled Execution Flow** (Extend My Laptop only):
   ```
   Scheduler (EventBridge/Logic Apps/Cloud Scheduler) triggers at scheduled time
   → Extend My Laptop starts (Fargate/Container Instance/Cloud Run)
   → Loads RAG from S3/Blob/GCS (estate, chat history, previous executions)
   → Sends query + context to AI Server
   → AI Server returns execution plan
   → Extend My Laptop executes operations → Cloud APIs
   → Stores results in RAG → Uploads RAG to S3/Blob/GCS
   → Extend My Laptop shuts down (event-based lifecycle)
   ```

4. **State Synchronization**:
   - **Local Only**: Periodic backup from local RAG to S3/Blob/GCS (hourly, configurable)
   - **Extend My Laptop**: Physical Laptop pulls latest state from S3/Blob/GCS on demand or periodically
   - **Single Source of Truth**: Local RAG (Local Only) or S3/Blob/GCS RAG (Extend My Laptop)

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

### **Alert & Event Handling Architecture**

Escher provides **two types of alert systems** to ensure comprehensive monitoring - "Sensors" that act like the nervous system, continuously monitoring the cloud environment and alerting the "Brain" (AI Server) when action is needed.

#### **1. Real-Time Operational Alerts** (🚨 Can't Wait - Immediate Action Required)

**Purpose**: Immediate notification and action for critical events that require urgent attention

**Target Events**:
- 🔴 **CRITICAL**: Production database down, security breach (public S3 with PII), service outage, budget exceeded 200%
- 🟠 **HIGH**: Performance degradation, significant cost spike, compliance violation, resource failure
- 🟡 **MEDIUM**: Resource warnings, capacity approaching limits, optimization opportunities
- ℹ️ **INFO**: Informational events (aggregated in morning report)

**Setup Process** (During Extend My Laptop Installation):

1. **Add Escher Listener to Source of Truth**:
   User grants Escher permission to add event listeners to cloud-native alert sources:
   - **AWS**: CloudWatch Alarms → EventBridge → Extend My Laptop
   - **Azure**: Azure Monitor Alerts → Event Grid → Extend My Laptop
   - **GCP**: Cloud Monitoring → Pub/Sub → Extend My Laptop

2. **Pre-Approve Auto-Remediation** (Setup Wizard):
   User chooses which "first aid" actions Escher can perform automatically:
   ```
   ☑️ Automatically make public S3 buckets private
   ☑️ Automatically stop idle instances after 2 hours
   ☑️ Automatically enable encryption on unencrypted volumes
   ☑️ Automatically scale up resources approaching capacity
   ☐ Automatically restart failed services
   ☐ Automatically rollback failed deployments
   ```
   - User can modify these settings anytime in application settings
   - Each auto-remediation action logs to immutable audit trail

**Real-Time Alert Flow**:

```
Critical Event Occurs (e.g., S3 bucket made public)
↓
Cloud-Native Alert (CloudWatch/Azure Monitor/GCP Monitoring) detects at source of truth
↓
Event published to EventBridge/Event Grid/Pub Sub
↓
Extend My Laptop wakes up (Fargate/Container Instance/Cloud Run triggered)
↓
Extend My Laptop loads RAG from S3/Blob/GCS:
├─ Estate: Which S3 bucket? Production or dev? Contains PII?
├─ Alert Rules: User's configured severity thresholds and customized critical definitions
├─ Previous Incidents: Similar alerts in past? How resolved?
└─ Auto-Remediation Settings: Is "make bucket private" pre-approved?
↓
Normalize event to unified schema:
├─ event_type: "s3_bucket_public"
├─ severity: Auto-calculated based on estate context (PII detected = CRITICAL)
├─ resource: { bucket_name, region, account_id, tags }
├─ context: { environment: "production", data_classification: "PII" }
└─ timestamp: ISO-8601
↓
Send unified event + context to AI Server (Escher Brain)
↓
AI Server analyzes:
├─ Severity Assessment: CRITICAL (PII exposed publicly)
├─ Root Cause: Security group rule changed by user john@company.com at 10:34 AM
├─ First Aid Recommendation: Make bucket private immediately (aws s3api put-bucket-acl)
├─ Impact Assessment: ~1.2M customer records exposed, zero downtime to fix
├─ Playbook: Step-by-step remediation + prevent future occurrences
└─ Risk: If not fixed within 1 hour, potential GDPR violation
↓
Response sent back to Extend My Laptop
↓
Decision Point - Is auto-remediation pre-approved?
├─ YES → Execute immediately:
│   ├─ Run: aws s3api put-bucket-acl --bucket my-data --acl private
│   ├─ Verify: Bucket now private ✅
│   ├─ Store result in RAG (Alerts & Events collection)
│   └─ Prepare notification: "CRITICAL alert auto-resolved"
│
└─ NO → Request approval:
    ├─ Store alert in RAG
    └─ Prepare notification: "CRITICAL alert requires approval"
↓
Notification via cloud-native services:
├─ 🔴 CRITICAL: AWS SNS/Azure Notification Hubs/GCP Cloud Messaging
│   └─ Channels: Email + SMS + Slack/PagerDuty
├─ 🟠 HIGH: Email + Slack only
└─ 🟡 MEDIUM: In-app notification banner
↓
Store complete alert record in RAG (Alerts & Events collection):
├─ Event details, AI analysis, action taken, notification sent
├─ Immutable (cannot be modified after creation)
└─ Used for queries, comparisons, and compliance reporting
↓
Extend My Laptop shuts down (event-based lifecycle)
```

**Unified Event Schema** (Cross-Cloud Normalization):

All cloud events are normalized to a unified schema before sending to AI Server:

```typescript
interface UnifiedEvent {
  event_id: string;
  event_type: string; // "s3_bucket_public", "vm_stopped", "cost_spike", etc.
  severity: "CRITICAL" | "HIGH" | "MEDIUM" | "INFO"; // Auto-calculated or user-customized
  cloud_provider: "aws" | "azure" | "gcp";
  account_id: string;
  region: string;
  resource: {
    type: string; // "s3_bucket", "ec2_instance", etc.
    id: string;
    name: string;
    tags: Record<string, string>;
    metadata: Record<string, any>;
  };
  context: {
    environment?: "production" | "staging" | "dev";
    data_classification?: "PII" | "confidential" | "public";
    cost_impact?: number; // USD per day
    affected_users?: number;
  };
  timestamp: string; // ISO-8601
  raw_event: any; // Original cloud-specific event for reference
}
```

**Notification Channels** (Cloud-Native):

- **AWS**: Amazon SNS → Email, SMS, Slack (via webhook), PagerDuty (via integration)
- **Azure**: Azure Notification Hubs → Email, SMS, Teams, Slack
- **GCP**: Cloud Messaging → Email, SMS, Slack, PagerDuty

**Local Only Limitation**:
- Real-time alerts require **Extend My Laptop** for 24/7 monitoring
- OR laptop must remain **always-on** for Local Only mode
- Local Only users with always-on can receive real-time alerts via polling (every 1 minute for CRITICAL, every 5 minutes for HIGH)

---

#### **2. Scheduled Scan Alerts** (ℹ️ Can Wait - Interactive Morning Report)

**Purpose**: Proactive monitoring, optimization suggestions, and aggregated insights delivered daily

**Scan Schedule**: Daily at 2am (same as cost/audit sync), user-configurable

**Scans Performed**:
- 💰 **Cost Analysis**: Spending trends, budget tracking, anomaly detection, waste identification
- 🔒 **Security Posture**: Compliance checks (CIS, SOC2, HIPAA), policy violations, encryption status
- ⚡ **Resource Optimization**: Idle resources, rightsizing opportunities, over-provisioned VMs
- 🔧 **Operational Health**: Backup status, snapshot age, service availability, performance metrics
- 📊 **Performance Monitoring**: Resource utilization, bottlenecks, capacity planning

**Scheduled Scan Flow**:

```
2am: Scheduler triggers daily scan (EventBridge/Logic Apps/Cloud Scheduler)
↓
Extend My Laptop wakes up (or Physical Laptop if Local Only and online)
↓
Execute scans across all clouds in parallel:
├─ AWS Cost Explorer API (yesterday's costs, budget status)
├─ Azure Cost Management API (spending trends, anomalies)
├─ GCP Billing API (cost breakdown, forecasts)
├─ Security scans (public resources, encryption validation, IAM analysis)
├─ Performance metrics (CloudWatch, Azure Monitor, GCP Monitoring)
└─ Resource inventory (idle instances, old snapshots, unattached volumes)
↓
Load RAG from S3/Blob/GCS:
├─ Estate: Current resource inventory for comparison
├─ Previous Scans: Yesterday's results to calculate deltas
├─ Alert Rules: User's customized thresholds (e.g., alert if cost > $100 increase)
├─ Immutable Reports: Historical cost/security data for trend analysis
└─ Report Templates: User's customized morning report preferences
↓
Send scan results + context to AI Server
↓
AI Server analyzes:
├─ Aggregate findings (group similar issues: "5 idle instances" not "Instance 1 idle, Instance 2...")
├─ Calculate deltas (what changed since yesterday?)
├─ Prioritize by severity and cost impact (sort by potential savings)
├─ Generate actionable recommendations (with 1-click fix options)
├─ Create interactive morning report using user's template
└─ Format for conversational Q&A (enable "ask me anything" on report)
↓
Store report in Immutable Reports (Alerts & Events collection):
├─ Acts as aggregated data for historical queries and comparisons
├─ Permanent storage (not deleted after 7 days)
├─ Enables: "Compare this week vs last week" queries
└─ Enables: "Show me cost trends over 3 months" analysis
↓
Extend My Laptop uploads RAG to S3/Blob/GCS → Shuts down
↓
When user opens laptop:
Display interactive morning report banner with AI-powered Q&A
```

**Interactive Morning Report** (Better Than Email):

```
☀️ Good Morning Report - March 15, 2025
Generated at 2:00 AM | Data current as of 11:59 PM yesterday

────────────────────────────────────────────────────────────

🚨 CRITICAL ALERTS (Last 24h):
[Highlighted in red background, requires immediate attention]

• Production RDS instance exceeded 90% storage capacity
  └─ Auto-scaled from 100GB → 150GB ✅ (Cost impact: +$7.50/month)
  └─ Root cause: Log retention increased from 7 to 30 days

• Security group sg-abc123 opened port 22 to 0.0.0.0/0
  └─ Auto-remediated: Restricted to company IP range ✅
  └─ Alert sent to: security@company.com

────────────────────────────────────────────────────────────

💰 COST SUMMARY:
Yesterday: $1,247 | This Month (MTD): $18,705 | Budget: $25,000

+$186 (+17.5%) vs previous day 🔴 Above your threshold ($100)
+$2,450 (+15%) vs last month 🟡 Trending higher

Top Cost Drivers (Yesterday):
1. EC2 Instances: $567 (+$144 from 3 new m5.2xlarge in production)
2. RDS: $289 (+$25 from storage auto-scaling)
3. S3 Storage: $156 (+$12 from new backups)

💡 Potential Savings Identified: $412/month

────────────────────────────────────────────────────────────

📊 TOP CHANGES (Requires Your Attention):

1. 🔴 3 new EC2 instances launched in production
   • Instance Type: m5.2xlarge (8 vCPU, 32GB RAM)
   • Cost Impact: +$144/day ($4,320/month)
   • Launched by: john@company.com at 10:34 AM
   • Purpose (from tags): "web-tier-scaling"
   • Status: Currently running

   💬 Ask: "Why were these instances created?"
   💬 Ask: "Are these still needed?"
   💬 Ask: "Can we use spot instances instead?"

2. 🟡 RDS snapshot storage increased 25GB
   • New Size: 125GB (+25GB from yesterday)
   • Cost Impact: +$2.50/day
   • Reason: Daily automated snapshots accumulating
   • Current: 42 snapshots (retention: 30 days)

   💬 Ask: "Can we reduce snapshot retention to 7 days?"
   💬 Ask: "How much will I save?"
   🔧 1-Click: Reduce retention to 7 days (saves $18/month)

────────────────────────────────────────────────────────────

🔒 SECURITY & COMPLIANCE:

✅ GOOD NEWS:
• No public S3 buckets detected
• All production RDS instances encrypted
• IAM password policy compliant

⚠️ ATTENTION REQUIRED:
• 2 unencrypted EBS volumes detected
  └─ Environment: dev-environment
  └─ Volumes: vol-abc123 (50GB), vol-def456 (100GB)
  └─ Risk: Medium (dev data, but may contain test PII)

  💬 Ask: "Show me these volumes"
  💬 Ask: "What data is on them?"
  🔧 1-Click: Enable encryption (creates encrypted copy, migrates data)

────────────────────────────────────────────────────────────

⚡ OPTIMIZATION OPPORTUNITIES:

💡 5 idle EC2 instances detected (Potential savings: $203/month)
• Criteria: CPU < 5% for 7 consecutive days
• Instances: i-abc123, i-def456, i-ghi789, i-jkl012, i-mno345
• Environment: dev (3), staging (2)

💬 Ask: "Which instances are idle?"
💬 Ask: "What are they used for?"
💬 Ask: "Is it safe to stop them?"
🔧 Stop All Idle Instances | 🔧 Stop Dev Only | ℹ️ Remind Me Tomorrow

💡 3 over-provisioned VMs (Potential savings: $142/month)
• Criteria: Average utilization < 30% over 30 days
• Recommendations:
  - vm-web-01: m5.2xlarge → m5.xlarge (save $72/month)
  - vm-api-02: m5.2xlarge → m5.xlarge (save $72/month)
  - vm-db-staging: db.m5.large → db.t3.medium (save $28/month)

💬 Ask: "Show me utilization graphs"
💬 Ask: "Will downsizing impact performance?"
🔧 View Rightsizing Recommendations

💡 67GB of old EBS snapshots (Potential savings: $6.70/month)
• Criteria: Snapshots older than 90 days, source volume deleted
• Count: 13 snapshots
• Total Size: 67GB

💬 Ask: "Are these snapshots safe to delete?"
🔧 Delete Old Snapshots

────────────────────────────────────────────────────────────

🔧 OPERATIONAL HEALTH:

✅ ALL SYSTEMS OPERATIONAL:
• All backups successful (42 RDS snapshots, 18 EC2 AMIs)
• No failed deployments
• Service availability: 100% uptime

⚠️ MAINTENANCE RECOMMENDED:
• 1 RDS snapshot older than 30 days (snap-abc123, 5GB, $0.50/month)
  └─ Recommendation: Delete or archive to S3 Glacier (90% cost reduction)

────────────────────────────────────────────────────────────

💬 ASK ME ANYTHING ABOUT THIS REPORT:

Examples:
• "Why did spending increase 17%?"
• "Show me the 5 idle instances"
• "Which optimizations should I do first?"
• "Create a playbook to fix all security issues"
• "Compare costs with last week"
• "Show me cost breakdown by environment"

Type your question below ↓
```

**Interactive Q&A on Report**:

User can ask questions about the report conversationally. The AI Server has the complete report context and can answer any question:

```
User: "Why did spending increase 17%?"

AI Response:
"Spending increased by $186 (17.5%) yesterday due to three main factors:

1. **3 new EC2 instances in production** (+$144/day = 77% of increase)
   - Instance IDs: i-0a1b2c3, i-4d5e6f7, i-8g9h0i1
   - Type: m5.2xlarge (8 vCPU, 32GB RAM) - $0.384/hour each
   - Launched: Yesterday 10:34 AM by john@company.com
   - Purpose (from tags): "web-tier-scaling" for Black Friday prep
   - Running time: 13.5 hours = $15.60 each = $46.80 total (but daily cost will be $144)

2. **RDS storage auto-scaling** (+$25/day = 13% of increase)
   - Database: prod-db-01 scaled from 100GB → 150GB
   - Trigger: 90% capacity threshold reached
   - Root cause: Log retention changed from 7 → 30 days (3.2GB logs/day)
   - Cost: $0.115/GB-month = $2.50/day additional

3. **Increased S3 storage** (+$12/day = 6% of increase)
   - New backup policy storing application logs to S3
   - Data ingested: 960GB over last 24 hours
   - Cost: $0.023/GB-month = $12/day

💡 **To reduce costs**:
- If Black Friday prep is complete, stop the 3 EC2 instances (saves $144/day)
- Reduce RDS log retention to 14 days (saves $10/day)
- Enable S3 Intelligent-Tiering for logs (saves ~30% = $3.60/day)

**Total potential daily savings: $157.60 ($4,728/month)**

Would you like me to:
1. Check if the EC2 instances are still needed (analyze current load)?
2. Create a playbook to optimize these costs?
3. Schedule automatic shutdown of instances after business hours?"
```

**Report Customization** (Template Models):

**Default Template** (Escher provides):
- Comprehensive morning report (shown above)
- Includes: Critical alerts, cost summary, security, optimization, operational health
- Format: Concise, scannable, actionable
- Q&A enabled by default

**User Customization Options**:
```
Report Settings:
├─ Sections to Include:
│   ☑️ Critical Alerts (always included)
│   ☑️ Cost Summary
│   ☑️ Top Changes
│   ☑️ Security & Compliance
│   ☑️ Optimization Opportunities
│   ☑️ Operational Health
│   ☐ Performance Metrics (optional, adds detailed graphs)
│
├─ Thresholds:
│   • Cost increase alert: >$100 or >10% (customizable)
│   • Idle instance: <5% CPU for 7 days (customizable)
│   • Old snapshot: >90 days (customizable)
│
├─ Focus Areas:
│   ○ Balanced (default - equal weight to all areas)
│   ○ Cost-Focused (emphasize savings opportunities)
│   ○ Security-Focused (emphasize compliance and vulnerabilities)
│   ○ Operational-Focused (emphasize uptime and performance)
│
├─ Format:
│   ○ Detailed (default - shown above, ~50 lines)
│   ○ Compact (summary only, ~20 lines)
│   ○ Executive (high-level metrics + top 3 issues, ~15 lines)
│
└─ Severity Customization:
    • Define what's "CRITICAL" for your organization:
      ☑️ Any public S3 bucket
      ☑️ Budget overrun >$1000
      ☐ Any unencrypted volume (default: only production)
      ☑️ Production database >85% capacity
```

**Template Storage**: Stored in RAG (Alerts & Events collection), synced across devices

**Deployment Model Differences**:
- **Local Only**: Scans run on physical laptop (must be online at 2am or miss that day's report)
- **Extend My Laptop**: Scans run in cloud reliably every day, report ready when user wakes up

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
  5. **Alerts & Events**: Alert rules, alert history, scan results, auto-remediation settings, report templates, morning reports
- **Purpose**: Provides context to AI Server queries, enables offline operation documentation
- **Storage**:
  - **Local Only**: Local vector store on laptop + periodic backup snapshots to S3/Blob/GCS (hourly, configurable)
  - **Extend My Laptop**: S3/Blob/GCS vector store (single source of truth)

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
- **Syncs**:
  - Cost data (AWS Cost Explorer, Azure Cost Management, GCP Billing APIs)
  - Audit logs (all operations executed)
  - Compliance reports (CIS benchmarks, policy violations)
  - **Security scans** (public resources, encryption status, IAM analysis)
  - **Idle resource detection** (unused instances, volumes, snapshots)
  - **Performance monitoring** (resource utilization, bottlenecks)
  - **Interactive morning report generation** (aggregated insights with AI-powered Q&A)
- **Benefit**: Manager wakes up to fresh data and actionable morning report without manual refresh
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

### 9. **Alerts & Monitoring**

**Real-Time Operational Alerts** (🚨 Can't Wait):
- Critical event detection (production outages, security breaches, budget overruns)
- Cloud-native alert sources (CloudWatch, Azure Monitor, GCP Monitoring)
- Escher listener added to source of truth (EventBridge/Event Grid/Pub Sub)
- Auto-remediation with pre-approved first aid options (configured during installation)
- Multi-channel notifications (email, SMS, Slack, PagerDuty) via cloud-native services
- Severity-based routing (Critical → immediate, High → email, Medium → in-app, Info → morning report)
- Unified event schema for cross-cloud normalization

**Scheduled Scan Alerts** (ℹ️ Can Wait - Morning Report):
- Daily proactive scans (cost, security, optimization, operational health)
- Interactive morning report (better than email - ask questions on report)
- AI-powered aggregation and prioritization
- Template-based customization (default provided, user can modify)
- Actionable recommendations with 1-click fixes
- Permanent storage in vector store for historical queries and comparisons

**Implementation**:
- **Real-Time**: Event subscriptions at source (EventBridge/Event Grid/Pub Sub) → Extend My Laptop
- **Scheduled**: Daily scans at 2am (configurable), results in morning report
- **Storage**: Alerts & Events collection in RAG (alert rules, history, templates, morning reports)
- **Notification**: Cloud-native services (SNS, Notification Hubs, Cloud Messaging)
- **Auto-remediation**: Pre-approved during installation, modifiable in settings
- **Local Only Limitation**: Requires laptop always-on for real-time alerts

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
  - **Local Only**: Stored on physical laptop + periodic backups to S3/Blob/GCS (hourly, configurable)
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
- **Daily sync for Manager persona** (cost + audit logs + security scans + morning report at 2am)
- **Client-side RAG** with 5 collections (estate, chat, operations, immutable reports, **alerts & events**)
- **Server-side RAG** with playbook library and cloud operations knowledge
- **V2 release plan** for optional central immutable reports
- **Alert & Event Handling**:
  - Real-time operational alerts (Escher listener at source, auto-remediation, cloud-native notifications)
  - Scheduled scan alerts (daily scans, interactive morning report with AI-powered Q&A)
  - Unified event schema for cross-cloud normalization
  - Permanent storage in vector store for historical analysis

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