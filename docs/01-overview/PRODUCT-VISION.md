# Escher - Cloud Operations Management Platform
## Product Vision & Architecture Goals

**Last Updated**: October 2025
**Status**: Active Discussion - Defining Complete Scope

---

## Product Overview

**Escher** is a Cloud Operations AI Platform that enables users to manage AWS cloud operations through conversational interface.

### Core Philosophy
- **Conversational Management**: Natural language interface for all cloud operations
- **Client-Side State**: AWS estate, chat history, and credentials remain on customer's machine
- **Client-Side Execution**: Operations execute from client using customer's credentials
- **Privacy First**: No AWS credentials or estate data stored on server
- **AI-Powered Intelligence**: Server provides operations knowledge, recommendations, and playbook generation

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
- Start/Stop/Restart resources (EC2, RDS, Lambda, etc.)
- Resize/Scale resources (instance types, storage, compute)
- Create/Delete resources
- Configure resources (security groups, tags, settings)
- Backup/Restore operations
- Snapshot management

**Execution**: Client-side with user credentials

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

### 6. **Multi-Account Management**
- **Org-Level Visibility**: All accounts in unified view
- **Cross-Account Operations**: Batch operations across accounts
- **Consolidated Reporting**: Org-wide costs, compliance, security
- **Centralized Policy Enforcement**: Consistent policies across accounts
- **Account Governance**: Account creation, access management

**Key Questions**:
- [ ] How many accounts per customer typically?
- [ ] Role-based access control per account?
- [ ] Cross-account assume role patterns?

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
- **Conversational Queries**: "What's my biggest cost driver?", "Show me all public S3 buckets"
- **Smart Recommendations**: AI suggests optimizations based on usage patterns
- **Anomaly Detection**: Unusual spending, security events, performance issues
- **Playbook Generation**: AI creates multi-step operation plans
- **Natural Language Execution**: "Stop all dev instances in us-east-1"
- **Context-Aware Responses**: Understands user's AWS estate and history

**Current**: Server-side AI with client-side context enrichment

---

## Architecture Questions to Resolve

### 🔴 **Critical - Execution Model**

#### 1. Immediate Operations (User-Initiated)
- ✅ **DECIDED**: Client executes with stored credentials
- ✅ Already documented in client architecture

#### 2. Scheduled Operations (Automated)
- ❓ **OPEN**: Who executes when client is offline?
  - **Option A**: Client must be online (scheduled tasks run from client)
  - **Option B**: Server executes with AssumeRole (temporary credentials)
  - **Option C**: Hybrid (critical operations via server, others wait for client)

#### 3. Continuous Monitoring & Automated Remediation
- ❓ **OPEN**: How triggered?
  - Event-driven (CloudWatch Events)?
  - Polling from client?
  - Server-side monitoring with AssumeRole?

---

### 🟡 **Important - Data & Storage**

#### 1. Historical Data for Reports
- ❓ **OPEN**: Where stored?
  - **Option A**: Client only (limited retention)
  - **Option B**: Server (longer retention, cross-device access)
  - **Option C**: Hybrid (recent on client, historical on server)

#### 2. Cost Data
- ❓ **OPEN**: How accessed?
  - AWS Cost Explorer API from client?
  - Aggregated on server for trends?
  - Both?

#### 3. Reports Storage
- ❓ **OPEN**: Where generated and stored?
  - Client-side generation, stored locally?
  - Server-side generation, cloud storage?
  - Generate on client, backup to server?

---

### 🟢 **Moderate - Access & Permissions**

#### 1. Server AssumeRole Access
- ❓ **OPEN**: For what operations?
  - Scheduled operations only?
  - Reports generation?
  - Monitoring?
  - Or none (stay pure client-side)?

#### 2. Privacy Model
- ❓ **OPEN**: Current docs say "no credentials on server"
  - Does AssumeRole violate this?
  - Or is temporary limited access acceptable?
  - Need to clarify privacy boundaries

#### 3. Multi-Account Access
- ❓ **OPEN**: How managed?
  - Client stores credentials per account?
  - AssumeRole chains from single credential?
  - Both patterns supported?

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

### Phase 1: Define Execution Model
1. Decide on scheduled operations execution (client vs server)
2. Define AssumeRole mechanism (if needed)
3. Update privacy model documentation

### Phase 2: Define Data Architecture
1. Historical data retention strategy
2. Reports generation and storage model
3. Cost data aggregation approach

### Phase 3: Define Personas & RBAC
1. Complete persona definitions (Manager, Executor, others?)
2. Permission model per persona
3. Approval workflows
4. "Extend me" pattern details

### Phase 4: Document Complete Operations
1. List all supported cloud operations by category
2. Define which operations need approval
3. Risk levels per operation type
4. Automation boundaries

---

## Alignment Check

### ✅ **Currently Aligned**
- Cloud ops AI platform with conversational interface
- Client-side state (AWS estate, chat, credentials)
- Client-side execution for immediate operations
- Privacy-first approach (credentials don't leave client for user operations)

### ⚠️ **Needs Clarification**
- Scheduled operations execution model
- Server AssumeRole access for reports/automation
- Historical data storage location
- Reports generation architecture
- Complete persona definitions
- "Extend me" pattern implementation

### ❌ **Not Yet Documented**
- Full list of cloud operations by category
- Reports & analytics architecture
- Schedulers/automation architecture
- Personas & RBAC model
- Budget management features
- Collaboration workflows

---

**This document will be updated as we make architectural decisions.**