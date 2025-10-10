# AWS CloudOps AI Agent - Architecture with Tauri + Local AWS Estate

## **Updated Architecture Overview**

```
┌──────────────────────────────────────────────────────────────────┐
│                    CLIENT (Tauri App)                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Frontend (React/Vue/Svelte)                           │    │
│  │  • Chat UI                                             │    │
│  │  • Resource Explorer                                   │    │
│  │  • Execution Monitor                                   │    │
│  └────────────────────────────────────────────────────────┘    │
│                          ↕                                      │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Rust Backend (Tauri Core)                            │    │
│  │  • Request Builder                                     │    │
│  │  • Resource Lookup Engine                             │    │
│  │  • AWS CLI Executor                                   │    │
│  │  • Workflow Engine                                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                          ↕                                      │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  LOCAL STORAGE LAYER                                   │    │
│  │                                                         │    │
│  │  ┌──────────────────┐  ┌──────────────────────────┐  │    │
│  │  │ Qdrant           │  │ AWS Estate Cache         │  │    │
│  │  │ (Vector DB)      │  │                          │  │    │
│  │  │                  │  │ Resources:               │  │    │
│  │  │ • Resource       │  │ • EC2: 2,500 instances  │  │    │
│  │  │   embeddings     │  │ • RDS: 150 databases    │  │    │
│  │  │ • Semantic       │  │ • S3: 430 buckets       │  │    │
│  │  │   search         │  │ • Lambda: 85 functions  │  │    │
│  │  │ • Fast lookup    │  │ • ELB: 22 balancers     │  │    │
│  │  │                  │  │                          │  │    │
│  │  │ Indexed:         │  │ Metadata per resource:  │  │    │
│  │  │ • Names          │  │ • ID, name, type        │  │    │
│  │  │ • Tags           │  │ • Account, region       │  │    │
│  │  │ • Descriptions   │  │ • State, config         │  │    │
│  │  │ • Resource IDs   │  │ • Tags, labels          │  │    │
│  │  └──────────────────┘  │ • Permissions           │  │    │
│  │                        │ • Dependencies          │  │    │
│  │                        │ • Last updated          │  │    │
│  │                        └──────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────┘    │
│                          ↕                                      │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  AWS Credentials (Secure Storage)                      │    │
│  │  • OS Keychain                                         │    │
│  │  • Multiple AWS accounts                               │    │
│  │  • Never sent to server                                │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                              ↕
                         HTTPS/REST
                              ↕
┌──────────────────────────────────────────────────────────────────┐
│                         SERVER                                   │
├──────────────────────────────────────────────────────────────────┤
│  • Classification Agent + RAG                                    │
│  • AWS Operations Agent                                          │
│  • Playbook Repository (Git)                                     │
│  • Metadata Database (Redis)                                │
│  • Global Vector DB (Qdent) - for playbooks                  │
│  • NO AWS credentials                                            │
│  • NO AWS estate data stored                                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## **Key Architecture Changes**

### **1. Tauri Benefits**

| Aspect | Electron | Tauri | Impact |
|--------|----------|-------|--------|
| **Size** | ~100MB | ~3-5MB | 95% smaller |
| **Performance** | Node.js | Rust | Faster execution |
| **Memory** | ~100-200MB | ~30-50MB | 70% less RAM |
| **Security** | JS isolation | Rust + OS sandbox | More secure |
| **Local DB** | Better Node support | Better Rust support | Qdrant native |

### **2. Local Qdrant Vector Database**

**Purpose:** Fast semantic search over AWS resources + chat history storage

**Why Qdrant locally?**
- Semantic search: "production database" → finds pg-instance-main1
- Tag matching: "env=prod AND app=main" → finds related resources
- Name variations: "pg-main1" → finds "pg-instance-main1"
- Offline capability: Works without internet
- Privacy: AWS estate never leaves client

**Architecture - Dual Collection Strategy:**

The Storage Service uses a single Qdrant instance with two collections optimized for different use cases:

```
Local Qdrant Instance (Embedded mode, ~20-30 MB)

┌─────────────────────────────────────────────────────────────────┐
│ Collection 1: "chat_history"                                     │
├─────────────────────────────────────────────────────────────────┤
│ Purpose: Store conversation history                              │
│ Vectors: 1D dummy vectors (filter-based access, not semantic)    │
│ Point IDs: Random UUIDs (immutable, prevents conflicts)          │
│ Encryption: AES-256-GCM for message content                      │
│                                                                   │
│ Example Point:                                                    │
│ {                                                                 │
│   "id": "uuid-v4-random",                                         │
│   "vector": [0.0],  // 1D dummy vector                           │
│   "payload": {                                                    │
│     "conversation_id": "conv-123",                                │
│     "user_id": "user-456",                                        │
│     "timestamp": 1696800000,                                      │
│     "role": "user",                                               │
│     "encrypted_content": "AES-256-GCM encrypted message"          │
│   }                                                               │
│ }                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Collection 2: "aws_estate"                                       │
├─────────────────────────────────────────────────────────────────┤
│ Purpose: Store AWS resources with semantic search                │
│ Vectors: 384D real embeddings (all-MiniLM-L6-v2 model)           │
│ Point IDs: Deterministic {account}-{region}-{identifier}         │
│ Encryption: AES-256-GCM for sensitive data (IAM, metadata)       │
│                                                                   │
│ Example Point:                                                    │
│ {                                                                 │
│   "id": "123456789012-us-west-2-i-0123456789abcdef0",            │
│   "vector": [0.123, 0.456, ...],  // 384D embedding              │
│   "payload": {                                                    │
│     // Plain text (indexed for filtering)                        │
│     "resource_type": "ec2_instance",                              │
│     "account_id": "123456789012",                                 │
│     "region": "us-west-2",                                        │
│     "state": "running",                                           │
│     "tags": {"env": "production", "app": "api"},                  │
│     "last_synced": 1696800000,                                    │
│                                                                   │
│     // Encrypted sensitive data                                  │
│     "encrypted_data": {                                           │
│       "identifier": "i-0123456789abcdef0",                        │
│       "arn": "arn:aws:ec2:...",                                   │
│       "name": "web-server-1",                                     │
│       "iam": {                                                    │
│         "allowed_actions": ["ec2:StopInstances", ...],            │
│         "denied_actions": ["ec2:TerminateInstances"],             │
│         "user_context": {                                         │
│           "username": "john.doe",                                 │
│           "role_arn": "arn:aws:iam::...:role/Developer"           │
│         }                                                         │
│       },                                                          │
│       "constraints": {                                            │
│         "can_stop": true,                                         │
│         "can_start": false,                                       │
│         "can_delete": false                                       │
│       },                                                          │
│       "metadata": { /* service-specific */ }                      │
│     }                                                             │
│   }                                                               │
│ }                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key Design Decisions:**
- **Deterministic IDs for Estate**: Same resource always gets same ID → upserts update existing points (no duplicates)
- **Random IDs for Chat**: Immutable conversations → prevents conflicts
- **Encryption**: AES-256-GCM with OS Keychain (macOS/Windows/Linux)
- **IAM per Resource**: Every resource stores its allowed/denied actions for current user
- **Embedding Model**: all-MiniLM-L6-v2 (384 dimensions, fast, accurate)

---

## **Enhanced Request Flow Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: LOCAL RESOURCE DISCOVERY (Client)                    │
└─────────────────────────────────────────────────────────────────┘

User: "Stop pg-instance-main1"
  ↓
┌─────────────────────────────────────────────────────────────┐
│ Client - Semantic Search in Local Qdrant                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Generate embedding: embed("pg-instance-main1")            │
│                                                             │
│ Search Qdrant:                                             │
│ • Query: "pg-instance-main1"                               │
│ • Filters: None (search all resource types)               │
│                                                             │
│ Results:                                                    │
│ 1. rds-pg-instance-main1 (score: 0.99) ← Perfect match   │
│ 2. ec2-pg-backup-server (score: 0.45)                     │
│ 3. s3-pg-backups-bucket (score: 0.42)                     │
│                                                             │
│ Selected: rds-pg-instance-main1                            │
│                                                             │
│ Extract full metadata from payload:                         │
│ • Resource type: RDS                                        │
│ • Account: 123456789012                                    │
│ • Region: us-west-2                                        │
│ • Current state: available                                  │
│ • Permissions: [rds:StopDBInstance, ...]                   │
│ • Constraints: {can_stop: true, has_replicas: false}       │
└─────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2: BUILD ENRICHED REQUEST (Client)                   │
└─────────────────────────────────────────────────────────────┘

Client packages complete context:
  ↓
Request Payload:
{
  "prompt": "Stop pg-instance-main1",
  
  "identified_resources": [
    {
      "resource_type": "rds_instance",
      "db_instance_identifier": "pg-instance-main1",
      "account_id": "123456789012",
      "region": "us-west-2",
      "current_state": "available",
      "engine": "postgres",
      "engine_version": "14.7",
      "multi_az": true,
      "tags": {"environment": "production"},
      "available_permissions": ["rds:StopDBInstance", ...],
      "constraints": {
        "can_stop": true,
        "has_read_replicas": false,
        "automated_backups_enabled": true
      }
    }
  ],
  
  "user_context": {
    "user_id": "user-123",
    "default_region": "us-west-2",
    "aws_accounts": ["123456789012"]
  }
}
  ↓
Send to Server
  ↓
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3: SERVER PROCESSING                                 │
└─────────────────────────────────────────────────────────────┘

Server receives COMPLETE context (no guessing needed)
  ↓
┌─────────────────────────────────────────────────────────────┐
│ Classification Agent                                        │
├─────────────────────────────────────────────────────────────┤
│ • Intent: "Stop RDS instance"                              │
│ • Confidence: 0.99 (resource already identified)           │
│ • System: AWS RDS                                          │
│ • Operation: Stop                                           │
│ • Route to: AWS Operations Agent                           │
└─────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────┐
│ RAG Search (Server's Playbook Vector DB)                   │
├─────────────────────────────────────────────────────────────┤
│ Query: "stop rds instance"                                  │
│ Filters: {system: "aws", service: "rds"}                   │
│                                                             │
│ Results:                                                    │
│ 1. aws_rds_stop_instance (score: 0.95)                    │
│ 2. aws_rds_stop_with_snapshot (score: 0.89)               │
│ 3. aws_rds_reboot_instance (score: 0.45)                  │
└─────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────┐
│ AWS Operations Agent                                        │
├─────────────────────────────────────────────────────────────┤
│ Load playbook: aws_rds_stop_instance.yaml                  │
│                                                             │
│ Parameter Resolution:                                       │
│ • db_instance_identifier: "pg-instance-main1"              │
│   (from identified_resources)                              │
│ • region: "us-west-2"                                      │
│   (from identified_resources)                              │
│ • account: "123456789012"                                  │
│   (from identified_resources)                              │
│                                                             │
│ ALL PARAMETERS PRE-FILLED - NO AMBIGUITY                   │
│                                                             │
│ Risk Assessment:                                            │
│ • Production database (from tags)                          │
│ • Currently available (can be stopped)                     │
│ • No read replicas (safe to stop)                          │
│ • Multi-AZ enabled (consider impact)                       │
│ • Risk: Medium                                              │
│ • Approval: Required                                        │
└─────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────┐
│  PHASE 4: SERVER RESPONSE                                   │
└─────────────────────────────────────────────────────────────┘

Server → Client:
{
  "explain_plan": "I will stop the RDS PostgreSQL instance 
                   'pg-instance-main1' in us-west-2. This is 
                   a production Multi-AZ database. The instance 
                   will be unavailable but can be restarted 
                   at any time.",
  
  "script": {
    "steps": [
      {
        "step_id": "1",
        "command": "aws rds stop-db-instance",
        "args": [
          "--db-instance-identifier", "pg-instance-main1",
          "--region", "us-west-2"
        ]
        // ALL PARAMETERS FILLED - READY TO EXECUTE
      }
    ]
  },
  
  "risk_assessment": {
    "level": "medium",
    "reasons": [
      "Production database",
      "Multi-AZ enabled",
      "No read replicas to affect"
    ],
    "impact": "Database will be unavailable until restarted"
  },
  
  "approval_required": true
}
  ↓
┌─────────────────────────────────────────────────────────────┐
│  PHASE 5: CLIENT EXECUTION                                  │
└─────────────────────────────────────────────────────────────┘

Client receives ready-to-execute script
  ↓
Display to user + Request approval
  ↓
User approves
  ↓
Execute: aws rds stop-db-instance --db-instance-identifier 
         pg-instance-main1 --region us-west-2
  ↓
Show result to user
```

---

## **Data Synchronization Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│  AWS ESTATE SYNC SYSTEM (Client)                               │
│  Powered by Estate Scanner Module                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ESTATE SCANNER - THIN ORCHESTRATOR                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Architecture: Coordinates Execution Engine + Storage Service│
│                                                             │
│  Sync Strategy:                                            │
│  • Full sync: Every 6 hours (or on-demand)                │
│  • Incremental sync: Every 15 minutes                      │
│  • Event-driven sync: On AWS CloudWatch events (optional) │
│                                                             │
│  4-Level Parallelism:                                      │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Level 1: Accounts (parallel)                         │ │
│  │   ├─ Production Account (123456789012)              │ │
│  │   ├─ Staging Account (987654321098)                 │ │
│  │   └─ Development Account (456789012345)             │ │
│  │                                                       │ │
│  │ Level 2: Regions (parallel per account)              │ │
│  │   ├─ us-east-1                                       │ │
│  │   ├─ us-west-2                                       │ │
│  │   └─ eu-west-1                                       │ │
│  │                                                       │ │
│  │ Level 3: Services (parallel per region)              │ │
│  │   ├─ EC2Scanner                                      │ │
│  │   ├─ RDSScanner                                      │ │
│  │   ├─ S3Scanner                                       │ │
│  │   ├─ LambdaScanner                                   │ │
│  │   └─ VPCScanner                                      │ │
│  │                                                       │ │
│  │ Level 4: Resources (batch per service)               │ │
│  │   └─ Process 100 resources at a time                 │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Pluggable Scanner Trait:                                  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ trait ServiceScanner {                               │ │
│  │   async fn scan(&self) -> Vec<AWSResource>;         │ │
│  │   async fn discover_iam(&self, arn) -> IAMPerms;    │ │
│  │   async fn analyze_constraints(&self) -> Constraints;│ │
│  │ }                                                     │ │
│  │                                                       │ │
│  │ Implementations:                                      │ │
│  │ • EC2Scanner - Instances, volumes, snapshots         │ │
│  │ • RDSScanner - DB instances, clusters, snapshots     │ │
│  │ • S3Scanner - Buckets, lifecycle policies            │ │
│  │ • LambdaScanner - Functions, layers, aliases         │ │
│  │ • VPCScanner - VPCs, subnets, security groups        │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Sync Process:                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 1. Execute AWS CLI via Execution Engine              │ │
│  │    • Returns JSON in stdout                          │ │
│  │    • Background execution with streaming             │ │
│  └──────────────────────────────────────────────────────┘ │
│               ↓                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 2. Transform to AWSResource format                   │ │
│  │    • Parse AWS CLI JSON output                       │ │
│  │    • Normalize structure across services             │ │
│  │    • Build ARN, extract tags                         │ │
│  └──────────────────────────────────────────────────────┘ │
│               ↓                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 3. Discover IAM permissions                          │ │
│  │    • Call IAM SimulatePrincipalPolicy API            │ │
│  │    • Parse allowed/denied actions                    │ │
│  │    • Embed user context (role ARN, session)          │ │
│  └──────────────────────────────────────────────────────┘ │
│               ↓                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 4. Analyze constraints                               │ │
│  │    • Check resource state (can_stop, can_start)      │ │
│  │    • Check dependencies (has_dependencies)           │ │
│  │    • Check backups (has_backups)                     │ │
│  └──────────────────────────────────────────────────────┘ │
│               ↓                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 5. Generate embeddings                               │ │
│  │    • Model: all-MiniLM-L6-v2 (384D)                  │ │
│  │    • Input: name + tags + identifier                 │ │
│  │    • Output: [0.123, 0.456, ...] (384 floats)        │ │
│  └──────────────────────────────────────────────────────┘ │
│               ↓                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 6. Upsert to Storage Service                         │ │
│  │    • Generate deterministic point ID                 │ │
│  │    • Encrypt sensitive data (AES-256-GCM)            │ │
│  │    • Batch upsert to Qdrant                          │ │
│  │    • Remove deleted resources                        │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  Graceful Degradation:                                     │
│  • Service scan failure → Continue with other services    │
│  • IAM discovery failure → Store with empty permissions   │
│  • Embedding failure → Retry or skip resource             │
│                                                             │
│  Performance:                                               │
│  • ~30-50s for 1000 resources                              │
│  • Concurrency limit: 10 parallel scans                    │
│  • Batch size: 100 resources per upsert                    │
│                                                             │
│  Sync Status Tracking:                                     │
│  • Last sync timestamp per service                         │
│  • Sync errors and retries                                 │
│  • Resource count changes                                  │
│  • IAM discovery success rate                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  MULTI-ACCOUNT SUPPORT                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User has multiple AWS accounts:                           │
│  • Production Account (123456789012)                       │
│  • Staging Account (987654321098)                          │
│  • Development Account (456789012345)                      │
│                                                             │
│  Qdrant Structure:                                         │
│  {                                                          │
│    "id": "rds-prod-pg-instance-main1",                     │
│    "payload": {                                             │
│      "account_id": "123456789012",                         │
│      "account_name": "production",                         │
│      ...                                                    │
│    }                                                        │
│  }                                                          │
│                                                             │
│  Search with account filter:                               │
│  • Default: Search all accounts                            │
│  • Explicit: "Stop pg-main1 in production account"        │
│  • Filter by context                                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  IAM PERMISSION DISCOVERY                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                             │
│  Estate Scanner discovers per-resource IAM permissions:     │
│                                                             │
│  Flow:                                                      │
│  1. Get current user/role ARN (via STS GetCallerIdentity)  │
│  2. For each discovered resource:                          │
│     a. Determine relevant actions for resource type        │
│        (e.g., ec2:StopInstances, ec2:StartInstances)      │
│     b. Call IAM SimulatePrincipalPolicy API                │
│     c. Parse allowed/denied actions from response          │
│     d. Create IAMPermissions struct                        │
│  3. Embed IAMPermissions in AWSResource                    │
│  4. Store encrypted in Qdrant                              │
│                                                             │
│  Result: Every resource knows what actions user can take   │
│                                                             │
│  Example IAM Discovery:                                     │
│  Resource: arn:aws:ec2:us-west-2:123456789012:instance/i-123│
│  User: arn:aws:iam::123456789012:role/DeveloperRole       │
│                                                             │
│  Actions checked:                                           │
│  • ec2:DescribeInstances → Allowed                        │
│  • ec2:StartInstances → Allowed                           │
│  • ec2:StopInstances → Allowed                            │
│  • ec2:RebootInstances → Allowed                          │
│  • ec2:TerminateInstances → Denied (explicit)             │
│                                                             │
│  Stored in resource payload:                               │
│  {                                                          │
│    "iam": {                                                 │
│      "allowed_actions": [                                   │
│        "ec2:DescribeInstances",                            │
│        "ec2:StartInstances",                               │
│        "ec2:StopInstances",                                │
│        "ec2:RebootInstances"                               │
│      ],                                                     │
│      "denied_actions": ["ec2:TerminateInstances"],         │
│      "user_context": {                                      │
│        "username": "john.doe",                             │
│        "role_arn": "arn:aws:iam::...:role/Developer",     │
│        "session_expires": "2025-10-10T23:59:59Z"          │
│      }                                                      │
│    }                                                        │
│  }                                                          │
│                                                             │
│  Benefits:                                                  │
│  • Server receives pre-validated permissions               │
│  • Client can show/hide actions based on permissions       │
│  • No need to check IAM at request time                    │
│  • Supports role assumption and session tokens             │
└─────────────────────────────────────────────────────────────┘
```

---

## **Benefits of This Architecture**

### **1. Precision**

| Without Local Estate | With Local Estate |
|---------------------|-------------------|
| Server guesses parameters | All parameters known upfront |
| "Which account?" "Which region?" | Account + region pre-filled |
| Ambiguous resource names | Exact resource identification |
| Multiple back-and-forth | Single request-response |

### **2. Performance**

| Aspect | Impact |
|--------|--------|
| **Resource lookup** | Instant (local Qdrant) |
| **No guessing** | Faster server processing |
| **Fewer LLM calls** | Lower cost |
| **Offline search** | Works without internet |

### **3. Security**

| Aspect | Benefit |
|--------|---------|
| **AWS estate stays local** | Never sent to server (privacy) |
| **Credentials local** | Never leave client |
| **Fine-grained permissions** | Known per-resource |
| **Audit trail** | Local execution history |

### **4. User Experience**

| Feature | UX Benefit |
|---------|-----------|
| **Fuzzy search** | "pg-main" finds "pg-instance-main1" |
| **Smart suggestions** | Autocomplete resource names |
| **Visual explorer** | Browse AWS estate locally |
| **Fast response** | No waiting for server lookup |

---

## **Architecture Comparison**

### **Old (Stateless Client)**

```
User: "Stop pg-instance-main1"
  ↓
Client → Server: "Stop pg-instance-main1"
  ↓
Server: "Which account? Which region? Is this RDS or EC2?"
  ↓
Client ← Server: Clarifying questions
  ↓
User provides more info
  ↓
Client → Server: Updated request
  ↓
Server generates script
```

**Issues:** Multiple round-trips, ambiguity, slow

---

### **New (Stateful Client with Local Estate)**

```
User: "Stop pg-instance-main1"
  ↓
Client searches Qdrant: Found RDS in us-west-2, account 123456
  ↓
Client → Server: Complete context (resource + metadata)
  ↓
Server generates precise script (no ambiguity)
  ↓
Client ← Server: Ready-to-execute script
  ↓
Client executes immediately
```

**Benefits:** Single round-trip, precise, fast

---

## **Key Architecture Decisions**

| Decision | Rationale |
|----------|-----------|
| **Tauri over Electron** | Smaller, faster, more secure, better for Qdrant |
| **Qdrant local vector DB** | Semantic search, fuzzy matching, offline capability |
| **AWS estate on client** | Privacy, speed, precision, offline mode |
| **Enriched context to server** | Server gets exact parameters, no guessing |
| **Server stays stateless** | Scalable, no estate data storage |
| **Periodic sync** | Fresh data without constant API calls |

---

## **Component Breakdown**

### **Client Modules**

For complete module documentation, see [Client Modules Overview](../../docs/02-client/modules/overview.md).

| Module | Status | Description |
|--------|--------|-------------|
| **[Storage Service](../../docs/02-client/modules/storage-service/)** | ✅ Complete | Single Qdrant instance, dual collections, IAM integration, S3 backup |
| **[Execution Engine](../../docs/02-client/modules/execution-engine/)** | ✅ Complete | Pure Rust crate, Tokio + streaming, background execution |
| **[Estate Scanner](../../docs/02-client/modules/estate-scanner/)** | ✅ Complete | Thin orchestrator, pluggable scanners, IAM discovery, semantic embeddings |
| **[Common Types](../../docs/02-client/modules/common/)** | ✅ Complete | Shared data structures (AWSResource, IAMPermissions, etc.) |
| **[Request Builder](../../docs/02-client/modules/request-builder/)** | 🔄 Next | Context enrichment, server communication |

#### Module Details

**Storage Service** (`cloudops-storage-service`):
- Single embedded Qdrant instance (~20-30 MB)
- Dual collection strategy (chat history + AWS estate)
- Application-level encryption (AES-256-GCM + OS Keychain)
- Automatic S3 backup (7d local, 30d S3)
- IAM permissions embedded per resource

**Execution Engine** (`cloudops-execution-engine`):
- Pure Rust crate (reusable library, no framework dependencies)
- Async execution with Tokio + real-time streaming
- Background execution (returns execution_id immediately)
- Multiple strategies (serial, parallel, dependency-based)
- Command types: Script, Exec, Shell, AwsCli
- Pluggable event handlers (Tauri, WebSocket, logging)

**Estate Scanner** (`cloudops-estate-scanner`):
- Thin orchestrator (coordinates Execution Engine + Storage Service)
- Pluggable scanner traits (EC2, RDS, S3, Lambda, VPC)
- IAM discovery via SimulatePrincipalPolicy API
- Semantic embeddings (384D using all-MiniLM-L6-v2)
- 4-level parallelism (Accounts → Regions → Services → Resources)
- Performance: ~30-50s for 1000 resources

**Common Types** (`cloudops-common`):
- Shared data structures across all modules
- Core types: AWSResource, IAMPermissions, UserContext, ResourceConstraints
- Zero framework dependencies (only serde, chrono)
- Single source of truth for type safety

**Request Builder** (`cloudops-request-builder`):
- Context enrichment (adds resource metadata + chat history)
- Server communication (HTTP client)
- Request/response handling

### **Server Components**

1. **Classification Agent** - Routes requests (now easier with context)
2. **AWS Operations Agent** - Generates scripts (now with exact params)
3. **Playbook Repository** - YAML playbooks in Git
4. **Metadata DB** - Playbook metadata for RAG
5. **Vector DB** - Playbook embeddings (separate from client's resource DB)

---

## **Summary**

This architecture gives you:

✅ **Fast** - Local semantic search in Qdrant
✅ **Precise** - Server receives exact resource details
✅ **Private** - AWS estate never leaves client
✅ **Offline-capable** - Search works without internet
✅ **Smart** - Fuzzy matching, tag-based search
✅ **Scalable** - Server remains stateless
✅ **Secure** - Credentials and estate stay local

The key insight: **Client knows everything about AWS resources, server knows everything about operations**. Together they generate perfect execution scripts.

---

## **Learn More**

### Client Module Documentation

For detailed implementation guides, see:

- **[Client Modules Overview](../02-client/modules/overview.md)** - Complete module architecture
- **[Storage Service](../02-client/modules/storage-service/)** - Qdrant, encryption, backup (~4,000 lines)
- **[Execution Engine](../02-client/modules/execution-engine/)** - Command execution with Tokio (~6,000 lines)
- **[Estate Scanner](../02-client/modules/estate-scanner/)** - AWS discovery, IAM, embeddings (~3,000 lines)
- **[Common Types](../02-client/modules/common/)** - Shared data structures (~650 lines)
- **[Module Interaction Analysis](../../working-docs/MODULE-INTERACTION-ANALYSIS.md)** - Data flow between modules

### Related Documentation

- **[System Overview](system-overview.md)** - High-level system architecture
- **[Key Decisions](key-decisions.md)** - Major architectural choices
- **[Technology Stack](technology-stack.md)** - Tech choices and rationale
- **[ADRs](../../adr/)** - Architecture Decision Records