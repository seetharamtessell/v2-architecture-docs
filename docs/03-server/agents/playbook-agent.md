# Playbook Agent

**Role**: RAG-powered playbook search and LLM-based intelligent ranking engine

**Type**: Server-side Rust service using storage-service crate + LLM integration (Claude/GPT)

**Location**: Runs on Escher server (ECS/EC2 with EBS/EFS volume)

**Input**: Structured JSON from Master Agent (intent already classified)

**Flow**: JSON → RAG Search → LLM Ranking → Output

---

**Multi-Cloud Note**: This documentation primarily uses AWS examples (S3, RDS, EC2) for concreteness. The same architecture applies to:
- **Azure**: Blob Storage, Azure Database, Container Instances, Logic Apps
- **GCP**: Cloud Storage, Cloud SQL, Cloud Run, Cloud Scheduler

All playbooks support multi-cloud operations through the `cloud_provider` field, which accepts `["aws"]`, `["azure"]`, `["gcp"]`, or combinations like `["aws", "azure", "gcp"]`.

---

## Table of Contents

1. [What It Does](#what-it-does)
2. [Core Architecture Principles](#core-architecture-principles)
3. [Implementation Overview](#implementation-overview)
4. [Playbook Structure](#playbook-structure)
5. [Three Playbook Types](#three-playbook-types)
6. [How It Works: The Intelligence Flow](#how-it-works-the-intelligence-flow)
7. [Playbook Lifecycle & States](#playbook-lifecycle--states)
8. [Storage Strategies](#storage-strategies)
9. [S3 Storage Structure](#s3-storage-structure)
10. [Architecture](#architecture)
11. [Data Flow](#data-flow)
12. [Best Practices & Quality Validation](#best-practices--quality-validation)
13. [Related Documentation](#related-documentation)
14. [Next Steps](#next-steps)
15. [Appendix: Implementation Details](#appendix-implementation-details)
16. [Appendix: Why LLM + RAG Works Better Than Just Search](#appendix-why-llm--rag-works-better-than-just-search)

## What It Does

The Playbook Agent receives **structured JSON input** from the Master Agent (intent already classified) and performs:

1. **RAG Vector Search**: Finds candidate playbooks using semantic embeddings across Escher + Tenant libraries
2. **Intelligent Ranking**: Uses LLM (Claude/GPT) to evaluate candidates with full context and precedence rules
3. **Complete Response**: Returns ranked playbooks with explain plans, scripts, and reasoning

**Key Insight**: This agent does NOT do intent classification - that's the Master Agent's job. It receives clean, structured JSON and focuses on finding and ranking the best playbooks.

---

## Core Architecture Principles

### Normalized Storage Pattern (Database-Style Foreign Keys)

Playbooks and Scripts are **separated** like a normalized database:

```
┌─────────────────────────────────────────────────┐
│           SCRIPT LIBRARY                        │
│     (Reusable, Multi-Language)                  │
├─────────────────────────────────────────────────┤
│  scripts/create-rds-snapshot/v1.0.0/           │
│  ├── metadata.json                              │
│  │   {                                          │
│  │     "available_types": ["bash", "python"],  │
│  │     "default_type": "bash"                  │
│  │   }                                          │
│  ├── parameters.json                            │
│  └── code/                                      │
│      ├── bash/script.sh    ← User chooses      │
│      ├── python/main.py    ← at runtime        │
│      └── terraform/main.tf                      │
└─────────────────────────────────────────────────┘
                    ↑
                    │ (Foreign Key Reference)
                    │ script_ref: { script_id, version }
                    │
┌─────────────────────────────────────────────────┐
│          PLAYBOOK LIBRARY                       │
│     (Orchestration, References Only)            │
├─────────────────────────────────────────────────┤
│  playbooks/stop-rds-with-snapshot/v1.0.0/      │
│  ├── metadata.json                              │
│  ├── parameters.json                            │
│  └── orchestration.json  ← References IDs      │
│      {                                          │
│        "steps": [{                              │
│          "type": "script",                      │
│          "script_ref": {                        │
│            "script_id": "create-rds-snapshot", │
│            "version": "1.0.0"                   │
│          }                                      │
│        }]                                       │
│      }                                          │
└─────────────────────────────────────────────────┘
```

**Benefits**:
- ✅ **Script Reusability**: One script used by multiple playbooks
- ✅ **Independent Versioning**: Update script without touching playbooks
- ✅ **Multi-Implementation**: Same script in bash/python/node/terraform/cloudformation
- ✅ **User Choice**: Select implementation at execution time

**Parameter Extraction Responsibility**: The Master Agent (a separate server-side agent) performs all parameter extraction using LLM analysis of user prompts and estate context. The Playbook Agent receives these **already-extracted** parameters in structured JSON format. Neither the Playbook Agent nor the client performs parameter extraction. See Master Agent documentation for details on parameter extraction flow.

---

## Implementation Overview

This section provides a high-level mental model for implementing the Playbook Agent. Read this first to understand the complete architecture before diving into detailed implementation guidance.

### Architecture at a Glance

The Playbook Agent is a **search-only service** that helps clients find and retrieve playbooks:

- **Server Role**: Search engine that finds relevant playbooks using RAG + LLM ranking
- **Client Role**: Execution engine that runs playbooks locally on user's infrastructure
- **S3 Role**: Source of truth for all playbooks and scripts
- **Qdrant Role**: Search index with embedded playbook content (cached from S3)

**Critical Principle**: Server NEVER executes scripts or accesses user's AWS infrastructure. It only searches and returns complete playbook definitions.

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    S3 STORAGE                           │
│              (Source of Truth)                          │
│                                                          │
│  playbooks/                    scripts/                 │
│    stoprds-v1.yaml              rds/                    │
│    stoprds-v2.yaml                stop-instance.sh      │
│    rds-backup-v1.yaml             create-snapshot.sh    │
│    rds-backup-v2.yaml           ec2/                    │
│                                   create-ami.sh         │
└─────────────────────────────────────────────────────────┘
            │
            │ Manual Sync API Call
            │ (POST /api/playbooks/sync)
            ↓
┌─────────────────────────────────────────────────────────┐
│              SERVER QDRANT                              │
│          (Search Index + Cache)                         │
│                                                          │
│  Each entry contains:                                   │
│    - Vector embedding (384 dims)                        │
│    - Complete playbook JSON                             │
│    - Scripts EMBEDDED in steps                          │
│    - Metadata for filtering                             │
└─────────────────────────────────────────────────────────┘
            ↑
            │
            │
┌─────────────────────────────────────────────────────────┐
│                   CLIENT                                │
│              (Desktop App)                              │
│                                                          │
│  User: "Stop my production RDS for weekend"            │
│    ↓                                                     │
│  Collects context:                                      │
│    - Chat history                                       │
│    - Local estate scan (RDS: prod-db-01 in us-east-1)  │
│    - User preferences                                   │
│    ↓                                                     │
│  Sends to server →                                      │
└─────────────────────────────────────────────────────────┘
            │
            │ POST /api/chat
            │ { prompt, history, estate_context }
            ↓
┌─────────────────────────────────────────────────────────┐
│                  MASTER AGENT (Server)                  │
│                  (LLM Orchestrator)                     │
│                                                          │
│  1. Receives prompt + context                           │
│  2. Does intent classification                          │
│  3. EXTRACTS PARAMETERS from estate_context (LLM)       │
│       - Sees RDS instance "prod-db-01"                  │
│       - Extracts region "us-east-1"                     │
│       - Understands "weekend" = scheduled shutdown      │
│  4. Calls Playbook Agent with structured input →        │
└─────────────────────────────────────────────────────────┘
            │
            │ Structured JSON:
            │ {
            │   action: "stop",
            │   resource: "rds::instance",
            │   extracted_parameters: {
            │     instance_id: "prod-db-01",
            │     region: "us-east-1"
            │   },
            │   estate_context: {...}
            │ }
            ↓
┌─────────────────────────────────────────────────────────┐
│              PLAYBOOK AGENT (Server)                    │
│              (Search & Ranking)                         │
│                                                          │
│  1. Receives structured input from Master Agent         │
│  2. Searches Qdrant for relevant playbooks              │
│  3. LLM ranks candidates                                │
│  4. Returns playbook with PREFILLED parameters ←        │
└─────────────────────────────────────────────────────────┘
            │
            │ Response:
            │ {
            │   playbook_id: "stoprds-v2",
            │   parameters: {
            │     instance_id: "prod-db-01",  ← Already filled!
            │     region: "us-east-1"          ← Already filled!
            │   },
            │   steps: [ {script: "#!/bin/bash..."} ]
            │ }
            ↓
┌─────────────────────────────────────────────────────────┐
│                   CLIENT                                │
│            (Execution Engine)                           │
│                                                          │
│  1. Receives playbook with parameters ALREADY FILLED    │
│  2. Shows user what will be executed                    │
│  3. User confirms                                       │
│  4. Executes scripts locally (bash/python/node)         │
│  5. Streams progress to user                            │
│                                                          │
│  ✓ Has AWS credentials                                  │
│  ✓ Runs on user's machine                               │
│  ✓ No LLM access (parameters already extracted)         │
│  ✓ No S3 access (scripts already embedded)              │
└─────────────────────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────────────────────┐
│              USER'S AWS ESTATE                          │
│         (RDS, EC2, S3, Lambda, etc.)                    │
└─────────────────────────────────────────────────────────┘
```

### Key Design Principles

#### 1. S3 as Source of Truth
- All playbooks and scripts live in S3
- Qdrant is just a search cache/index
- On node restart, Qdrant recovers from its own S3 snapshots
- Manual sync updates Qdrant from playbook S3 bucket

#### 2. Embedded Scripts (No Client S3 Access)
- During sync, server downloads scripts from S3 and embeds them into playbook JSON
- Qdrant stores complete playbooks with script content inline
- Client receives everything in one API response
- **Why**: Avoids S3 permission issues on client side, simpler client implementation

#### 3. Versioning via Filename
- No folder nesting per playbook
- Version is part of filename: `stoprds-v1.yaml`, `stoprds-v2.yaml`
- Each version is a separate searchable entity in Qdrant
- Old versions remain searchable (no automatic deprecation)

#### 4. Manual Sync (Not Automatic)
- Sync is triggered by explicit API call: `POST /api/playbooks/sync`
- New playbooks won't appear in search until sync is run
- Gives control over when changes go live
- Incremental sync uses marker to track last synced state

#### 5. Incremental Sync with Marker
- Server tracks "last sync marker" (timestamp or playbook ID)
- Only syncs S3 files modified after last marker
- Avoids re-processing entire library on every sync
- Marker must persist across server restarts

#### 6. Stateless Search
- Each search request is independent
- No session state between requests
- Flow: Embedding generation → Vector search → LLM re-ranking → Return results
- Response contains complete playbook JSON ready for execution

### What You Need to Build

As a **server implementer**, you must build:

1. **Sync Mechanism**
   - API endpoint: `POST /api/playbooks/sync`
   - Read playbooks from S3 (incremental based on marker)
   - Download referenced scripts from S3
   - Embed scripts into playbook JSON (replace `script_ref` with actual code)
   - Generate embeddings from playbook content
   - Upsert to Qdrant with full payload
   - Update sync marker

2. **Search API**
   - Endpoint: `POST /api/playbooks/search`
   - Input: User intent + AWS resource context
   - Generate embedding from intent
   - Qdrant vector search with metadata filters
   - LLM re-ranking of top candidates
   - Return complete playbook JSON (with embedded scripts)

3. **Registration API** (Optional)
   - Endpoint: `POST /api/playbooks/register`
   - Upload new playbook YAML to S3
   - Upload associated scripts to S3
   - Trigger immediate sync (don't wait for manual sync)

4. **Marker Storage Strategy**
   - Choose where to store sync marker (Qdrant? S3? Database?)
   - Must persist across server restarts
   - Used for incremental sync

### What You Don't Need to Build

As a **server implementer**, you do NOT need:

- ❌ AWS execution engine (client does this)
- ❌ Parameter extraction from user context (Master Agent already did this)
- ❌ S3 credential management for client (scripts are embedded in response)
- ❌ Script validation/sandboxing (client responsibility)
- ❌ WebSocket streaming (client handles progress streaming)
- ❌ Playbook execution state tracking (client manages execution)
- ❌ AWS resource scanning (client does this locally)

### Response Format Example

When client calls search API, server returns:

```json
{
  "playbooks": [
    {
      "playbook_id": "stoprds-v2",
      "name": "Stop RDS Instance with Validation",
      "description": "Safely stops an RDS instance with pre-checks",
      "confidence": 0.95,
      "reasoning": "User requested RDS stop, this playbook includes validation",
      "risk_level": 2,
      "estimated_duration": 300,

      "parameters": [
        {
          "name": "db_instance_id",
          "type": "resource_id",
          "required": true,
          "description": "RDS instance identifier to stop"
        }
      ],

      "steps": [
        {
          "id": "step-1",
          "name": "Validate Instance Exists",
          "script": "#!/bin/bash\nset -e\nDB_ID=$1\naws rds describe-db-instances --db-instance-identifier \"${DB_ID}\"\necho \"Instance ${DB_ID} exists\"",
          "parameters": ["{{db_instance_id}}"],
          "timeout": 30,
          "on_failure": "stop"
        },
        {
          "id": "step-2",
          "name": "Stop RDS Instance",
          "script": "#!/bin/bash\nset -e\nDB_ID=$1\naws rds stop-db-instance --db-instance-identifier \"${DB_ID}\"\necho \"Stopping ${DB_ID}...\"",
          "parameters": ["{{db_instance_id}}"],
          "depends_on": ["step-1"],
          "timeout": 60,
          "on_failure": "stop"
        }
      ]
    }
  ]
}
```

**Note**: Scripts are EMBEDDED in the response. Client doesn't need to download anything else.

### Quick Start Checklist

Before implementing, ensure you have:

- [ ] **S3 Bucket Setup**
  - Create bucket: `s3://escher-library/`
  - Create folders: `playbooks/` and `scripts/`
  - Set up versioning on bucket
  - Configure appropriate IAM permissions

- [ ] **Qdrant Setup**
  - Install Qdrant with S3 snapshot persistence
  - Create collection: "playbooks" (384 dimensions)
  - Configure HNSW index parameters
  - Test recovery from S3 snapshots

- [ ] **Embedding Model**
  - Install `sentence-transformers` library
  - Load model: `all-MiniLM-L6-v2` (384 dimensions)
  - Test embedding generation
  - Consider caching embeddings

- [ ] **LLM Integration**
  - Set up Claude or GPT API access
  - Design re-ranking prompt template
  - Test with sample playbooks
  - Configure timeout and retry logic

- [ ] **Marker Storage Decision**
  - Choose storage location (Qdrant special entry? S3 file? Database?)
  - Implement marker read/write functions
  - Test persistence across restarts

- [ ] **API Endpoints**
  - Implement `POST /api/playbooks/sync`
  - Implement `POST /api/playbooks/search`
  - Implement `POST /api/playbooks/register` (optional)
  - Add authentication/authorization

- [ ] **Testing**
  - Create sample playbooks in S3
  - Test manual sync
  - Test search with various intents
  - Test incremental sync with marker
  - Test server restart (Qdrant recovery)

### Common Pitfalls to Avoid

1. **Don't auto-sync on startup**: Manual sync only, gives control over when changes appear
2. **Don't return S3 paths to client**: Embed scripts in response, avoid client S3 permissions
3. **Don't store scripts separately in Qdrant**: Embed in step definitions for atomic retrieval
4. **Don't forget the marker**: Without it, every sync re-processes entire S3 bucket
5. **Don't make Qdrant payload too small**: 50-100 KB per playbook is acceptable with embedded scripts
6. **Don't expose script registration without validation**: Always validate YAML structure and script references

---

## Playbook Structure

### Complete Playbook Format

A playbook consists of:
1. **metadata.json** - Core metadata and configuration
2. **orchestration.json** - Step definitions with script/playbook references (NEW)
3. **parameters.json** - Input parameters with extraction hints for Master Agent
4. **explain_plan.json** - Human-readable explanation with nested sub-steps

### 1. Metadata.json Structure

```json
{
  "playbook_id": "user-weekend-shutdown",
  "version": "1.2.0",
  "name": "Weekend DB Shutdown",
  "description": "Automated workflow for stopping production RDS instances over weekends to save costs while maintaining data integrity",

  "purpose": "Cost optimization during low-traffic weekends",

  "use_cases": [
    "Weekend database shutdown for cost savings",
    "Planned maintenance windows",
    "Development environment cost reduction"
  ],

  "keywords": [
    "production",
    "database",
    "rds",
    "cost-saving",
    "weekend",
    "shutdown",
    "snapshot"
  ],

  "cloud_provider": ["aws"],

  "resource_types": [
    "rds::instance",
    "rds::cluster"
  ],

  "author": "user_custom",

  "storage_strategy": "uploaded_for_review",

  "status": "active",

  "tags": {
    "category": "cost-optimization",
    "risk_level": "medium",
    "requires_approval": true
  },

  "estimated_impact": {
    "downtime_minutes": 10,
    "cost_savings_monthly_usd": 120,
    "affected_resources": "single-rds-instance"
  },

  "prerequisites": [
    "RDS instance must not have active connections",
    "Automated snapshot policy configured",
    "CloudWatch alarms configured for monitoring"
  ],

  "created_at": 1726329600,
  "updated_at": 1728391162,

  "based_on_escher": true,
  "escher_playbook_id": "aws-rds-stop",
  "escher_version": "1.0.0"
}
```

### 2. Parameters Structure

Parameters define what inputs the playbook needs. The `extraction_hint` field helps the **Master Agent** know where to look for values.

```json
{
  "parameters": [
    {
      "name": "instance_id",
      "type": "string",
      "description": "RDS instance identifier",
      "prompt": "Which RDS instance would you like to shut down?",
      "required": true,
      "default": null,
      "validation": {
        "pattern": "^[a-zA-Z][a-zA-Z0-9-]{0,62}$",
        "min_length": 1,
        "max_length": 63
      },
      "extraction_hint": {
        "source": "user_estate",
        "estate_query": {
          "resource_type": "rds::instance",
          "filters": {
            "tags.environment": "production",
            "state": "available"
          }
        },
        "display_field": "name",
        "value_field": "identifier"
      }
    },
    {
      "name": "snapshot_name",
      "type": "string",
      "description": "Name for the backup snapshot (optional)",
      "prompt": "What would you like to name the snapshot?",
      "required": false,
      "default": "auto-snapshot-{timestamp}",
      "extraction_hint": {
        "source": "generated",
        "template": "weekend-shutdown-{instance_id}-{timestamp}"
      }
    },
    {
      "name": "region",
      "type": "string",
      "description": "AWS region",
      "prompt": "Which AWS region is the instance in?",
      "required": true,
      "extraction_hint": {
        "source": "context",
        "context_field": "resource.region"
      }
    },
    {
      "name": "approval_email",
      "type": "string",
      "description": "Email to notify",
      "prompt": "Who should be notified?",
      "required": true,
      "extraction_hint": {
        "source": "prompt"
      }
    }
  ]
}
```

#### Extraction Hint Sources

These hints tell the **Master Agent** (server-side LLM) where to look for parameter values:

| Source | Description | How Master Agent Uses It |
|--------|-------------|---------------------------|
| `user_estate` | Query user's cloud resources | Master Agent looks in the estate_context JSON sent by client for matching resources |
| `context` | Extract from execution context | Master Agent examines the resource context to find this value |
| `generated` | Generate using template | Master Agent generates value using template (timestamps, names) |
| `static` | Use fixed value | Master Agent uses the default value |
| `prompt` | Ask user (cannot extract) | Master Agent cannot extract → leaves empty → User fills manually |

**Critical Understanding**: 

1. **Master Agent** (server LLM) reads these hints and tries to extract parameter values
2. **Playbook Agent** receives parameters **already extracted** by Master Agent  
3. **Client** receives playbook with:
   - Parameters Master Agent extracted: filled with values
   - Parameters Master Agent couldn't extract: empty placeholders ""
4. **User** manually fills any empty parameters
5. **Client** executes with all parameters filled

**There is NO client-side parameter extraction or "auto-fill". Only the Master Agent (server LLM) extracts parameters.**

10. **Step importance communicates criticality** - users understand which steps are essential vs nice-to-have

### 3. orchestration.json Structure (NEW)

**Purpose**: Defines the execution sequence using references to scripts and playbooks

#### Format for Pure Script Playbook

```json
{
  "orchestration_version": "1.0",
  "steps": [
    {
      "step_number": 1,
      "name": "Check RDS instance state",
      "type": "script",
      "script_ref": {
        "script_id": "check-rds-state",
        "version": "1.0.0",
        "implementation": "bash"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}",
        "region": "${playbook.region}"
      },
      "continue_on_error": false,
      "timeout_seconds": 30
    },
    {
      "step_number": 2,
      "name": "Create RDS snapshot",
      "type": "script",
      "script_ref": {
        "script_id": "create-rds-snapshot",
        "version": "2.1.0",
        "implementation": "python"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}",
        "snapshot_name": "${playbook.snapshot_name}",
        "region": "${playbook.region}"
      },
      "continue_on_error": false,
      "timeout_seconds": 300
    },
    {
      "step_number": 3,
      "name": "Stop RDS instance",
      "type": "script",
      "script_ref": {
        "script_id": "stop-rds-instance",
        "version": "1.0.0",
        "implementation": "bash"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}",
        "region": "${playbook.region}"
      },
      "continue_on_error": false,
      "timeout_seconds": 600
    }
  ]
}
```

#### Format for Pure Orchestration Playbook

```json
{
  "orchestration_version": "1.0",
  "steps": [
    {
      "step_number": 1,
      "name": "Create RDS snapshot",
      "type": "playbook",
      "playbook_ref": {
        "playbook_id": "create-rds-snapshot",
        "version": "1.0.0"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}",
        "snapshot_name": "${playbook.snapshot_name}"
      },
      "continue_on_error": false
    },
    {
      "step_number": 2,
      "name": "Stop RDS instance",
      "type": "playbook",
      "playbook_ref": {
        "playbook_id": "stop-rds-instance",
        "version": "2.0.0"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}"
      },
      "continue_on_error": false
    }
  ]
}
```

#### Format for Hybrid Playbook

```json
{
  "orchestration_version": "1.0",
  "steps": [
    {
      "step_number": 1,
      "name": "Create RDS snapshot",
      "type": "playbook",
      "playbook_ref": {
        "playbook_id": "create-rds-snapshot",
        "version": "1.0.0"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}"
      },
      "continue_on_error": false
    },
    {
      "step_number": 2,
      "name": "Verify backup integrity",
      "type": "script",
      "script_ref": {
        "script_id": "verify-backup",
        "version": "1.0.0",
        "implementation": "python"
      },
      "parameter_mapping": {
        "snapshot_id": "${step1.output.snapshot_id}",
        "instance_id": "${playbook.instance_id}"
      },
      "continue_on_error": false,
      "timeout_seconds": 120
    },
    {
      "step_number": 3,
      "name": "Stop RDS instance",
      "type": "playbook",
      "playbook_ref": {
        "playbook_id": "stop-rds-instance",
        "version": "2.0.0"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}"
      },
      "continue_on_error": false
    }
  ]
}
```

#### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `orchestration_version` | string | Format version (currently "1.0") |
| `steps` | array | Ordered list of execution steps |
| `step_number` | integer | Sequential step number (1, 2, 3...) |
| `name` | string | Human-readable step name |
| `type` | enum | "script" or "playbook" |
| `script_ref` | object | Reference to script (if type=script) |
| `playbook_ref` | object | Reference to playbook (if type=playbook) |
| `parameter_mapping` | object | Maps step parameters to playbook parameters or previous step outputs |
| `continue_on_error` | boolean | If true, continue to next step even if this fails |
| `timeout_seconds` | integer | Step timeout (optional, default from script metadata) |

#### Advanced Step Configuration: Failure Handling & Validation

**NEW**: Each step can now include sophisticated failure handling strategies and validation scripts to ensure robust execution.

##### Failure Handling Strategy

Each step defines how to handle failures:

| Field | Type | Values | Description |
|-------|------|--------|-------------|
| `failure_action` | enum | `stop`, `ignore`, `retry` | Action to take when step fails |
| `importance` | enum | `critical`, `high`, `low` | User-visible importance level |
| `retry_config` | object | {...} | Retry configuration (only if `failure_action: retry`) |

**Failure Actions**:

- **`stop`**: Abort entire playbook execution immediately (default for critical steps)
  - Use for: Prerequisites, critical operations that can't be skipped
  - Example: Checking instance exists before operating on it

- **`ignore`**: Log error and continue to next step (for non-critical operations)
  - Use for: Notifications, logging, optional cleanup
  - Example: Sending Slack notification (nice to have, not required)

- **`retry`**: Attempt step again with exponential backoff
  - Use for: Transient failures (network timeouts, API rate limits, eventual consistency)
  - Example: Creating snapshot (may fail temporarily due to AWS internal issues)
  - Requires: `retry_config` object

**Retry Configuration**:

```json
{
  "retry_config": {
    "max_attempts": 3,
    "backoff_seconds": [30, 60, 120],
    "retry_on_exit_codes": [1, 2, 124],  // Optional: Only retry on specific exit codes
    "retry_on_errors": [".*timeout.*", ".*throttl.*"]  // Optional: Regex patterns in stderr
  }
}
```

##### Pre-Validation & Post-Validation (Optional)

Each step **can optionally** have validation scripts that run **before** and/or **after** the main script. Both are completely optional - you can have:
- ✅ No validation (just run the main script)
- ✅ Only pre-validation
- ✅ Only post-validation
- ✅ Both pre-validation and post-validation

**Pre-Validation** (optional - runs before main script):
- Verifies prerequisites are met
- Checks parameters exist and are valid
- Validates current state allows the operation
- Confirms permissions are available
- Checks dependencies from previous steps

**Post-Validation** (optional - runs after main script):
- Verifies operation actually succeeded
- Checks expected outputs exist
- Validates output formats and values
- Confirms state change occurred
- Supports polling for eventual consistency

**Validation Structure**:

```json
{
  "pre_validation": {
    "script_ref": {
      "script_id": "pre-validate-snapshot-creation",
      "version": "1.0.0"
    },
    "parameter_mapping": {
      "instance_id": "${playbook.instance_id}",
      "snapshot_name": "${playbook.snapshot_name}"
    },
    "checks": [
      {
        "type": "parameter_exists",
        "parameters": ["instance_id", "snapshot_name"]
      },
      {
        "type": "instance_state",
        "expected_states": ["available", "stopped"]
      },
      {
        "type": "permission_check",
        "required_permissions": ["rds:CreateDBSnapshot"]
      }
    ],
    "timeout_seconds": 30,
    "failure_action": "stop"
  },
  
  "post_validation": {
    "script_ref": {
      "script_id": "post-validate-snapshot-created",
      "version": "1.0.0"
    },
    "parameter_mapping": {
      "snapshot_id": "${step2.output.snapshot_id}"
    },
    "checks": [
      {
        "type": "output_variables_exist",
        "required_variables": ["snapshot_id", "snapshot_arn", "status"]
      },
      {
        "type": "snapshot_state",
        "expected_states": ["creating", "available"],
        "polling": {
          "enabled": true,
          "interval_seconds": 30,
          "max_attempts": 10
        }
      }
    ],
    "timeout_seconds": 300,
    "failure_action": "stop"
  }
}
```

**Validation Check Types**:

| Check Type | Purpose | Example |
|------------|---------|---------|
| `parameter_exists` | Verify required parameters are present | Check instance_id exists |
| `previous_step_output_exists` | Verify outputs from previous steps | Check step1.output.snapshot_id exists |
| `instance_state` | Check current AWS resource state | Instance must be "available" |
| `permission_check` | Verify IAM permissions | User has rds:CreateDBSnapshot |
| `output_variables_exist` | Verify script produced expected outputs | snapshot_id, snapshot_arn exist |
| `output_format` | Validate output format with regex | snapshot_id matches `^snap-[a-f0-9]{17}$` |
| `state_value_check` | Check output value matches expected | final_state == "stopped" |

##### Complete Step Example with All Features

```json
{
  "step_number": 2,
  "name": "Create RDS snapshot",
  "type": "script",
  
  "failure_action": "retry",
  "retry_config": {
    "max_attempts": 3,
    "backoff_seconds": [30, 60, 120]
  },
  "importance": "critical",
  
  "pre_validation": {
    "script_ref": {
      "script_id": "pre-validate-snapshot-creation",
      "version": "1.0.0"
    },
    "parameter_mapping": {
      "instance_id": "${playbook.instance_id}",
      "snapshot_name": "${playbook.snapshot_name}",
      "instance_state": "${step1.output.instance_state}"
    },
    "checks": [
      {
        "type": "parameter_exists",
        "parameters": ["instance_id", "snapshot_name"],
        "description": "Verify required parameters are present"
      },
      {
        "type": "previous_step_output_exists",
        "step": 1,
        "required_outputs": ["instance_state", "instance_arn"],
        "description": "Verify step 1 completed successfully"
      },
      {
        "type": "instance_state",
        "current_state": "${step1.output.instance_state}",
        "allowed_states": ["available", "stopped"],
        "description": "Can only snapshot from available or stopped state"
      },
      {
        "type": "snapshot_name_unique",
        "snapshot_name": "${playbook.snapshot_name}",
        "description": "Snapshot name must not already exist"
      },
      {
        "type": "permission_check",
        "required_permissions": ["rds:CreateDBSnapshot", "rds:DescribeDBInstances"],
        "description": "Verify user has required permissions"
      }
    ],
    "timeout_seconds": 30,
    "failure_action": "stop"
  },
  
  "script_ref": {
    "script_id": "create-rds-snapshot",
    "version": "2.1.0",
    "implementation": "python"
  },
  
  "parameter_mapping": {
    "instance_id": "${playbook.instance_id}",
    "snapshot_name": "${playbook.snapshot_name}",
    "region": "${playbook.region}"
  },
  
  "post_validation": {
    "script_ref": {
      "script_id": "post-validate-snapshot-created",
      "version": "1.0.0"
    },
    "parameter_mapping": {
      "snapshot_id": "${step2.output.snapshot_id}",
      "snapshot_name": "${playbook.snapshot_name}",
      "region": "${playbook.region}"
    },
    "checks": [
      {
        "type": "output_variables_exist",
        "required_variables": ["snapshot_id", "snapshot_arn", "status"],
        "description": "Verify script produced all required outputs"
      },
      {
        "type": "snapshot_state",
        "snapshot_id": "${step2.output.snapshot_id}",
        "expected_states": ["creating", "available"],
        "description": "Verify snapshot is being created or is available",
        "polling": {
          "enabled": true,
          "interval_seconds": 30,
          "max_attempts": 10,
          "description": "Poll AWS API until snapshot reaches expected state"
        }
      },
      {
        "type": "output_format",
        "validations": {
          "snapshot_id": "^snap-[a-f0-9]{17}$",
          "snapshot_arn": "^arn:aws:rds:.*"
        },
        "description": "Validate output formats match AWS conventions"
      }
    ],
    "timeout_seconds": 300,
    "failure_action": "stop"
  },
  
  "timeout_seconds": 300
}
```

##### Execution Flow with Validations

```
Step 2: "Create RDS snapshot"
  ↓
┌─────────────────────────────────────────────────────────────────┐
│ PRE-VALIDATION                                                  │
├─────────────────────────────────────────────────────────────────┤
│ Execute: pre-validate-snapshot-creation.sh                      │
│                                                                  │
│ Check 1: parameter_exists                                       │
│   ✓ instance_id = "prod-db-01"                                  │
│   ✓ snapshot_name = "weekend-shutdown-prod-db-01-1728391162"   │
│                                                                  │
│ Check 2: previous_step_output_exists                            │
│   ✓ step1.output.instance_state exists                          │
│   ✓ step1.output.instance_arn exists                            │
│                                                                  │
│ Check 3: instance_state                                         │
│   ✓ Current state: "available"                                  │
│   ✓ Allowed states: ["available", "stopped"]                    │
│                                                                  │
│ Check 4: snapshot_name_unique                                   │
│   ✓ No existing snapshot with this name                         │
│                                                                  │
│ Check 5: permission_check                                       │
│   ✓ User has rds:CreateDBSnapshot                               │
│   ✓ User has rds:DescribeDBInstances                            │
│                                                                  │
│ Result: PRE-VALIDATION PASSED ✓                                 │
└─────────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────────┐
│ MAIN SCRIPT EXECUTION                                           │
├─────────────────────────────────────────────────────────────────┤
│ Execute: create-rds-snapshot.py                                 │
│                                                                  │
│ Result: Exit code 0                                             │
│ Output: {                                                        │
│   "snapshot_id": "snap-0a1b2c3d4e5f67890",                      │
│   "snapshot_arn": "arn:aws:rds:us-east-1:123:snapshot:...",     │
│   "status": "creating"                                           │
│ }                                                                │
└─────────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────────┐
│ POST-VALIDATION                                                 │
├─────────────────────────────────────────────────────────────────┤
│ Execute: post-validate-snapshot-created.sh                      │
│                                                                  │
│ Check 1: output_variables_exist                                 │
│   ✓ snapshot_id exists                                          │
│   ✓ snapshot_arn exists                                         │
│   ✓ status exists                                               │
│                                                                  │
│ Check 2: snapshot_state (with polling)                          │
│   Poll 1: state = "creating" ✓ (wait 30s)                       │
│   Poll 2: state = "creating" ✓ (wait 30s)                       │
│   Poll 3: state = "available" ✓ (done)                          │
│                                                                  │
│ Check 3: output_format                                          │
│   ✓ snapshot_id matches regex: ^snap-[a-f0-9]{17}$              │
│   ✓ snapshot_arn matches regex: ^arn:aws:rds:.*                 │
│                                                                  │
│ Result: POST-VALIDATION PASSED ✓                                │
└─────────────────────────────────────────────────────────────────┘
  ↓
Step 2 Complete ✓
Continue to Step 3
```

##### Failure Scenario Examples

**Example 1: Pre-Validation Failure (Stop Execution)**

```
Step 4: "Stop RDS instance"
  ↓
PRE-VALIDATION
  Check: instance_state
    Current: "stopped"
    Expected: ["available", "starting", "running"]
    Result: ❌ FAILED
  ↓
failure_action = "stop"
  ↓
PLAYBOOK ABORTED
User message: "Cannot stop instance prod-db-01: Already in 'stopped' state"
```

**Example 2: Main Script Failure (Retry)**

```
Step 2: "Create RDS snapshot"
  ↓
PRE-VALIDATION PASSED ✓
  ↓
MAIN SCRIPT EXECUTION
  Result: Exit code 1 (Network timeout)
  ↓
failure_action = "retry"
retry_config.max_attempts = 3
  ↓
Attempt 1 FAILED
Wait backoff_seconds[0] = 30s
  ↓
RETRY ATTEMPT 2
PRE-VALIDATION PASSED ✓
MAIN SCRIPT EXECUTION
  Result: Exit code 0 ✓
  ↓
POST-VALIDATION PASSED ✓
  ↓
Step 2 Complete ✓ (succeeded on retry)
```

**Example 3: Non-Critical Step Failure (Ignore)**

```
Step 3: "Send Slack notification"
  ↓
PRE-VALIDATION PASSED ✓
  ↓
MAIN SCRIPT EXECUTION
  Result: Exit code 1 (Slack API unavailable)
  ↓
failure_action = "ignore"
importance = "low"
  ↓
Log warning: "Step 3 failed but marked as non-critical, continuing..."
  ↓
Continue to Step 4
```

**Example 4: Post-Validation Timeout (Stop)**

```
Step 4: "Stop RDS instance"
  ↓
PRE-VALIDATION PASSED ✓
  ↓
MAIN SCRIPT EXECUTION SUCCESS ✓
  Output: {final_state: "stopping"}
  ↓
POST-VALIDATION
  Check: instance_state (polling enabled)
    Poll 1-20: state = "stopping" (600 seconds elapsed)
    Timeout reached ❌
  ↓
failure_action = "stop"
  ↓
PLAYBOOK FAILED
User message: "Post-validation failed: Instance did not reach 'stopped' state within 600 seconds"
```

##### Benefits of Validation & Failure Handling

1. **User Understands Importance**
   - `importance: critical` → User knows this step cannot be skipped
   - `importance: low` → User knows failure is acceptable
   - Clear communication about what's essential vs nice-to-have

2. **Robust Execution**
   - Pre-validation catches issues before attempting operations
   - Post-validation confirms operations actually succeeded (not just API accepted)
   - Retry logic handles transient failures automatically

3. **Better Error Messages**
   - Pre-validation failures show exactly what prerequisite wasn't met
   - Post-validation failures show exactly what expected outcome didn't happen
   - Users get actionable information, not just "step failed"

4. **Smart Recovery**
   - Retry for transient failures (network, rate limits)
   - Ignore for non-critical operations (notifications, logging)
   - Stop for critical failures (missing resources, wrong state)

5. **Guaranteed State**
   - Polling in post-validation ensures state changes actually occurred
   - Not just "command sent" but "state confirmed changed"
   - Handles eventual consistency in cloud APIs

6. **Type Safety**
   - Output format validation ensures data integrity
   - Downstream steps receive correctly formatted data
   - Prevents cascading failures from malformed outputs


#### script_ref Structure

```json
{
  "script_id": "create-rds-snapshot",
  "version": "2.1.0",
  "implementation": "python"
}
```

**Fields**:
- `script_id`: Unique identifier in scripts library
- `version`: Script version (semantic versioning)
- `implementation`: User choice of bash/python/node/terraform/cloudformation

**Resolution at Runtime**:
- Client looks up: `scripts/create-rds-snapshot/v2.1.0/code/python/`
- Execution Engine runs the Python implementation
- User can override implementation in UI

#### playbook_ref Structure

```json
{
  "playbook_id": "stop-rds-instance",
  "version": "2.0.0"
}
```

**Fields**:
- `playbook_id`: Unique identifier in playbooks library
- `version`: Playbook version

**Resolution at Runtime**:
- Recursively resolve referenced playbook
- Expand sub-steps from nested playbook
- Track hierarchy for explain plan display

#### Parameter Mapping

Parameters use variable substitution with these sources:

| Source | Format | Example |
|--------|--------|---------|
| Playbook parameter | `${playbook.param_name}` | `${playbook.instance_id}` |
| Previous step output | `${stepN.output.field}` | `${step1.output.snapshot_id}` |
| Cloud estate context | `${estate.resource.field}` | `${estate.resource.region}` |
| Static value | Direct value | `"us-east-1"` or `true` |

**Example**:
```json
"parameter_mapping": {
  "instance_id": "${playbook.instance_id}",
  "snapshot_id": "${step1.output.snapshot_id}",
  "region": "${estate.resource.region}",
  "dry_run": false
}
```

### 4. Scripts Library Structure (NEW - Normalized Storage)

**Key Concept**: Scripts are stored **separately** from playbooks in a reusable library. Playbooks reference scripts via `script_id` + `version`.

#### Script metadata.json

Each script has metadata defining available implementations:

```json
{
  "script_id": "create-rds-snapshot",
  "version": "2.1.0",
  "name": "Create RDS Snapshot",
  "description": "Creates a manual snapshot of an RDS instance with verification",

  "available_types": ["bash", "python", "node"],
  "default_type": "bash",

  "parameters": [
    {
      "name": "instance_id",
      "type": "string",
      "required": true,
      "description": "RDS instance identifier"
    },
    {
      "name": "snapshot_name",
      "type": "string",
      "required": true,
      "description": "Name for the snapshot"
    },
    {
      "name": "region",
      "type": "string",
      "required": true,
      "description": "AWS region"
    }
  ],

  "outputs": [
    {
      "name": "snapshot_id",
      "type": "string",
      "description": "ID of the created snapshot"
    },
    {
      "name": "snapshot_arn",
      "type": "string",
      "description": "ARN of the snapshot"
    }
  ],

  "cloud_provider": "aws",
  "resource_types": ["rds::instance"],

  "permissions_required": [
    "rds:CreateDBSnapshot",
    "rds:DescribeDBSnapshots"
  ],

  "estimated_duration_seconds": 300,
  "timeout_seconds": 600,

  "created_at": 1726329600,
  "updated_at": 1728391162
}
```

#### Multi-Implementation Code Structure

Each script can have multiple implementations (bash, python, node, terraform, cloudformation):

```
scripts/create-rds-snapshot/v2.1.0/
├── metadata.json
├── parameters.json
└── code/
    ├── bash/
    │   ├── script.sh
    │   └── README.md
    ├── python/
    │   ├── main.py
    │   ├── requirements.txt
    │   └── README.md
    ├── node/
    │   ├── index.js
    │   ├── package.json
    │   └── README.md
    ├── terraform/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── README.md
    └── cloudformation/
        ├── template.yaml
        └── README.md
```

#### Implementation Details by Type

**Bash Implementation**:
```bash
#!/bin/bash
set -e

# create-rds-snapshot v2.1.0
# Creates a manual snapshot of an RDS instance

INSTANCE_ID="$1"
SNAPSHOT_NAME="$2"
REGION="$3"

# Create snapshot
aws rds create-db-snapshot \
  --db-instance-identifier "$INSTANCE_ID" \
  --db-snapshot-identifier "$SNAPSHOT_NAME" \
  --region "$REGION"

# Wait for snapshot to complete
aws rds wait db-snapshot-available \
  --db-snapshot-identifier "$SNAPSHOT_NAME" \
  --region "$REGION"

# Get snapshot details
SNAPSHOT_ARN=$(aws rds describe-db-snapshots \
  --db-snapshot-identifier "$SNAPSHOT_NAME" \
  --region "$REGION" \
  --query 'DBSnapshots[0].DBSnapshotArn' \
  --output text)

# Output (JSON format for step outputs)
echo "{\"snapshot_id\": \"$SNAPSHOT_NAME\", \"snapshot_arn\": \"$SNAPSHOT_ARN\"}"
```

**Python Implementation**:
```python
#!/usr/bin/env python3
import boto3
import json
import sys
import time

def create_rds_snapshot(instance_id: str, snapshot_name: str, region: str):
    """Create RDS snapshot and wait for completion"""

    rds = boto3.client('rds', region_name=region)

    # Create snapshot
    response = rds.create_db_snapshot(
        DBInstanceIdentifier=instance_id,
        DBSnapshotIdentifier=snapshot_name
    )

    # Wait for snapshot to be available
    waiter = rds.get_waiter('db_snapshot_available')
    waiter.wait(DBSnapshotIdentifier=snapshot_name)

    # Get snapshot details
    snapshot = rds.describe_db_snapshots(
        DBSnapshotIdentifier=snapshot_name
    )['DBSnapshots'][0]

    # Return outputs as JSON
    outputs = {
        "snapshot_id": snapshot['DBSnapshotIdentifier'],
        "snapshot_arn": snapshot['DBSnapshotArn']
    }

    print(json.dumps(outputs))
    return outputs

if __name__ == "__main__":
    instance_id = sys.argv[1]
    snapshot_name = sys.argv[2]
    region = sys.argv[3]

    create_rds_snapshot(instance_id, snapshot_name, region)
```

**Node.js Implementation**:
```javascript
const AWS = require('aws-sdk');

async function createRdsSnapshot(instanceId, snapshotName, region) {
  const rds = new AWS.RDS({ region });

  // Create snapshot
  await rds.createDBSnapshot({
    DBInstanceIdentifier: instanceId,
    DBSnapshotIdentifier: snapshotName
  }).promise();

  // Wait for snapshot to be available
  await rds.waitFor('dbSnapshotAvailable', {
    DBSnapshotIdentifier: snapshotName
  }).promise();

  // Get snapshot details
  const { DBSnapshots } = await rds.describeDBSnapshots({
    DBSnapshotIdentifier: snapshotName
  }).promise();

  const snapshot = DBSnapshots[0];

  // Return outputs as JSON
  const outputs = {
    snapshot_id: snapshot.DBSnapshotIdentifier,
    snapshot_arn: snapshot.DBSnapshotArn
  };

  console.log(JSON.stringify(outputs));
  return outputs;
}

// Main
const [instanceId, snapshotName, region] = process.argv.slice(2);
createRdsSnapshot(instanceId, snapshotName, region)
  .catch(err => {
    console.error(err);
    process.exit(1);
  });
```

**Terraform Implementation**:
```hcl
# main.tf
variable "instance_id" {
  type = string
}

variable "snapshot_name" {
  type = string
}

variable "region" {
  type = string
}

provider "aws" {
  region = var.region
}

resource "aws_db_snapshot" "manual" {
  db_instance_identifier = var.instance_id
  db_snapshot_identifier = var.snapshot_name
}

output "snapshot_id" {
  value = aws_db_snapshot.manual.id
}

output "snapshot_arn" {
  value = aws_db_snapshot.manual.db_snapshot_arn
}
```

**CloudFormation Implementation**:
```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Create RDS Snapshot

Parameters:
  InstanceId:
    Type: String
  SnapshotName:
    Type: String
  Region:
    Type: String

Resources:
  RDSSnapshot:
    Type: AWS::RDS::DBSnapshot
    Properties:
      DBInstanceIdentifier: !Ref InstanceId
      DBSnapshotIdentifier: !Ref SnapshotName

Outputs:
  SnapshotId:
    Value: !Ref RDSSnapshot
  SnapshotArn:
    Value: !GetAtt RDSSnapshot.DBSnapshotArn
```

#### Benefits of Multi-Implementation

| Benefit | Description |
|---------|-------------|
| **User Choice** | DevOps teams can use their preferred language/tool |
| **Flexibility** | Same logic available in multiple formats |
| **Gradual Migration** | Add new implementations without breaking existing playbooks |
| **Optimization** | Use bash for simple tasks, Python for complex logic |
| **Tool-Specific Features** | Terraform for IaC, CloudFormation for AWS-native |

#### Script Storage on Disk

**Local (Client)**:
```
~/.escher/scripts/
├── create-rds-snapshot/
│   ├── v1.0.0/
│   └── v2.1.0/
│       ├── metadata.json
│       └── code/
│           ├── bash/script.sh
│           ├── python/main.py
│           └── node/index.js
├── stop-rds-instance/
└── verify-backup/
```

**S3 (Server)**:
```
s3://escher-library/scripts/
├── create-rds-snapshot/
│   └── v2.1.0/
│       ├── metadata.json
│       └── code/
│           ├── bash/script.sh
│           ├── python/main.py
│           ├── node/index.js
│           ├── terraform/main.tf
│           └── cloudformation/template.yaml
```

### 5. Explain Plan Structure

```json
{
  "explain_plan": {
    "what_it_does": "Creates a snapshot of the RDS instance and stops it gracefully",

    "why_use_it": [
      "Save costs during low-traffic periods (weekends, holidays)",
      "Prevent data loss with automatic snapshot creation",
      "Reduce infrastructure costs without deleting resources"
    ],

    "what_happens": [
      {
        "step": 1,
        "action": "Verify RDS instance state",
        "details": "Check that instance is in 'available' state and has no active connections",
        "duration_seconds": 5,
        "can_fail": true,
        "rollback": null
      },
      {
        "step": 2,
        "action": "Create snapshot",
        "details": "Create a manual snapshot with name pattern: weekend-shutdown-{instance}-{timestamp}",
        "duration_seconds": 30,
        "can_fail": true,
        "rollback": "Delete incomplete snapshot"
      },
      {
        "step": 3,
        "action": "Stop RDS instance",
        "details": "Issue StopDBInstance command and wait for instance to reach 'stopped' state",
        "duration_seconds": 120,
        "can_fail": true,
        "rollback": "Restart instance if snapshot was created"
      },
      {
        "step": 4,
        "action": "Verify shutdown",
        "details": "Confirm instance is in 'stopped' state and no charges are accruing",
        "duration_seconds": 10,
        "can_fail": false,
        "rollback": null
      }
    ],

    "estimated_impact": {
      "downtime_minutes": 10,
      "cost_savings_monthly_usd": 120,
      "cost_savings_calculation": "db.t3.medium running 8 hours/day (weekdays only) vs 24/7",
      "affected_resources": "single-rds-instance",
      "reversible": true,
      "data_loss_risk": "none"
    },

    "risks": [
      {
        "risk": "Active connections disrupted",
        "severity": "medium",
        "mitigation": "Pre-flight check verifies no active connections"
      },
      {
        "risk": "Snapshot creation failure",
        "severity": "low",
        "mitigation": "Playbook aborts if snapshot fails, instance remains running"
      }
    ],

    "rollback_strategy": "If any step fails after snapshot creation, instance can be restarted. Snapshot remains for recovery.",

    "success_criteria": [
      "Instance state is 'stopped'",
      "Snapshot created successfully",
      "No errors in CloudWatch logs"
    ]
  }
}
```

#### Nested Sub-Steps with Hierarchical Structure (NEW)

**Key Concept**: When a playbook calls another playbook (Pure Orchestration or Hybrid), the explain plan displays nested sub-steps to show the complete hierarchy.

**Example: Parent Playbook Calling Child Playbooks**

Parent playbook `weekend-rds-automation` orchestrates two child playbooks:

```json
{
  "explain_plan": {
    "what_it_does": "Automated weekend RDS cost optimization with backup and shutdown",
    "what_happens": [
      {
        "step": 1,
        "action": "Create RDS snapshot",
        "type": "playbook",
        "playbook_ref": {
          "playbook_id": "create-rds-snapshot",
          "version": "1.0.0"
        },
        "duration_seconds": 35,
        "sub_steps": [
          {
            "step": "1.1",
            "action": "Verify RDS instance exists",
            "type": "script",
            "script_id": "check-rds-state",
            "duration_seconds": 5,
            "blocking": true
          },
          {
            "step": "1.2",
            "action": "Create manual snapshot",
            "type": "script",
            "script_id": "create-snapshot",
            "duration_seconds": 25,
            "blocking": true
          },
          {
            "step": "1.3",
            "action": "Verify snapshot completion",
            "type": "script",
            "script_id": "verify-snapshot",
            "duration_seconds": 5,
            "blocking": true
          }
        ],
        "can_fail": true,
        "rollback": "Delete incomplete snapshot"
      },
      {
        "step": 2,
        "action": "Stop RDS instance",
        "type": "playbook",
        "playbook_ref": {
          "playbook_id": "stop-rds-instance",
          "version": "2.0.0"
        },
        "duration_seconds": 130,
        "sub_steps": [
          {
            "step": "2.1",
            "action": "Check for active connections",
            "type": "script",
            "script_id": "check-connections",
            "duration_seconds": 10,
            "blocking": true
          },
          {
            "step": "2.2",
            "action": "Issue stop command",
            "type": "script",
            "script_id": "stop-instance",
            "duration_seconds": 5,
            "blocking": true
          },
          {
            "step": "2.3",
            "action": "Wait for stopped state",
            "type": "script",
            "script_id": "wait-stopped",
            "duration_seconds": 115,
            "blocking": true
          }
        ],
        "can_fail": true,
        "rollback": "Restart instance if needed"
      },
      {
        "step": 3,
        "action": "Send notification",
        "type": "script",
        "script_id": "send-sns-notification",
        "duration_seconds": 2,
        "can_fail": false,
        "rollback": null
      }
    ]
  }
}
```

**UI Display Example**:

```
Weekend RDS Automation
├─ Step 1: Create RDS snapshot (35s)
│  ├─ 1.1: Verify RDS instance exists (5s) ✓ BLOCKING
│  ├─ 1.2: Create manual snapshot (25s) ✓ BLOCKING
│  └─ 1.3: Verify snapshot completion (5s) ✓ BLOCKING
│
├─ Step 2: Stop RDS instance (130s)
│  ├─ 2.1: Check for active connections (10s) ✓ BLOCKING
│  ├─ 2.2: Issue stop command (5s) ✓ BLOCKING
│  └─ 2.3: Wait for stopped state (115s) ✓ BLOCKING
│
└─ Step 3: Send notification (2s)

Total Duration: ~167 seconds
```

**Blocking Behavior**:

All sub-steps are **blocking** by default, meaning:
- Sub-step 1.1 must complete before 1.2 starts
- Sub-step 1.2 must complete before 1.3 starts
- All sub-steps (1.1, 1.2, 1.3) must complete before Step 2 starts

**Failure Handling with Sub-Steps**:

```json
{
  "step": 1,
  "action": "Create RDS snapshot",
  "sub_steps": [
    {
      "step": "1.1",
      "action": "Verify RDS instance exists",
      "can_fail": true,
      "on_failure": "abort_parent"
    },
    {
      "step": "1.2",
      "action": "Create manual snapshot",
      "can_fail": true,
      "on_failure": "rollback_and_abort"
    },
    {
      "step": "1.3",
      "action": "Verify snapshot completion",
      "can_fail": false
    }
  ]
}
```

**on_failure Options**:
- `abort_parent`: Stop entire parent step (Step 1) and propagate failure
- `rollback_and_abort`: Run rollback logic, then abort parent
- `continue`: Log error but continue to next sub-step (rare)

**Deep Nesting (3 Levels)**:

Playbooks can nest up to 3 levels deep:

```
Parent Playbook (Level 1)
├─ Step 1: High-level workflow (Level 2 - calls another playbook)
│  ├─ 1.1: Mid-level task (Level 3 - calls scripts)
│  │  ├─ 1.1.1: Script execution
│  │  └─ 1.1.2: Script execution
│  └─ 1.2: Mid-level task
│     └─ 1.2.1: Script execution
└─ Step 2: Direct script call
```

**Example Use Case**: Disaster Recovery Playbook

```
DR: Restore Production RDS
├─ Step 1: Validate backup availability (playbook)
│  ├─ 1.1: Find latest snapshot (script)
│  ├─ 1.2: Verify snapshot integrity (script)
│  └─ 1.3: Check restore prerequisites (script)
│
├─ Step 2: Restore database (playbook)
│  ├─ 2.1: Create new RDS instance (script)
│  ├─ 2.2: Wait for instance ready (script)
│  ├─ 2.3: Restore from snapshot (script)
│  └─ 2.4: Verify restore completion (script)
│
├─ Step 3: Update DNS records (playbook)
│  ├─ 3.1: Update Route53 entry (script)
│  └─ 3.2: Verify DNS propagation (script)
│
└─ Step 4: Run smoke tests (script)
```

**Generation Process**:

1. **At Publish Time** (Server):
   - Playbook Agent recursively resolves all playbook_refs
   - Expands sub-steps from nested playbooks
   - Flattens to 3-level hierarchy
   - Stores in explain_plan.json

2. **At Search Time** (Server):
   - Returns explain_plan with all sub-steps pre-populated
   - Client displays hierarchical tree view

3. **At Execution Time** (Client):
   - Execution Engine follows step order: 1 → 1.1 → 1.2 → 1.3 → 2 → 2.1...
   - Updates UI with real-time progress: ✓ 1.1 Done → ⏳ 1.2 Running → ...

**Benefits**:
- **Transparency**: User sees complete workflow before execution
- **Predictability**: Know exact duration and order
- **Debugging**: Pinpoint failures to specific sub-steps
- **Trust**: Understand what each nested playbook does

### 6. Complete Playbook Structure on Disk

```
~/.escher/playbooks/synced/user-weekend-shutdown/
├── v1.0.0/
│   ├── metadata.json              # Core metadata
│   ├── parameters.json            # Parameter definitions
│   ├── explain_plan.json          # Explain plan
│   ├── shell/
│   │   └── main.sh               # Shell script
│   ├── cloudformation/
│   │   └── template.yaml         # CloudFormation template
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── python/
│       ├── stop_rds.py           # Python script
│       └── requirements.txt      # Dependencies
└── v1.2.0/
    ├── metadata.json
    ├── parameters.json
    ├── explain_plan.json
    └��─ ... (same structure)
```

---

## Three Playbook Types

### Type 1: Pure Script Playbook

**Definition**: Playbook that only executes scripts (no sub-playbook calls)

**Example**: "stop-rds-instance"

```
playbooks/stop-rds-instance/v1.0.0/
├── metadata.json
├── parameters.json
└── orchestration.json
    {
      "steps": [
        {
          "step_number": 1,
          "type": "script",
          "script_ref": {
            "script_id": "check-rds-state",
            "version": "1.0.0"
          }
        },
        {
          "step_number": 2,
          "type": "script",
          "script_ref": {
            "script_id": "stop-rds-instance",
            "version": "1.0.0"
          }
        }
      ]
    }
```

**Characteristics**:
- References only script_ids
- Self-contained execution
- No nested playbooks

---

### Type 2: Pure Orchestration Playbook

**Definition**: Playbook that only calls other playbooks (no direct scripts)

**Example**: "stop-rds-with-snapshot"

```
playbooks/stop-rds-with-snapshot/v1.0.0/
├── metadata.json
├── parameters.json
└── orchestration.json
    {
      "steps": [
        {
          "step_number": 1,
          "type": "playbook",
          "playbook_ref": {
            "playbook_id": "create-rds-snapshot",
            "version": "1.0.0"
          }
        },
        {
          "step_number": 2,
          "type": "playbook",
          "playbook_ref": {
            "playbook_id": "stop-rds-instance",
            "version": "2.0.0"
          }
        }
      ]
    }
```

**Characteristics**:
- References only playbook_ids
- Pure coordination logic
- Builds complex workflows from simple building blocks

**Benefits**:
- **Reusability**: "create-rds-snapshot" used by 10+ parent playbooks
- **Maintainability**: Update child → all parents benefit
- **Composition**: Complex operations from simple primitives

---

### Type 3: Hybrid Playbook

**Definition**: Playbook that mixes script calls + playbook calls

**Example**: "backup-and-stop-rds"

```
playbooks/backup-and-stop-rds/v1.0.0/
├── metadata.json
├── parameters.json
└── orchestration.json
    {
      "steps": [
        {
          "step_number": 1,
          "type": "playbook",
          "playbook_ref": {
            "playbook_id": "create-rds-snapshot",
            "version": "1.0.0"
          }
        },
        {
          "step_number": 2,
          "type": "script",
          "script_ref": {
            "script_id": "verify-backup",
            "version": "1.0.0"
          }
        },
        {
          "step_number": 3,
          "type": "playbook",
          "playbook_ref": {
            "playbook_id": "stop-rds-instance",
            "version": "2.0.0"
          }
        }
      ]
    }
```

**Characteristics**:
- Mix of script_refs + playbook_refs
- Most flexible type
- Allows custom logic between orchestrated steps

---

## How It Works: The Intelligence Flow

### Overview

```
Master Agent (Intent Classification)
    ↓
Structured JSON Input
    ↓
┌─────────────────────────────────────────────────────┐
│ PLAYBOOK AGENT                                       │
├─────────────────────────────────────────────────────┤
│ STEP 1: RAG Vector Search                            │
│  Find candidate playbooks using embeddings          │
├─────────────────────────────────────────────────────┤
│ STEP 2: LLM Ranking & Reasoning                     │
│  LLM evaluates each candidate with context          │
├─────────────────────────────────────────────────────┤
│ STEP 3: Package & Return                            │
│  Send playbooks with explain plan + scripts         │
└─────────────────────────────────────────────────────┘
    ↓
Client receives ranked playbooks ready to execute
```

**Key Point**: The Playbook Agent receives **structured JSON** from the Master Agent, not raw user queries. Intent classification has already been done.

---

### Input Format (From Master Agent)

The Playbook Agent receives this structured JSON from the Master Agent, which has already performed intent classification and parameter extraction:

```json
{
  "action": "stop",
  "cloud_provider": "aws",
  "resource_types": ["rds::instance"],
  "filters": {"environment": "production"},
  "use_case": "cost-optimization",
  "time_based": true,
  "keywords": ["weekend", "cost-saving", "automated-shutdown", "snapshot"],

  "extracted_parameters": {
    "instance_id": {
      "value": null,
      "source": "not_provided",
      "candidates": ["prod-db-01", "prod-db-02"],
      "extraction_confidence": 0.0
    },
    "region": {
      "value": "us-east-1",
      "source": "user_context",
      "extraction_confidence": 0.95
    },
    "time_window": {
      "value": "weekend",
      "source": "user_query",
      "extraction_confidence": 0.98
    }
  },

  "user_context": {
    "current_resource": {
      "type": "rds::instance",
      "id": "prod-db-01",
      "region": "us-east-1",
      "tags": {
        "environment": "production",
        "team": "platform"
      }
    },
    "user_role": "devops_engineer",
    "organization_id": "abc123"
  }
}
```

**Key Fields Explained:**

| Field | Purpose | Example |
|-------|---------|---------|
| **`action`** | Primary operation to perform | `"stop"`, `"start"`, `"backup"`, `"scale"` |
| **`cloud_provider`** | Target cloud platform | `"aws"`, `"gcp"`, `"azure"` |
| **`resource_types`** | Specific resource types to operate on | `["rds::instance"]`, `["ec2::instance"]` |
| **`filters`** | Additional filtering criteria | `{"environment": "production"}` |
| **`use_case`** | Business reason for action | `"cost-optimization"`, `"disaster-recovery"` |
| **`time_based`** | Whether action is time-sensitive | `true` for scheduled operations |
| **`keywords`** | Semantic tags from user query | `["weekend", "cost-saving"]` |
| **`extracted_parameters`** | Parameters extracted by Master Agent | Pre-filled values with confidence scores |
| **`user_context`** | Current user's environment and resource | Active resource, role, organization |

**Parameter Extraction by Master Agent:**

The Master Agent extracts potential parameter values from:
1. **User query**: "shut down prod-db-01 for the weekend"
   - Extracts: `instance_id = "prod-db-01"`, `time_window = "weekend"`
2. **User context**: Currently viewing resource in UI
   - Extracts: `region = "us-east-1"`, `current_resource = prod-db-01`
3. **User estate**: Available resources
   - Provides: `candidates = ["prod-db-01", "prod-db-02"]`

**How Playbook Agent Uses This:**

1. **Search**: Uses `action`, `cloud_provider`, `resource_types`, `keywords` for RAG search
2. **Ranking**: Considers `use_case`, `filters`, `time_based` when ranking playbooks
3. **Parameter Pre-filling**: Passes `extracted_parameters` to returned playbooks
4. **Context Analysis**: LLM uses `user_context` to predict which parameters can be extracted

**Example: From User Query to Structured Input**

```
USER QUERY:
"I need to shut down prod-db-01 this weekend to save costs"

↓ Master Agent Processing ↓

STRUCTURED INPUT TO PLAYBOOK AGENT:
{
  "action": "stop",
  "cloud_provider": "aws",
  "resource_types": ["rds::instance"],
  "keywords": ["weekend", "cost-saving", "shutdown"],
  "extracted_parameters": {
    "instance_id": {
      "value": "prod-db-01",
      "source": "user_query",
      "extraction_confidence": 0.99
    }
  }
}
```

---

### STEP 1: RAG Vector Search

**Purpose**: Find candidate playbooks using semantic similarity

**Process**:

1. **Generate Embedding** from enhanced query:
```rust
let enhanced_query = format!(
    "{} {} {} {}",
    intent.action,
    intent.resource_types.join(" "),
    intent.use_case,
    intent.keywords.join(" ")
);
// "stop rds cost-optimization weekend cost-saving automated-shutdown snapshot"

let query_vector = embedder.embed(&enhanced_query).await?;
// Returns: [0.234, -0.567, 0.123, ...] (384 dimensions)
```

2. **Search Escher Library** (global collection):
```rust
let escher_results = storage.search_points(
    "escher_library",
    query_vector,
    Filter::must([
        Filter::match("cloud_provider", "aws"),
        Filter::match("resource_types", "rds::instance"),
        Filter::match("status", "active"),
    ]),
    limit=10
).await?;
```

**Results**:
```
[
  {
    playbook_id: "escher-aws-rds-stop",
    similarity_score: 0.87,
    metadata: { /* ... */ }
  },
  {
    playbook_id: "escher-aws-rds-weekend-automation",
    similarity_score: 0.82,
    metadata: { /* ... */ }
  }
]
```

3. **Search Tenant Playbooks** (user's custom):
```rust
let tenant_results = storage.search_points(
    "tenant_abc123_playbooks",
    query_vector,
    Filter::must([
        Filter::match("cloud_provider", "aws"),
        Filter::match("status", "active"),
    ]),
    limit=10
).await?;
```

**Results**:
```
[
  {
    playbook_id: "user-weekend-shutdown",
    similarity_score: 0.91,
    metadata: { /* ... */ }
  }
]
```

4. **Merge & Deduplicate**:
```rust
let candidates = merge_results(escher_results, tenant_results);
// Total: 3 unique playbooks
```

---

### STEP 2: LLM Ranking & Reasoning

**Purpose**: Evaluate each candidate with full context and explain WHY

**LLM Prompt** (sent to Claude/GPT):
```
You are an expert cloud automation advisor. Evaluate these playbooks and rank them for this user's need.

USER QUERY: "Stop production RDS for weekend"

EXTRACTED INTENT:
- Action: stop
- Cloud: aws
- Resource: rds::instance
- Use case: cost-optimization
- Context: Weekend automation for production

CANDIDATE PLAYBOOKS:

1. user-weekend-shutdown (v1.2.0)
   Source: USER_CUSTOM
   Description: "Automated workflow for stopping production RDS instances over weekends..."
   Success Rate: 100% (12/12 executions)
   Prerequisites: ["RDS must have no active connections", "Snapshot policy configured"]
   Keywords: ["production", "database", "weekend", "cost-saving"]

2. escher-aws-rds-stop (v1.2.0)
   Source: ESCHER_LIBRARY
   Description: "Safely stop RDS instance with pre-flight checks..."
   Success Rate: 98% (1547 executions)
   Prerequisites: ["Valid RDS instance", "Stop permission"]
   Keywords: ["rds", "stop", "safe-shutdown"]

3. escher-aws-rds-weekend-automation (v1.5.0)
   Source: ESCHER_LIBRARY
   Description: "Complete weekend cost optimization with start/stop scheduling..."
   Success Rate: 97% (892 executions)
   Prerequisites: ["CloudWatch Events configured", "SNS topic for notifications"]
   Keywords: ["weekend", "automation", "schedule", "cost-saving"]

RANKING RULES:
1. USER_CUSTOM takes precedence (user knows their environment)
2. ESCHER_LIBRARY are battle-tested (prefer if no custom solution)
3. Consider: success_rate, execution_count, prerequisites match, keyword overlap

For each playbook, provide:
- Rank (1, 2, 3)
- Confidence score (0-1)
- Reason (2-3 sentences explaining why this playbook fits)

Return JSON array sorted by rank.
```

**LLM Output**:
```json
[
  {
    "rank": 1,
    "playbook_id": "user-weekend-shutdown",
    "version": "1.2.0",
    "source": "tenant",
    "confidence": 0.95,
    "reason": "This is your team's custom playbook specifically designed for weekend RDS shutdowns in production. It has a perfect success rate (12/12) and includes environment-specific logic that Escher library playbooks don't have. Since it's built for your exact use case, it should be your first choice."
  },
  {
    "rank": 2,
    "playbook_id": "escher-aws-rds-weekend-automation",
    "version": "1.5.0",
    "source": "escher_library",
    "confidence": 0.88,
    "reason": "This is Escher's comprehensive weekend automation solution with scheduling, notifications, and automatic restart on Monday. It's battle-tested (97% success, 892 uses) and handles the complete weekend workflow. Use this if you want a more complete solution with monitoring."
  },
  {
    "rank": 3,
    "playbook_id": "escher-aws-rds-stop",
    "version": "1.2.0",
    "source": "escher_library",
    "confidence": 0.72,
    "reason": "This is a simple RDS stop playbook without weekend-specific logic or scheduling. While it's very reliable (98% success, 1547 uses), it doesn't handle the time-based automation you need. Consider this only if you want manual control."
  }
]
```

**Key Features**:
- LLM considers **full context** (not just keywords)
- Explains **trade-offs** between options
- Respects **precedence rules** but explains when library might be better
- Natural language **reasoning** helps user decide

---

### STEP 3: Package & Return to Client

**Purpose**: Send complete playbooks ready for execution

**Response to Client**:
```json
{
  "query": "Stop production RDS for weekend",
  "results": [
    {
      "rank": 1,
      "playbook_id": "user-weekend-shutdown",
      "version": "1.2.0",
      "source": "tenant",
      "confidence": 0.95,
      "reason": "This is your team's custom playbook...",

      "metadata": {
        "name": "Weekend DB Shutdown",
        "description": "Automated workflow for stopping production RDS...",
        "cloud_provider": ["aws"],
        "resource_types": ["rds::instance"],
        "estimated_impact": {
          "downtime_minutes": 10,
          "cost_savings_monthly_usd": 120
        }
      },

      "parameters": {
        "instance_id": {
          "type": "string",
          "required": true,
          "extraction_hint": {
            "source": "user_estate",
            "estate_query": {
              "resource_type": "rds::instance",
              "filters": {"tags.environment": "production"}
            }
          }
        },
        "skip_snapshot": {
          "type": "boolean",
          "default": false
        }
      },

      "explain_plan": {
        "what_it_does": "Creates a snapshot of the RDS instance and stops it gracefully",
        "what_happens": [
          {
            "step": 1,
            "action": "Verify RDS instance state",
            "duration_seconds": 5
          },
          {
            "step": 2,
            "action": "Create snapshot",
            "duration_seconds": 30
          },
          {
            "step": 3,
            "action": "Stop RDS instance",
            "duration_seconds": 120
          }
        ],
        "risks": [
          {
            "risk": "Active connections disrupted",
            "severity": "medium",
            "mitigation": "Pre-flight check verifies no active connections"
          }
        ]
      },

      "scripts": {
        "shell": {
          "code": "#!/bin/bash\nset -e\n\n# Weekend RDS Shutdown Script\n...",
          "entry_point": "main.sh"
        },
        "python": {
          "code": "import boto3\n\ndef stop_rds_instance():\n    ...",
          "entry_point": "stop_rds.py"
        }
      }
    },

    // Rank 2 and 3 playbooks...
  ]
}
```

**What Client Gets**:
- ✅ Ranked list (top 3-5 playbooks)
- ✅ Full explain plan for each
- ✅ Complete scripts (shell, python, terraform, cloudformation)
- ✅ Parameters with parameter extraction hints
- ✅ LLM-generated reasoning
- ✅ Confidence scores

**Client's Next Steps**:
1. User reviews explain plans
2. Selects preferred playbook (usually rank 1)
3. Reviews/fills parameters (Master Agent extraction hints help)
4. Execution Engine runs the script

---

## Playbook Lifecycle & States

Every playbook has a **status** field that controls its behavior in search, ranking, and execution. Understanding playbook states is critical for the agent's decision-making.

---

### 10 Playbook Status Values

| Status | Description | Visible in Search? | Rank Priority | Used By |
|--------|-------------|-------------------|---------------|---------|
| **draft** | Being created/edited | ❌ No | N/A | User creating playbook |
| **ready** | Saved locally, not uploaded | ❌ No (local only) | N/A | Local playbooks |
| **active** | Live and ready for use | ✅ Yes | **High** | Team (if shared) or user (if local) |
| **deprecated** | Old version, newer available | ⚠️ Yes (with warning) | Low | Legacy playbooks |
| **archived** | No longer available | ❌ No | N/A | Historical records |
| **pending_review** | Uploaded, awaiting approval | ⚠️ Yes (with warning) | Very Low | Review workflow |
| **approved** | Reviewed and validated | ✅ Yes | **High** | Trusted custom playbooks |
| **rejected** | Rejected by review | ❌ No | N/A | Failed review |
| **broken** | Known to fail | ❌ No | N/A | Disabled playbooks |
| **needs_update** | Works but outdated | ⚠️ Yes (with warning) | Medium | Maintenance |

---

### State Transition Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ USER CREATES PLAYBOOK                                           │
└─────────────────────────────────────────────────────────────────┘
    ↓
┌─────────┐
│  draft  │ ← User is actively editing
└─────────┘
    ↓ (User saves)
┌─────────┐
│  ready  │ ← Saved locally, not shared
└─────────┘
    │
    ├───→ (User keeps private)
    │     ┌─────────┐
    │     │ active  │ ← local_only: Only visible to this user
    │     └─────────┘
    │
    └───→ (User uploads for review)
          ┌──────────────────┐
          │ pending_review   │ ← uploaded_for_review: Team can see
          └──────────────────┘
               │
               ├───→ (Approved)
               │     ┌──────────┐
               │     │ approved │ ← uploaded_trusted: High rank
               │     └──────────┘
               │          ↓
               │     ┌─────────┐
               │     │ active  │ ← Available to entire team
               │     └─────────┘
               │          │
               │          ├───→ (New version released)
               │          │     ┌──────────────┐
               │          │     │ deprecated   │ ← Old version
               │          │     └──────────────┘
               │          │
               │          ├───→ (Bug discovered)
               │          │     ┌─────────┐
               │          │     │ broken  │ ← Hidden from search
               │          │     └─────────┘
               │          │
               │          └───→ (Outdated but working)
               │                ┌───────────────┐
               │                │ needs_update  │ ← Lower rank
               │                └───────────────┘
               │
               └───→ (Rejected)
                     ┌──────────┐
                     │ rejected │ ← Hidden from search
                     └──────────┘

┌─────────────────────────────────────────────────────────────────┐
│ ESCHER LIBRARY PLAYBOOKS                                        │
└─────────────────────────────────────────────────────────────────┘
    ↓
┌─────────┐
│ active  │ ← All Escher playbooks start as active
└─────────┘
    │
    ├───→ (New version)
    │     ┌──────────────┐
    │     │ deprecated   │ ← Old version
    │     └──────────────┘
    │
    └───→ (No longer supported)
          ┌──────────┐
          │ archived │ ← Removed from search
          └─────���────┘
```

---

### Status in Action: How Agent Uses States

#### 1. **Search Filtering**

The agent **only searches** playbooks with specific statuses:

```rust
// In search_playbooks()
let status_filter = Filter::should([
    Filter::match("status", "active"),      // Primary
    Filter::match("status", "approved"),    // After review
    Filter::match("status", "deprecated"),  // With warning
    Filter::match("status", "needs_update"), // With warning
    Filter::match("status", "pending_review"), // With warning
]);

// NEVER search:
// - draft (incomplete)
// - ready (local only)
// - rejected (failed review)
// - broken (known to fail)
// - archived (removed)
```

**Example RAG Query**:
```rust
storage.search_points(
    "tenant_abc123_playbooks",
    query_vector,
    Filter::must([
        Filter::match("cloud_provider", "aws"),
        Filter::should([
            Filter::match("status", "active"),
            Filter::match("status", "approved"),
        ]),
    ]),
    limit=10
).await?;
```

---

#### 2. **Ranking by Status**

Different statuses get different rank bonuses:

```rust
fn calculate_rank_bonus(status: &str) -> f32 {
    match status {
        "active" => 50.0,       // Full bonus
        "approved" => 50.0,     // Full bonus (same as active)
        "pending_review" => 5.0, // Very low (experimental)
        "deprecated" => 10.0,   // Lower than active
        "needs_update" => 20.0, // Medium
        _ => 0.0,               // No bonus
    }
}
```

**Example Ranking**:

| Playbook | Base Score | Status | Status Bonus | Final Score |
|----------|------------|--------|--------------|-------------|
| user-weekend-shutdown | 92 | active | +50 | **142** |
| user-old-shutdown | 88 | deprecated | +10 | 98 |
| ai-generated-shutdown | 85 | pending_review | +5 | 90 |

---

#### 3. **LLM Reasoning with Status**

When LLM ranks playbooks (STEP 3), status is included in the context:

**LLM Prompt** (excerpt):
```
CANDIDATE PLAYBOOKS:

1. user-weekend-shutdown (v1.2.0)
   Status: ACTIVE ✅
   Description: "Automated weekend RDS shutdown..."
   Success Rate: 100% (12/12)
   → This is trusted and battle-tested

2. user-weekend-shutdown (v1.0.0)
   Status: DEPRECATED ⚠️
   Description: "Original version, replaced by v1.2.0"
   Success Rate: 100% (5/5)
   → Old version, prefer newer v1.2.0

3. ai-generated-rds-stop (v1.0.0)
   Status: PENDING_REVIEW ⚠️
   Description: "AI-generated playbook from user query..."
   Success Rate: N/A (untested)
   → Untested, use with caution

RANKING RULES:
- ACTIVE/APPROVED: Highest trust
- DEPRECATED: Prefer newer version
- PENDING_REVIEW: Experimental, warn user
- NEEDS_UPDATE: Works but outdated
```

**LLM Output**:
```json
[
  {
    "rank": 1,
    "playbook_id": "user-weekend-shutdown",
    "version": "1.2.0",
    "status": "active",
    "confidence": 0.95,
    "reason": "This is the latest active version (v1.2.0) with perfect success rate. Status is ACTIVE, meaning it's trusted and battle-tested."
  },
  {
    "rank": 2,
    "playbook_id": "user-weekend-shutdown",
    "version": "1.0.0",
    "status": "deprecated",
    "confidence": 0.65,
    "reason": "This is an older version (v1.0.0) that has been DEPRECATED. While it worked well (5/5 success), you should use v1.2.0 instead."
  },
  {
    "rank": 3,
    "playbook_id": "ai-generated-rds-stop",
    "version": "1.0.0",
    "status": "pending_review",
    "confidence": 0.40,
    "reason": "This is an AI-generated playbook with status PENDING_REVIEW. It hasn't been tested yet. Use only if you want to experiment and can verify the code."
  }
]
```

---

### Review Workflow: uploaded_for_review

When a user creates a custom playbook and uploads it, it enters the **review workflow**:

```
User creates playbook
    ↓
User clicks "Share with Team"
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Client)                          │
├────────────────────────────────────────────────────┤
│ 1. Upload scripts to S3                            │
│ 2. Set storage_strategy: "uploaded_for_review"     │
│ 3. Set status: "pending_review"                    │
│ 4. Publish metadata to server                      │
└────────────────────────────────────────────────────┘
    ↓
Server upserts to tenant_abc123_playbooks
    status: "pending_review"
    ↓
┌────────────────────────────────────────────────────┐
│ Team Lead Reviews Playbook                         │
├────────────────────────────────────────────────────┤
│ Option 1: APPROVE                                  │
│   POST /api/playbook/.../status                    │
│   { "status": "approved" }                         │
│   → storage_strategy: "uploaded_trusted"           │
│   → status: "approved" → "active"                  │
│   → High rank in search                            │
│                                                     │
│ Option 2: REJECT                                   │
│   POST /api/playbook/.../status                    │
│   { "status": "rejected", "reason": "..." }        │
│   → Hidden from search                             │
│   → Creator notified                               │
└────────────────────────────────────────────────────┘
```

**Agent Behavior During Review**:
- ⚠️ **pending_review** playbooks ARE visible in search
- ⚠️ Ranked VERY LOW (bonus: +5)
- ⚠️ Returned with warning: "⚠️ UNTESTED - Pending Review"
- ✅ Once approved → status: "approved" → higher rank

---

### Status Update API

```rust
pub async fn update_status(
    &self,
    tenant_id: &str,
    playbook_id: &str,
    version: &str,
    new_status: &str,
) -> Result<()> {
    let collection_name = format!("tenant_{}_playbooks", tenant_id);
    let point_id = format!("{}-{}", playbook_id, version);

    // Get existing point
    let point = self.storage.get_point(&collection_name, &point_id).await?;
    let mut payload = point.payload;

    // Update status
    payload["status"] = new_status.into();

    // If approving, also update storage_strategy
    if new_status == "approved" {
        payload["storage_strategy"] = "uploaded_trusted".into();
    }

    // Update timestamp
    payload["updated_at"] = SystemTime::now()
        .duration_since(UNIX_EPOCH)?
        .as_secs()
        .into();

    // Upsert back
    self.storage.upsert_point(
        &collection_name,
        &point_id,
        point.vector,
        payload,
    ).await?;

    Ok(())
}
```

---

### State Management Examples

#### Example 1: Deprecate Old Version

```rust
// User uploads new version v1.3.0
publish_metadata(tenant_id, "user-weekend-shutdown", "1.3.0", "active");

// Automatically deprecate v1.2.0
update_status(tenant_id, "user-weekend-shutdown", "1.2.0", "deprecated");
```

**Agent Search Result**:
```json
{
  "results": [
    {
      "playbook_id": "user-weekend-shutdown",
      "version": "1.3.0",
      "status": "active",
      "confidence": 0.95,
      "reason": "Latest version with new features"
    },
    {
      "playbook_id": "user-weekend-shutdown",
      "version": "1.2.0",
      "status": "deprecated",
      "confidence": 0.60,
      "reason": "⚠️ Old version - v1.3.0 is available"
    }
  ]
}
```

---

#### Example 2: Mark as Broken

```rust
// Execution fails 3 times in a row
if failure_count >= 3 {
    update_status(tenant_id, "user-broken-playbook", "1.0.0", "broken");
}
```

**Agent Search Result**:
```
(playbook is HIDDEN from search)
```

**User Dashboard**:
```
⚠️ Your playbook "user-broken-playbook" has been marked as BROKEN
   Reason: Failed 3 consecutive executions
   Action: Please review and fix, or delete
```

---

#### Example 3: Needs Update

```rust
// Playbook works but uses deprecated AWS CLI v1
if uses_deprecated_api {
    update_status(tenant_id, "user-old-cli", "1.0.0", "needs_update");
}
```

**Agent Search Result**:
```json
{
  "playbook_id": "user-old-cli",
  "version": "1.0.0",
  "status": "needs_update",
  "confidence": 0.70,
  "reason": "⚠️ Works but uses deprecated AWS CLI v1. Consider updating to v2."
}
```

---

### Status in RAG Collection Schema

Each playbook point in Qdrant includes:

```json
{
  "id": "user-weekend-shutdown-v1.2.0",
  "vector": [0.123, -0.456, ...],
  "payload": {
    "playbook_id": "user-weekend-shutdown",
    "version": "1.2.0",
    "status": "active",  // ← CRITICAL FIELD
    "storage_strategy": "uploaded_trusted",
    "created_at": 1726329600,
    "updated_at": 1728391162,
    "execution_count": 47,
    "success_count": 47,
    "success_rate": 1.0,
    "last_executed_at": 1728388800
  }
}
```

---

### Status Precedence Rules

When multiple versions exist, agent prefers:

1. **active** > deprecated > needs_update
2. **Higher version number** (v1.3.0 > v1.2.0)
3. **Higher success rate** (100% > 95%)
4. **More recent** (updated < 30 days)

**Example**:
```
v1.3.0 (active, 95% success, 2 days old)     → Rank 1
v1.2.0 (deprecated, 100% success, 90 days)  → Rank 2
v1.0.0 (deprecated, 100% success, 180 days) → Rank 3
```

---

## Storage Strategies

A playbook can have one of 4 storage strategies:

### 1. `local_only` (Private)

**Use Case**: Sensitive playbooks that should never leave the user's machine

**Storage**:
- ✅ Local disk: `~/.escher/playbooks/local/`
- ✅ Local RAG: `user_playbooks` collection (encrypted)
- ❌ Server: NOT uploaded
- ❌ Server RAG: NOT published

**Behavior**:
- Playbook only visible to local client
- Can be executed locally
- Not shared with team or server
- Full encryption at rest

**Example**: Playbooks with hardcoded credentials, proprietary scripts

```json
{
  "storage_strategy": "local_only",
  "synced_to_server": false,
  "encrypted_locally": true
}
```

---

### 2. `uploaded_for_review` (Pending Approval)

**Use Case**: User-created playbooks submitted for team review before trust

**Storage**:
- ✅ Local disk: `~/.escher/playbooks/synced/`
- ✅ Local RAG: `user_playbooks` collection
- ✅ Server S3: `s3://escher-tenant-data/tenants/{id}/playbooks/{playbook_id}/{version}/`
- ✅ Server RAG: `tenant_{id}_playbooks` collection (metadata only)

**Metadata in Server RAG**:
```json
{
  "playbook_id": "user-weekend-shutdown",
  "version": "1.2.0",
  "name": "Weekend DB Shutdown",
  "status": "pending_review",
  "storage_strategy": "uploaded_for_review",
  "scripts": {
    "shell": "s3://escher-tenant-data/tenants/abc123/playbooks/user-weekend-shutdown/v1.2.0/shell/main.sh",
    "python": "s3://escher-tenant-data/tenants/abc123/playbooks/user-weekend-shutdown/v1.2.0/python/stop_rds.py"
  }
}
```

**Behavior**:
- Playbook visible in search results (lower rank)
- Can be executed with warning
- Team can review and approve/reject
- Status transitions: `pending_review` → `approved` or `rejected`

**Example**: User's custom automation, AI-generated playbooks

---

### 3. `uploaded_trusted` (Team Shared)

**Use Case**: Approved user playbooks shared with entire team

**Storage**:
- ✅ Local disk: `~/.escher/playbooks/synced/`
- ✅ Local RAG: `user_playbooks` collection
- ✅ Server S3: `s3://escher-tenant-data/tenants/{id}/playbooks/{playbook_id}/{version}/`
- ✅ Server RAG: `tenant_{id}_playbooks` collection

**Metadata in Server RAG**:
```json
{
  "playbook_id": "user-weekend-shutdown",
  "version": "1.2.0",
  "status": "approved",
  "storage_strategy": "uploaded_trusted",
  "execution_count": 47,
  "success_count": 47,
  "success_rate": 1.0
}
```

**Behavior**:
- Playbook highly ranked in search (USER_CUSTOM precedence)
- All team members can execute
- Execution stats tracked
- Can be marked as "deprecated" when newer version available

**Example**: Battle-tested custom playbooks with proven success

---

### 4. `using_default` (Escher Library)

**Use Case**: Using Escher's curated library playbooks

**Storage**:
- ❌ Local disk: NOT stored (downloaded to cache on-demand)
- ✅ Local cache: `~/.escher/cache/scripts/escher-{playbook_id}-{version}/` (TTL: 7 days)
- ✅ Server S3: `s3://escher-library/playbooks/{playbook_id}/{version}/`
- ✅ Server RAG: `escher_library` collection (global, read-only)

**Metadata in Server RAG**:
```json
{
  "playbook_id": "escher-aws-rds-stop",
  "version": "1.2.0",
  "name": "AWS RDS Stop Instance",
  "status": "active",
  "author": "escher_library",
  "storage_strategy": "using_default",
  "success_rate": 0.98,
  "execution_count": 1547,
  "scripts": {
    "shell": "s3://escher-library/playbooks/aws-rds-stop/v1.2.0/shell/stop.sh",
    "python": "s3://escher-library/playbooks/aws-rds-stop/v1.2.0/python/stop_rds.py"
  }
}
```

**Behavior**:
- Playbook available to all tenants
- High success rate (98%+)
- Validated by Escher team
- Scripts downloaded to cache on first use
- Ranked lower than USER_CUSTOM but higher than AI_GENERATED

**Example**: Standard AWS operations (stop RDS, scale EC2, backup S3)

---

### Storage Strategy Decision Tree

```
User creates/receives playbook
    │
    ├─→ Contains sensitive data?
    │   YES → local_only
    │
    ├─→ Custom script, needs team review?
    │   YES → uploaded_for_review
    │   │
    │   └─→ After approval → uploaded_trusted
    │
    ├─→ AI-generated, untested?
    │   YES → uploaded_for_review (requires testing)
    │
    └─→ Using Escher Library?
        YES → using_default (download on-demand)
```

---

## S3 Storage Structure

Playbooks are stored in S3 with a well-organized structure for easy access and management.

### Two S3 Buckets

#### 1. Escher Library Bucket (Global)

**Bucket**: `s3://escher-library/`
**Owner**: Escher (read-only for all tenants)
**Purpose**: Curated, battle-tested playbooks from Escher

```
s3://escher-library/
├── scripts/                        # ← NEW: Reusable script library
│   ├── create-rds-snapshot/
│   │   ├── v1.0.0/
│   │   │   ├── metadata.json
│   │   │   └── code/
│   │   │       ├── bash/
│   │   │       │   ├── script.sh
│   │   │       │   └── README.md
│   │   │       ├── python/
│   │   │       │   ├── main.py
│   │   │       │   ├── requirements.txt
│   │   │       │   └── README.md
│   │   │       ├── node/
│   │   │       │   ├── index.js
│   │   │       │   ├── package.json
│   │   │       │   └── README.md
│   │   │       ├── terraform/
│   │   │       │   ├── main.tf
│   │   │       │   ├── variables.tf
│   │   │       │   ├── outputs.tf
│   │   │       │   └── README.md
│   │   │       └── cloudformation/
│   │   │           ├── template.yaml
│   │   │           └── README.md
│   │   ├── v1.1.0/
│   │   └── v2.1.0/
│   │
│   ├── stop-rds-instance/
│   │   ├── v1.0.0/
│   │   └── v2.0.0/
│   │       ├── metadata.json
│   │       └── code/
│   │           ├── bash/script.sh
│   │           ├── python/main.py
│   │           └── node/index.js
│   │
│   ├── check-rds-state/
│   ├── verify-backup/
│   ├── start-ec2-instance/
│   ├── stop-ec2-instance/
│   └── backup-s3-bucket/
│
├── playbooks/                      # ← Orchestration only (references scripts)
│   ├── stop-rds-with-snapshot/
│   │   ├── v1.0.0/
│   │   │   ├── metadata.json
│   │   │   ├── parameters.json
│   │   │   ├── explain_plan.json
│   │   │   └── orchestration.json  # References script IDs
│   │   ├── v1.1.0/
│   │   └── v1.2.0/
│   │
│   ├── weekend-rds-automation/
│   │   ├── v1.0.0/
│   │   └── v1.5.0/
│   │
│   ├── backup-and-stop-rds/
│   ├── start-dev-environment/
│   ├── stop-dev-environment/
│   ├── scale-ec2-fleet/
│   └── rotate-rds-snapshots/
│
└── checksums/
    ├── scripts/
    │   ├── create-rds-snapshot-v2.1.0.sha256
    │   └── stop-rds-instance-v2.0.0.sha256
    └── playbooks/
        └── stop-rds-with-snapshot-v1.2.0.sha256
```

**Access**:
- All tenants: **READ-ONLY**
- Escher team: **WRITE** (for updates/new playbooks)

**S3 Policy Example**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::escher-library/*",
        "arn:aws:s3:::escher-library"
      ]
    }
  ]
}
```

---

#### 2. Tenant Data Bucket (Per-Tenant)

**Bucket**: `s3://escher-tenant-data/`
**Owner**: Escher (access controlled per tenant)
**Purpose**: Tenant-specific playbooks and data

```
s3://escher-tenant-data/
├── tenants/
│   ├── abc123/                    # Tenant ID
│   │   ├── playbooks/
│   │   │   ├── user-weekend-shutdown/
│   │   │   │   ├── v1.0.0/
│   │   │   │   │   ├── metadata.json
│   │   │   │   │   ├── parameters.json
│   │   │   │   │   ├── explain_plan.json
│   │   │   │   │   ├── shell/
│   │   │   │   │   │   └── main.sh
│   │   │   │   │   ├── cloudformation/
│   │   │   │   │   │   └── template.yaml
│   │   │   │   │   ├── terraform/
│   │   │   │   │   │   ├── main.tf
│   │   │   │   │   │   ├── variables.tf
│   │   │   │   │   │   └── outputs.tf
│   │   │   │   │   └── python/
│   │   │   │   │       ├── stop_rds.py
│   │   │   │   │       └── requirements.txt
│   │   │   │   ├── v1.1.0/
│   │   │   │   └── v1.2.0/
│   │   │   │
│   │   │   ├── user-backup-strategy/
│   │   │   │   └── v1.0.0/
│   │   │   │
│   │   │   └── ai-generated-cleanup/
│   │   │       └── v1.0.0/
│   │   │
│   │   ├── estate-snapshots/      # (Future) Estate backup snapshots
│   │   └── reports/               # (Future) Generated reports
│   │
│   ├── xyz789/                    # Another tenant
│   │   └── playbooks/
│   │       └── ...
│   │
│   └── def456/
│       └── playbooks/
│           └── ...
│
└── global/
    └── templates/                 # Shared templates (optional)
```

**Access**:
- Tenant `abc123`: **READ/WRITE** to `tenants/abc123/*`
- Tenant `abc123`: **NO ACCESS** to `tenants/xyz789/*`
- Escher server: **READ/WRITE** to all (for serving playbooks)

**S3 Policy Example** (per tenant):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:user/tenant-abc123"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::escher-tenant-data/tenants/abc123/*",
        "arn:aws:s3:::escher-tenant-data"
      ],
      "Condition": {
        "StringLike": {
          "s3:prefix": ["tenants/abc123/*"]
        }
      }
    }
  ]
}
```

---

### S3 Object Metadata

Each S3 object has metadata tags for easier management:

```json
{
  "Metadata": {
    "playbook-id": "user-weekend-shutdown",
    "version": "1.2.0",
    "tenant-id": "abc123",
    "uploaded-by": "user@example.com",
    "uploaded-at": "2025-10-14T10:30:00Z",
    "storage-strategy": "uploaded_trusted",
    "content-type": "application/x-sh",
    "checksum-sha256": "abc123..."
  },
  "Tags": [
    {"Key": "tenant", "Value": "abc123"},
    {"Key": "playbook-id", "Value": "user-weekend-shutdown"},
    {"Key": "version", "Value": "1.2.0"},
    {"Key": "format", "Value": "shell"},
    {"Key": "status", "Value": "active"}
  ]
}
```

---

### File Upload Flow

#### Client → S3 Upload

```
User publishes playbook "user-weekend-shutdown-v1.2.0"
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Client)                          │
├────────────────────────────────────────────────────┤
│ 1. Request pre-signed URL from server              │
│    POST /api/playbook/upload/request               │
│    {                                               │
│      "tenant_id": "abc123",                        │
│      "playbook_id": "user-weekend-shutdown",       │
│      "version": "1.2.0",                           │
│      "files": [                                    │
│        "metadata.json",                            │
│        "parameters.json",                          │
│        "explain_plan.json",                        │
│        "shell/main.sh",                            │
│        "python/stop_rds.py"                        │
│      ]                                             │
│    }                                               │
└────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Agent (Server)                            │
├────────────────────────────────────────────────────┤
│ 1. Validate tenant permissions                     │
│ 2. Generate pre-signed URLs (15 min expiry)        │
│    s3_client.generate_presigned_url(               │
│      'put_object',                                 │
│      Bucket='escher-tenant-data',                  │
│      Key='tenants/abc123/playbooks/...',           │
│      ExpiresIn=900                                 │
│    )                                               │
│                                                     │
│ 3. Return pre-signed URLs                          │
│    {                                               │
│      "urls": {                                     │
│        "metadata.json": "https://s3...?signature", │
│        "shell/main.sh": "https://s3...?signature"  │
│      }                                             │
│    }                                               │
└────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Client)                          │
├────────────────────────────────────────────────────┤
│ 1. Upload files directly to S3 using pre-signed    │
│    PUT https://s3.../metadata.json?signature       │
│    PUT https://s3.../shell/main.sh?signature       │
│                                                     │
│ 2. After upload, notify server                     │
│    POST /api/playbook/metadata/publish             │
│    {                                               │
│      "tenant_id": "abc123",                        │
│      "playbook_id": "user-weekend-shutdown",       │
│      "version": "1.2.0",                           │
│      "s3_paths": {                                 │
│        "shell": "s3://escher-tenant-data/...",     │
│        "python": "s3://escher-tenant-data/..."     │
│      }                                             │
│    }                                               │
└────────────────────────────────────────────────────┘
    ↓
Playbook Agent updates RAG with metadata + S3 paths
```

---

### Script Delivery: Embedded in Response

**IMPORTANT**: Scripts are NOT downloaded separately by the client. They are embedded in the playbook search response.

```
Server Response Structure:
{
  "playbooks": [
    {
      "playbook_id": "user-weekend-shutdown",
      "version": "1.2.0",
      "steps": [
        {
          "id": "step-1",
          "script": "#!/bin/bash\n# Complete script embedded here",
          "parameters": ["{{instance_id}}"]
        }
      ],
      "parameters": {
        "instance_id": "prod-db-01",    ← Pre-filled by Master Agent
        "approval_email": ""            ← Placeholder (user fills)
      }
    }
  ]
}
```

**Client responsibilities**:
- Receives complete playbook with embedded scripts
- Shows parameters to user (some pre-filled, some placeholders)
- User manually fills any empty parameters
- Writes scripts to temporary files for execution
- Executes scripts locally with user's AWS credentials

**Why embedded scripts?**
- ✅ No S3 permissions needed on client
- ✅ Single API call gets everything
- ✅ Simpler client implementation
- ✅ No download errors or network issues during execution
---

### S3 Lifecycle Policies

#### Escher Library Bucket

```json
{
  "Rules": [
    {
      "Id": "keep-all-versions",
      "Status": "Enabled",
      "Prefix": "playbooks/",
      "Expiration": {
        "Days": 0
      }
    }
  ]
}
```
**Note**: No expiration for Escher Library - all versions kept indefinitely

---

#### Tenant Data Bucket

```json
{
  "Rules": [
    {
      "Id": "archive-old-playbooks",
      "Status": "Enabled",
      "Prefix": "tenants/",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ]
    },
    {
      "Id": "delete-ai-generated-after-30-days",
      "Status": "Enabled",
      "Prefix": "tenants/",
      "TagFilters": [
        {
          "Key": "author",
          "Value": "ai_generated"
        },
        {
          "Key": "status",
          "Value": "rejected"
        }
      ],
      "Expiration": {
        "Days": 30
      }
    }
  ]
}
```

---

### S3 Cost Optimization

| Storage Class | Use Case | Cost (per GB/month) |
|---------------|----------|---------------------|
| **STANDARD** | Active playbooks (< 90 days) | $0.023 |
| **STANDARD_IA** | Archived playbooks (90-365 days) | $0.0125 |
| **GLACIER** | Old versions (> 365 days) | $0.004 |

**Estimated Costs**:
- 1,000 playbooks × 4 formats × 5 KB/format = 20 MB
- **STANDARD**: $0.00046/month
- **STANDARD_IA** (after 90 days): $0.00025/month
- **GLACIER** (after 365 days): $0.00008/month

**Tenant with 500 custom playbooks**: ~$0.23/month in S3 costs

---

### S3 Versioning

Both buckets have **versioning enabled** to prevent accidental deletion:

```bash
aws s3api put-bucket-versioning \
  --bucket escher-library \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-versioning \
  --bucket escher-tenant-data \
  --versioning-configuration Status=Enabled
```

**Benefits**:
- Accidental deletion recovery
- Version history for auditing
- Rollback to previous playbook versions

---

## Architecture

### storage-service: The Reusable RAG Crate

**storage-service** is a **Rust crate** (library) that provides RAG capabilities. It's NOT a separate service - it's just code that gets compiled into applications that need RAG.

```
┌─────────────────────────────────────────────────────────────┐
│ CLIENT (Tauri on Laptop)                                    │
├─────────────────────────────────────────────────────────────┤
│  Cargo.toml:                                                 │
│    storage-service = { path = "../storage-service" }        │
│                                                              │
│  Playbook Service                                           │
│      ↓ uses                                                  │
│  storage-service crate                                      │
│      ↓ configured as                                         │
│  QdrantMode::Embedded                                       │
│  storage_path: "~/.escher/qdrant/"                          │
│      ↓                                                       │
│  Embedded Qdrant (local filesystem)                         │
│    • user_playbooks collection                              │
│    • Full playbooks (encrypted)                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ SERVER (Rust on ECS/EC2)                                    │
├─────────────────────────────────────────────────────────────┤
│  Cargo.toml:                                                 │
│    storage-service = { path = "../storage-service" }        │
│                                                              │
│  Playbook Agent                                             │
│      ↓ uses                                                  │
│  storage-service crate                                      │
│      ↓ configured as                                         │
│  QdrantMode::Embedded                                       │
│  storage_path: "/mnt/qdrant/"  (EBS/EFS volume)             │
│      ↓                                                       │
│  Embedded Qdrant (shared filesystem)                        │
│    • escher_library collection                              │
│    • tenant_abc123_playbooks collection                     │
│    • tenant_xyz789_playbooks collection                     │
│    • Metadata only + S3 paths                               │
└─────────────────────────────────────────────────────────────┘
```

**Key Points:**
- ✅ **Same crate, different config** - Both use embedded Qdrant on filesystem
- ✅ **Client**: `~/.escher/qdrant/` (user's home directory)
- ✅ **Server**: `/mnt/qdrant/` (EBS/EFS mount for persistence)
- ✅ **Future flexibility**: Can switch to `QdrantMode::Remote` if needed

### Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│ PLAYBOOK AGENT (Rust Service)                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Uses: storage-service crate (embedded Qdrant)              │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ ���
│  │ Search Engine                                          │ │
│  │  • Multi-source RAG search                             │ │
│  │  • Query: escher_library + tenant_{id}_playbooks      │ │
│  │  • Merge & deduplicate results                         │ │
│  │  • Apply filters (cloud, resource_type, status)       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Ranking Engine                                         │ │
│  │  • Precedence: USER > ESCHER > AI                      │ │
│  │  • Scoring factors:                                    │ │
│  │    - Author type (user_custom +100, escher +50)       │ │
│  │    - Success rate (* 10)                               │ │
│  │    - Execution count (capped at 100)                   │ │
│  │    - Recency (prefer updated < 30 days)               │ │
│  │    - Vector similarity from RAG                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Metadata Manager                                       │ │
│  │  • Publish: Tenant uploads playbook metadata          │ │
│  │  • Update: Change status, execution stats             │ │
│  │  • Delete: Remove from tenant RAG collection          │ │
│  │  • Collection management: Create tenant collections   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────────────────────────────┐
│ storage-service crate (compiled into Playbook Agent)        │
├─────────────────────────────────────────────────────────────┤
│  • search_points(collection, vector, filter, limit)         │
│  • upsert_point(collection, id, vector, payload)            │
│  • delete_point(collection, id)                             │
│  • create_collection(name, config)                          │
└─────────────────────────────────────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────────────────────────────┐
│ Embedded Qdrant (/mnt/qdrant/ on EBS/EFS)                   │
│  • escher_library (global)                                   │
│  • tenant_abc123_playbooks (tenant-specific)                 │
│  • tenant_xyz789_playbooks (tenant-specific)                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### Flow 1: User Searches for Playbook

```
User: "Stop production RDS for weekend"
    ↓
Frontend → Server: POST /api/playbook/search
    {
      "query": "Stop production RDS for weekend",
      "tenant_id": "abc123",
      "filters": {
        "cloud_provider": "aws",
        "resource_types": ["rds::instance"]
      }
    }
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Agent                                     │
├────────────────────────────────────────────────────┤
│ 1. Generate embedding                              │
│    vector = embed("Stop production RDS...")        │
│                                                     │
│ 2. Search escher_library                           │
│    results_escher = storage.search_points(         │
│      "escher_library",                             │
│      vector,                                       │
│      filter={cloud: aws, resource: rds},           │
│      limit=10                                      │
│    )                                               │
│    → [escher-aws-rds-stop-v1.2.0: score=0.85]      │
│                                                     │
│ 3. Search tenant_abc123_playbooks                  │
│    results_tenant = storage.search_points(         │
│      "tenant_abc123_playbooks",                    │
│      vector,                                       │
│      filter={cloud: aws, resource: rds},           │
│      limit=10                                      │
│    )                                               │
│    → [user-weekend-shutdown-v1.2.0: score=0.92]    │
│                                                     │
│ 4. Merge & Deduplicate                             │
│    candidates = [user-weekend-shutdown, escher-rds]│
│                                                     │
│ 5. Apply Ranking                                   │
│    user-weekend:                                   │
│      base=92, precedence=+100, success=+10         │
│      final=202                                     │
│    escher-rds:                                     │
│      base=85, precedence=+50, success=+9.8         │
│      final=144.8                                   │
│                                                     │
│ 6. Return ranked results                           │
│    [user-weekend-shutdown, escher-rds-stop]        │
└────────────────────────────────────────────────────┘
    ↓
Server → Frontend:
    {
      "results": [
        {
          "playbook_id": "user-weekend-shutdown",
          "version": "1.2.0",
          "source": "tenant",
          "confidence": 0.95,
          "reason": "Your custom playbook (12/12 successes)"
        },
        {
          "playbook_id": "escher-aws-rds-stop",
          "version": "1.2.0",
          "source": "escher_library",
          "confidence": 0.88,
          "reason": "Proven Escher playbook (98% success, 1547 uses)"
        }
      ]
    }
```

### Flow 2: User Reviews and Executes Playbook

```
User selects: "user-weekend-shutdown-v1.2.0" from search results
    ↓
┌────────────────────────────────────────────────────┐
│ Client Receives Complete Playbook from Server     │
├────────────────────────────────────────────────────┤
│ Response includes:                                 │
│   - Playbook metadata                              │
│   - Scripts EMBEDDED in steps (no download needed) │
│   - Parameters with values:                        │
│     • instance_id: "prod-db-01" ← Filled by Master │
│     • region: "us-east-1"       ← Filled by Master │
│     • approval_email: ""        ← PLACEHOLDER      │
└────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────┐
│ Client UI Shows Playbook to User                   │
├────────────────────────────────────────────────────┤
│                                                     │
│  Playbook: Weekend RDS Shutdown                    │
│                                                     │
│  Parameters:                                       │
│    ✓ Instance ID:     prod-db-01  (pre-filled)    │
│    ✓ Region:          us-east-1   (pre-filled)    │
│    ⚠ Approval Email:  [_______]   (USER MUST FILL)│
│                                                     │
│  Steps:                                            │
│    1. Validate instance exists                     │
│    2. Create snapshot                              │
│    3. Stop RDS instance                            │
│                                                     │
│  [Cancel]  [Execute Playbook]                      │
│                                                     │
└────────────────────────────────────────────────────┘
    ↓
User fills missing parameter: approval_email = "admin@company.com"
    ↓
User clicks "Execute Playbook"
    ↓
┌────────────────────────────────────────────────────┐
│ Client Execution Engine                            │
├────────────────────────────────────────────────────┤
│ 1. Validates all required parameters filled        │
│ 2. Writes embedded scripts to temp files          │
│ 3. Substitutes parameters into script templates   │
│ 4. Executes scripts locally (bash/python/node)    │
│ 5. Streams progress to user                        │
└────────────────────────────────────────────────────┘
    ↓
Scripts execute on user's machine with user's AWS credentials
```

### Flow 3: User Publishes Playbook

```
User creates playbook locally
    ↓
User clicks "Share with Team"
    ↓
Frontend → Playbook Service (Client):
    publish_metadata("user-weekend-shutdown", "1.2.0")
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Client)                          │
├────────────────────────────────────────────────────┤
│ 1. Read metadata.json from disk                    │
│ 2. Upload scripts to S3                            │
│    PUT → s3://escher-tenant-data/tenants/abc123/... │
│ 3. Call server to publish metadata                 │
│    POST /api/playbook/metadata/publish             │
└────────────────────────────────────────────────────┘
    ↓
Server → Playbook Agent:
    publish_metadata(tenant_id, metadata, s3_paths)
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Agent (Server)                            │
├────────────────────────────────────────────────────┤
│ 1. Ensure tenant collection exists                 │
│    create_collection("tenant_abc123_playbooks")    │
│                                                     │
│ 2. Generate embedding                              │
│    text = metadata.name + description + keywords   │
│    vector = embed(text)                            │
│                                                     │
│ 3. Build payload                                   │
│    payload = {                                     │
│      playbook_id, version, name, description,      │
│      status: "pending_review",                     │
│      storage_strategy: "uploaded_for_review",      │
│      scripts: {shell: s3_path, python: s3_path}    │
│    }                                               │
│                                                     │
│ 4. Upsert to RAG                                   │
│    storage.upsert_point(                           │
│      "tenant_abc123_playbooks",                    │
│      "user-weekend-shutdown-v1.2.0",               │
│      vector,                                       │
│      payload                                       │
│    )                                               │
└────────────────────────────────────────────────────┘
    ↓
Playbook now searchable by all team members
```

---


1. ✅ Define complete playbook structure (metadata, parameters, explain_plan)
2. ✅ Define storage strategies (local_only, uploaded_for_review, uploaded_trusted, using_default)
3. ✅ Design architecture and data flows
4. ⏳ Implement Rust service with storage-service module
5. ⏳ Implement search & ranking logic
6. ⏳ Implement metadata publishing
7. ⏳ Add HTTP API endpoints
8. ⏳ Integration with client Playbook Service
9. ⏳ Testing & validation

---

## Best Practices & Quality Validation

The Playbook Agent enforces best practices and validates playbook quality during publish and ranking.

### Validation During Publish

When a user publishes a playbook, the agent validates it against [best practices](playbook-best-practices.md):

```rust
pub async fn publish_metadata(
    &self,
    tenant_id: &str,
    metadata: &PlaybookMetadata,
    scripts: &HashMap<String, String>,
) -> Result<PublishResult> {
    // Step 1: Validate metadata completeness
    let metadata_validation = validate_metadata_completeness(metadata)?;

    // Step 2: Validate parameters
    let param_validation = validate_parameters(&metadata.parameters)?;

    // Step 3: Validate scripts (if embedded)
    let script_validation = validate_scripts(&metadata.scripts)?;

    // Step 4: Check best practices adherence
    let best_practices_check = check_best_practices(metadata)?;

    // Step 5: Calculate quality score (0-100)
    let quality_score = calculate_quality_score(metadata);

    // Step 6: Determine if playbook meets minimum quality threshold
    if quality_score < 50.0 {
        return Err(Error::QualityThresholdNotMet {
            score: quality_score,
            required: 50.0,
            issues: best_practices_check.issues,
        });
    }

    // Step 7: Publish with quality metadata
    self.publish_with_quality_score(
        tenant_id,
        metadata,
        scripts,
        quality_score,
        best_practices_check,
    ).await
}
```

### Quality Scoring Algorithm

The agent calculates a quality score based on multiple factors:

```rust
fn calculate_quality_score(metadata: &PlaybookMetadata) -> f32 {
    let mut score = 0.0;

    // Metadata completeness (30 points)
    // - Name, description length and clarity
    // - Keywords (5+ required for full points)
    // - Use cases (2+ required)
    // - Prerequisites documented
    score += score_metadata_completeness(metadata) * 30.0;

    // Documentation quality (25 points)
    // - Explain plan completeness
    // - Risk documentation
    // - Rollback strategy
    // - Success criteria
    score += score_documentation_quality(metadata) * 25.0;

    // Parameter design (20 points)
    // - Clear names and descriptions
    // - Validation rules
    // - Auto-fill strategies
    // - Safe defaults
    score += score_parameter_design(metadata) * 20.0;

    // Error handling (15 points)
    // - Pre-flight checks
    // - Rollback strategy
    // - Auto-remediation config
    score += score_error_handling(metadata) * 15.0;

    // Testing coverage (10 points)
    // - Test plan documented
    // - Dry run mode available
    score += score_testing_coverage(metadata) * 10.0;

    score.min(100.0)
}
```

### Quality Score Impact

Quality scores affect playbook ranking and user experience:

| Score Range | Impact | User Experience |
|-------------|--------|-----------------|
| **90-100** | Featured | ⭐ Featured in search results, +20 ranking bonus, "High Quality" badge |
| **70-89** | Normal | Standard ranking, no special indicators |
| **50-69** | Warning | Lower ranking, ⚠️ "Needs Improvement" warning shown |
| **< 50** | Rejected | Cannot publish, must fix issues first |

### Publish Response with Quality Feedback

```json
{
  "status": "published",
  "playbook_id": "user-weekend-shutdown",
  "version": "1.2.0",
  "quality_score": 87,
  "quality_tier": "normal",
  "feedback": {
    "strengths": [
      "Excellent error handling with rollback strategy",
      "Comprehensive explain plan with clear risks and mitigations",
      "Good parameter validation with extraction hint strategies",
      "Idempotent design - safe to retry",
      "Clear documentation and examples"
    ],
    "improvements": [
      "Add 4 more keywords for better searchability (currently 6, recommended 10+)",
      "Include detailed cost savings calculation in estimated_impact",
      "Add test coverage documentation in explain plan"
    ],
    "warnings": [],
    "required_fixes": []
  },
  "best_practices_adherence": {
    "idempotency": true,
    "error_handling": true,
    "documentation": true,
    "security": true,
    "testing": false,
    "performance": true
  }
}
```

### Quality in LLM Ranking

During ranking (STEP 2), the LLM considers quality scores:

```
LLM Prompt:
"Evaluate these playbooks for the user's need:

Playbook 1: user-weekend-shutdown v1.2.0
- Quality Score: 87/100 (Normal)
- Best Practices: ✅ Idempotent, ✅ Error handling, ✅ Clear docs
- Strengths: Comprehensive explain plan, good parameter design
- Areas for improvement: More keywords needed, test coverage docs

Playbook 2: user-old-shutdown v1.0.0
- Quality Score: 62/100 (Needs Improvement)
- Best Practices: ⚠️ No error handling, ⚠️ Minimal documentation
- Strengths: Simple and straightforward
- Areas for improvement: Add error handling, improve documentation, add rollback

Playbook 3: escher-aws-rds-stop v1.2.0
- Quality Score: 98/100 (High Quality - Featured)
- Best Practices: ✅ All best practices followed
- Strengths: Battle-tested (1547 executions, 98% success), excellent docs
- Featured playbook from Escher library

RANKING RULES:
- Consider quality scores as a factor (but not the only factor)
- Higher quality indicates better maintenance and reliability
- Explain quality differences in reasoning
- USER_CUSTOM still takes precedence if quality is acceptable (>50)"
```

### Validation Rules

See [Playbook Best Practices - Validation Rules](playbook-best-practices.md#validation-rules) for detailed validation rules enforced by the agent.

Key validations include:
- **Metadata**: Required fields, format, length constraints
- **Parameters**: Naming conventions, validation rules, extraction hint strategies
- **Scripts**: Error handling, idempotency, output format
- **Documentation**: Explain plan completeness, risk documentation, rollback strategy
- **Security**: No hardcoded credentials, input sanitization, permissions documented

### Complete Best Practices Example

Here's a complete example of a high-quality playbook step that follows all best practices including validation and failure handling:

```json
{
  "playbook_id": "escher-aws-rds-backup-and-upgrade",
  "name": "RDS Database Backup and Upgrade",
  "version": "2.0.0",
  "description": "Creates a verified backup of an RDS instance before performing an in-place upgrade",

  "parameters": [
    {
      "name": "instance_id",
      "type": "string",
      "description": "RDS instance identifier to backup and upgrade",
      "required": true,
      "validation": {
        "pattern": "^[a-zA-Z][a-zA-Z0-9-]{0,62}$",
        "error_message": "Must be valid RDS instance ID"
      }
    },
    {
      "name": "target_engine_version",
      "type": "string",
      "description": "Target database engine version",
      "required": true,
      "validation": {
        "pattern": "^\\d+\\.\\d+(\\.\\d+)?$"
      }
    },
    {
      "name": "backup_retention_days",
      "type": "integer",
      "description": "Number of days to retain the backup",
      "required": false,
      "default": 7,
      "validation": {
        "min": 1,
        "max": 35
      }
    }
  ],

  "orchestration": {
    "steps": [
      {
        "step_number": 1,
        "name": "Verify RDS instance exists and is ready",
        "type": "script",
        "importance": "critical",
        "failure_action": "stop",

        "script_ref": {
          "script_id": "aws-rds-verify-instance",
          "version": "1.0.0"
        },

        "parameter_mapping": {
          "instance_id": "${playbook.instance_id}",
          "region": "${playbook.region}"
        },

        "post_validation": {
          "checks": [
            {
              "type": "output_variables_exist",
              "required_variables": ["instance_state", "engine_version", "storage_encrypted"],
              "description": "Verify instance details are retrieved"
            },
            {
              "type": "instance_state",
              "current_state": "${step1.output.instance_state}",
              "allowed_states": ["available"],
              "description": "Instance must be in available state for backup"
            }
          ],
          "failure_action": "stop"
        },

        "timeout_seconds": 60
      },

      {
        "step_number": 2,
        "name": "Create manual backup snapshot",
        "type": "script",
        "importance": "critical",
        "failure_action": "retry",

        "retry_config": {
          "max_attempts": 3,
          "backoff_seconds": [30, 60, 120],
          "retry_on_exit_codes": [1, 2],
          "retry_on_errors": ["ThrottlingException", "RequestLimitExceeded"]
        },

        "pre_validation": {
          "checks": [
            {
              "type": "parameter_exists",
              "parameters": ["instance_id"],
              "description": "Verify required parameters from previous step"
            },
            {
              "type": "previous_step_output_exists",
              "step_number": 1,
              "required_outputs": ["instance_state", "engine_version"],
              "description": "Need instance details from step 1"
            },
            {
              "type": "instance_state",
              "current_state": "${step1.output.instance_state}",
              "allowed_states": ["available"],
              "description": "Can only snapshot available instances"
            },
            {
              "type": "permission_check",
              "required_permissions": [
                "rds:CreateDBSnapshot",
                "rds:DescribeDBSnapshots",
                "rds:AddTagsToResource"
              ],
              "description": "Verify IAM permissions for snapshot creation"
            }
          ],
          "timeout_seconds": 30,
          "failure_action": "stop"
        },

        "script_ref": {
          "script_id": "aws-rds-create-manual-snapshot",
          "version": "2.1.0"
        },

        "parameter_mapping": {
          "instance_id": "${playbook.instance_id}",
          "snapshot_name": "pre-upgrade-${playbook.instance_id}-${execution.timestamp}",
          "tags": {
            "Purpose": "pre-upgrade-backup",
            "OriginalVersion": "${step1.output.engine_version}",
            "TargetVersion": "${playbook.target_engine_version}",
            "PlaybookExecution": "${execution.id}"
          }
        },

        "post_validation": {
          "checks": [
            {
              "type": "output_variables_exist",
              "required_variables": ["snapshot_id", "snapshot_arn", "status"],
              "description": "Verify snapshot creation outputs"
            },
            {
              "type": "snapshot_state",
              "snapshot_id": "${step2.output.snapshot_id}",
              "expected_states": ["creating", "available"],
              "description": "Verify snapshot is being created",
              "polling": {
                "enabled": true,
                "interval_seconds": 30,
                "max_attempts": 60,
                "success_states": ["available"],
                "failure_states": ["failed", "deleted"],
                "description": "Wait for snapshot to become available (up to 30 minutes)"
              }
            },
            {
              "type": "output_format",
              "variable": "snapshot_id",
              "format": "^arn:aws:rds:[a-z0-9-]+:\\d+:snapshot:.+$|^[a-zA-Z][a-zA-Z0-9-]{0,254}$",
              "description": "Verify snapshot ID or ARN format is valid"
            }
          ],
          "timeout_seconds": 1800,
          "failure_action": "stop"
        },

        "timeout_seconds": 1800
      },

      {
        "step_number": 3,
        "name": "Verify backup integrity",
        "type": "script",
        "importance": "critical",
        "failure_action": "stop",

        "pre_validation": {
          "checks": [
            {
              "type": "previous_step_output_exists",
              "step_number": 2,
              "required_outputs": ["snapshot_id", "status"],
              "description": "Need snapshot details from step 2"
            },
            {
              "type": "state_value_check",
              "variable": "${step2.output.status}",
              "expected_values": ["available"],
              "description": "Snapshot must be available before verification"
            }
          ],
          "failure_action": "stop"
        },

        "script_ref": {
          "script_id": "aws-rds-verify-snapshot-integrity",
          "version": "1.2.0"
        },

        "parameter_mapping": {
          "snapshot_id": "${step2.output.snapshot_id}",
          "expected_size_gb": "${step1.output.allocated_storage}",
          "expected_encrypted": "${step1.output.storage_encrypted}"
        },

        "post_validation": {
          "checks": [
            {
              "type": "output_variables_exist",
              "required_variables": ["integrity_status", "snapshot_size_gb", "encryption_verified"],
              "description": "Verify all integrity checks completed"
            },
            {
              "type": "state_value_check",
              "variable": "${step3.output.integrity_status}",
              "expected_values": ["verified", "passed"],
              "description": "Backup integrity must be verified before proceeding"
            }
          ],
          "failure_action": "stop"
        },

        "timeout_seconds": 300
      },

      {
        "step_number": 4,
        "name": "Perform database engine upgrade",
        "type": "script",
        "importance": "critical",
        "failure_action": "stop",

        "pre_validation": {
          "checks": [
            {
              "type": "previous_step_output_exists",
              "step_number": 3,
              "required_outputs": ["integrity_status"],
              "description": "Backup must be verified before upgrade"
            },
            {
              "type": "state_value_check",
              "variable": "${step3.output.integrity_status}",
              "expected_values": ["verified", "passed"],
              "description": "Cannot proceed without verified backup"
            },
            {
              "type": "permission_check",
              "required_permissions": [
                "rds:ModifyDBInstance",
                "rds:DescribeDBInstances"
              ],
              "description": "Verify permissions for upgrade operation"
            }
          ],
          "timeout_seconds": 30,
          "failure_action": "stop"
        },

        "script_ref": {
          "script_id": "aws-rds-upgrade-engine-version",
          "version": "3.0.0"
        },

        "parameter_mapping": {
          "instance_id": "${playbook.instance_id}",
          "target_version": "${playbook.target_engine_version}",
          "apply_immediately": true,
          "backup_snapshot_id": "${step2.output.snapshot_id}"
        },

        "post_validation": {
          "checks": [
            {
              "type": "output_variables_exist",
              "required_variables": ["upgrade_status", "new_engine_version"],
              "description": "Verify upgrade outputs"
            },
            {
              "type": "instance_state",
              "instance_id": "${playbook.instance_id}",
              "expected_states": ["upgrading", "available"],
              "description": "Verify instance is upgrading or completed",
              "polling": {
                "enabled": true,
                "interval_seconds": 60,
                "max_attempts": 120,
                "success_states": ["available"],
                "failure_states": ["failed", "incompatible-parameters"],
                "description": "Wait for upgrade to complete (up to 2 hours)"
              }
            },
            {
              "type": "state_value_check",
              "variable": "${step4.output.new_engine_version}",
              "expected_values": ["${playbook.target_engine_version}"],
              "description": "Verify upgrade reached target version"
            }
          ],
          "timeout_seconds": 7200,
          "failure_action": "stop"
        },

        "timeout_seconds": 7200
      },

      {
        "step_number": 5,
        "name": "Tag backup snapshot with retention policy",
        "type": "script",
        "importance": "low",
        "failure_action": "ignore",

        "script_ref": {
          "script_id": "aws-rds-tag-snapshot-retention",
          "version": "1.0.0"
        },

        "parameter_mapping": {
          "snapshot_id": "${step2.output.snapshot_id}",
          "retention_days": "${playbook.backup_retention_days}",
          "tags": {
            "UpgradeStatus": "completed",
            "NewVersion": "${step4.output.new_engine_version}",
            "RetentionDays": "${playbook.backup_retention_days}"
          }
        },

        "post_validation": {
          "checks": [
            {
              "type": "output_variables_exist",
              "required_variables": ["tags_applied"],
              "description": "Verify tagging completed"
            }
          ],
          "failure_action": "ignore"
        },

        "timeout_seconds": 60
      }
    ]
  },

  "explain_plan": {
    "steps": [
      {
        "step": 1,
        "action": "Verify RDS instance exists and retrieve current state",
        "reasoning": "Ensures instance exists and is in a state that allows backup creation",
        "expected_outcome": "Instance details retrieved, state confirmed as 'available'",
        "risk": "low",
        "reversible": true
      },
      {
        "step": 2,
        "action": "Create manual backup snapshot of the database",
        "reasoning": "Critical safety measure - ensures we can restore if upgrade fails",
        "expected_outcome": "Snapshot created and verified as 'available'",
        "risk": "low",
        "reversible": true,
        "notes": "Includes retry logic for transient AWS API failures. Polls until snapshot is fully available."
      },
      {
        "step": 3,
        "action": "Verify backup integrity and completeness",
        "reasoning": "Confirms backup is valid and can be used for restore if needed",
        "expected_outcome": "Backup integrity verified, size and encryption match source",
        "risk": "low",
        "reversible": true
      },
      {
        "step": 4,
        "action": "Perform in-place database engine upgrade",
        "reasoning": "Upgrades database to target version with verified backup available",
        "expected_outcome": "Database upgraded to target version, instance available",
        "risk": "medium",
        "reversible": true,
        "notes": "Instance will be unavailable during upgrade (typically 10-60 minutes). Can rollback using snapshot from step 2."
      },
      {
        "step": 5,
        "action": "Tag snapshot with retention policy",
        "reasoning": "Ensures backup is retained per policy and can be identified later",
        "expected_outcome": "Snapshot tagged with retention metadata",
        "risk": "low",
        "reversible": true,
        "notes": "Non-critical operation - marked as 'ignore' so failure won't block playbook"
      }
    ],
    "overall_risk": "medium",
    "estimated_duration": "30-90 minutes (depends on database size)",
    "prerequisites": [
      "AWS credentials with RDS permissions",
      "Instance must be in 'available' state",
      "Target version must be compatible with current engine version",
      "Sufficient storage quota for snapshot creation"
    ],
    "rollback_strategy": "If upgrade fails, restore from snapshot created in step 2 using AWS RDS restore-from-snapshot operation"
  },

  "success_criteria": [
    "RDS instance upgraded to target engine version",
    "Instance returns to 'available' state",
    "Verified backup snapshot exists and is valid",
    "All data integrity checks pass"
  ]
}
```

**Why This Example Demonstrates Best Practices:**

1. **Comprehensive Validation**
   - Pre-validation checks prerequisites before each critical operation
   - Post-validation confirms operations actually succeeded
   - Validation checks include permissions, state, outputs, and data integrity

2. **Smart Failure Handling**
   - Critical steps (1-4) use `failure_action: "stop"` to prevent proceeding with invalid state
   - Step 2 uses `retry` with exponential backoff for transient AWS API errors
   - Step 5 uses `ignore` since tagging is non-critical

3. **Importance Levels**
   - Steps 1-4 marked as `"critical"` - upgrade cannot succeed without them
   - Step 5 marked as `"low"` - nice to have but not essential

4. **Polling for Eventual Consistency**
   - Step 2 polls snapshot until available (up to 30 minutes)
   - Step 4 polls instance during upgrade (up to 2 hours)
   - Proper success/failure state detection

5. **Clear Documentation**
   - Comprehensive explain plan with reasoning for each step
   - Risk assessment and rollback strategy documented
   - Prerequisites and success criteria clearly defined

6. **Parameter Design**
   - Required vs optional parameters clearly marked
   - Validation rules with helpful error messages
   - Safe defaults provided (backup_retention_days: 7)

7. **Security & Safety**
   - Permission checks before critical operations
   - Verified backup before destructive upgrade
   - Proper tagging for auditability and retention

This playbook would receive a quality score of **95+/100** and be featured in search results.

---

## Related Documentation

- **[Playbook Best Practices](playbook-best-practices.md)** - Complete guide for writing high-quality playbooks
- [Playbook Service (Client)](../../04-services/playbook-service/README.md) - Client-side playbook management
- [Storage Service](../../04-services/storage-service/README.md) - RAG module used by agent
- [Storage Service Collections](../../04-services/storage-service/collections.md) - user_playbooks schema
---

## Appendix: Implementation Details

This appendix contains technical implementation details, code examples, and API specifications for developers building or extending the Playbook Agent.

## Implementation

### Rust Service Structure

```rust
use storage_service::{StorageService, StorageConfig, QdrantMode};
use std::sync::Arc;

pub struct PlaybookAgent {
    storage: Arc<StorageService>,
    config: PlaybookAgentConfig,
}

pub struct PlaybookAgentConfig {
    pub qdrant_url: String,
    pub embedding_service_url: String,
}

impl PlaybookAgent {
    pub async fn new(config: PlaybookAgentConfig) -> Result<Self> {
        // Initialize storage service in remote mode
        let storage_config = StorageConfig {
            qdrant: QdrantConfig {
                mode: QdrantMode::Remote,
                storage_path: None,
                url: Some(config.qdrant_url.clone()),
            },
            embedding: EmbeddingConfig {
                provider: EmbeddingProvider::Server {
                    endpoint: config.embedding_service_url.clone(),
                    api_key: None,
                },
                model: "text-embedding-3-small".to_string(),
                dimension: 384,
            },
            encryption: EncryptionConfig {
                enabled: false,  // Server: metadata only, no encryption
            },
            ..Default::default()
        };

        let storage = Arc::new(StorageService::new(storage_config).await?);

        Ok(Self { storage, config })
    }
}
```

### Key Methods (to be implemented)

```rust
impl PlaybookAgent {
    // Search across escher_library + tenant playbooks
    pub async fn search_playbooks(
        &self,
        query: &str,
        tenant_id: &str,
        filters: Option<PlaybookFilters>,
        limit: usize,
    ) -> Result<Vec<RankedPlaybook>>;

    // Publish user playbook metadata to tenant collection
    pub async fn publish_metadata(
        &self,
        tenant_id: &str,
        metadata: &PlaybookMetadata,
        scripts: &HashMap<String, String>,  // format -> S3 path
    ) -> Result<String>;

    // Update playbook status (pending_review → approved)
    pub async fn update_status(
        &self,
        tenant_id: &str,
        playbook_id: &str,
        version: &str,
        new_status: &str,
    ) -> Result<()>;

    // Delete playbook metadata
    pub async fn delete_metadata(
        &self,
        tenant_id: &str,
        playbook_id: &str,
        version: &str,
    ) -> Result<()>;

    // Update execution statistics
    pub async fn update_execution_stats(
        &self,
        tenant_id: &str,
        playbook_id: &str,
        version: &str,
        success: bool,
    ) -> Result<()>;
}
```

---

## API

### HTTP Endpoints (Axum)

```rust
use axum::{Router, routing::{get, post, delete}, Json};

pub fn router() -> Router {
    Router::new()
        .route("/api/playbook/search", post(search_playbooks))
        .route("/api/playbook/metadata/publish", post(publish_metadata))
        .route("/api/playbook/metadata/:tenant_id/:playbook_id/:version/status",
               post(update_status))
        .route("/api/playbook/metadata/:tenant_id/:playbook_id/:version",
               delete(delete_metadata))
        .route("/api/playbook/stats/:tenant_id/:playbook_id/:version",
               post(update_stats))
}
```

---

## Next Steps

---

## Appendix: Why LLM + RAG Works Better Than Just Search

### Traditional Keyword Search ❌
```
Query: "Stop production RDS for weekend"
→ Matches: playbooks containing words "stop", "rds", "production"
→ Problem: Misses semantic meaning (cost optimization, time-based)
→ Problem: Can't explain why one is better than another
```

### Vector Search Only ⚠️
```
Query: "Stop production RDS for weekend"
→ Embedding similarity finds related playbooks
→ Better: Understands "shutdown" = "stop", "weekend" relates to "cost-saving"
→ Problem: Still can't reason about which is BEST for user's context
```

### LLM + RAG ✅
```
Query: "Stop production RDS for weekend"
→ LLM understands: cost optimization, weekend automation, production safety
→ RAG finds: candidates with semantic similarity
→ LLM evaluates: success rates, prerequisites, user's environment
→ LLM explains: "Use your custom playbook because it has environment-specific logic..."
→ User gets: Ranked options with clear reasoning
```

---

