# Playbook Agent - Complete Design (Working Document)

**Status**: 🔄 Active Brainstorming
**Date**: October 16, 2025
**Purpose**: Complete redesign of Playbook Agent architecture covering all scenarios

---

## Table of Contents

1. [Overview](#overview)
2. [Playbook Types](#playbook-types)
3. [Storage Architecture](#storage-architecture)
4. [RAG Structure](#rag-structure)
5. [Complete Flow: Success](#complete-flow-success)
6. [Complete Flow: Failure & Remediation](#complete-flow-failure--remediation)
7. [Server Response JSON](#server-response-json)
8. [Open Questions](#open-questions)

---

## Overview

**Playbook Agent** is a server-side intelligent agent that:
1. Searches playbooks using LLM + RAG
2. Returns fully expanded execution plans with all sub-steps
3. Pre-fills parameters from user prompts
4. Provides smart summaries after successful execution
5. Auto-suggests fixes when execution fails

**Key Principles**:
- **Normalized Storage**: Scripts are separate reusable entities (Foreign Key pattern)
- **Multi-Implementation**: Each script can support multiple languages (bash, python, node, terraform, cloudformation)
- **User Choice**: Agent returns available implementations, user selects at execution time
- **Fully Expanded**: Client receives complete command tree upfront, no abstraction

---

## Complete Architecture Summary

### Storage Model (Normalized Database Pattern)

```
┌─────────────────────────────────────────────────────────────┐
│                    SCRIPT LIBRARY                           │
│             (Reusable, Multi-Language)                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  scripts/{script_id}/v1.0.0/                               │
│  ├── metadata.json  (available_types: [bash, python, ...]) │
│  ├── parameters.json                                        │
│  └── code/                                                  │
│      ├── bash/       ← User selects implementation          │
│      ├── python/                                            │
│      ├── node/                                              │
│      ├── terraform/                                         │
│      └── cloudformation/                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                            ↑
                            │ (Foreign Key Reference)
                            │
┌─────────────────────────────────────────────────────────────┐
│                    PLAYBOOK LIBRARY                         │
│              (Orchestration, References Only)               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  playbooks/{playbook_id}/v1.0.0/                           │
│  ├── metadata.json                                          │
│  ├── parameters.json                                        │
│  ├── orchestration.json  ← References script_ids           │
│  └── explain_plan.json   (optional)                        │
│                                                             │
│  orchestration.json contains:                               │
│  {                                                          │
│    "steps": [                                               │
│      {                                                      │
│        "type": "script",                                    │
│        "script_ref": {                                      │
│          "script_id": "create-rds-snapshot",                │
│          "version": "1.0.0"                                 │
│        },                                                   │
│        "implementation_choice": "user_selects"              │
│      }                                                      │
│    ]                                                        │
│  }                                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Server Response Flow

```
User Query → Playbook Agent
    ↓
1. LLM Intent Extraction
    ↓
2. RAG Search (playbooks only)
    ↓
3. Recursive Resolution:
   For each step in playbook:
     - If type="playbook": Fetch and expand recursively
     - If type="script": Fetch script metadata
    ↓
4. Script Implementation Discovery:
   For each script reference:
     - Fetch script metadata.json
     - Read available_types: ["bash", "python", "node"]
     - Read entry_points for each type
    ↓
5. Return to Client:
   {
     "playbook": {...},
     "steps": [
       {
         "sub_steps": [
           {
             "script_ref": "create-rds-snapshot",
             "available_implementations": [
               {
                 "type": "bash",
                 "entry_point": "s3://.../code/bash/script.sh",
                 "description": "Fast, no dependencies"
               },
               {
                 "type": "python",
                 "entry_point": "s3://.../code/python/main.py",
                 "description": "Better error handling"
               }
             ],
             "recommended": "bash"
           }
         ]
       }
     ]
   }
    ↓
6. Client Displays:
   "This step can run as: [Bash] [Python] [Node]"
    ↓
7. User Selects Implementation
    ↓
8. Client Downloads & Executes
```

### Benefits of This Architecture

1. **Script Reusability**: One script (e.g., "create-rds-snapshot") can be used by multiple playbooks
2. **Independent Versioning**: Update script implementation without touching playbooks
3. **User Flexibility**: Same playbook, different languages based on user preference/environment
4. **Clean Storage**: No duplication, foreign key pattern like normalized database
5. **Easy Maintenance**: Fix bug in one script → all playbooks using it benefit
6. **Multi-Cloud Ready**: Same logical operation, different implementation per cloud provider

---

## Playbook Types

### Type 1: Pure Script Playbook

**Example**: "stop-rds-instance"

```
stop-rds-instance/
└── Just runs AWS CLI commands
    ├── Check instance state
    ├── Send stop command
    └── Wait for stopped
```

**Characteristics**:
- Has actual script files (shell/python/terraform)
- No sub-playbook calls
- Self-contained execution

---

### Type 2: Pure Orchestration Playbook

**Example**: "stop-rds-with-snapshot"

```
stop-rds-with-snapshot/
├── Step 1: Call "create-rds-snapshot" playbook
├── Step 2: Call "stop-rds-instance" playbook
└── Step 3: Call "send-notification" playbook
```

**Characteristics**:
- NO script files of its own
- Only orchestrates other playbooks
- Pure coordination logic

---

### Type 3: Hybrid Playbook

**Example**: "backup-and-stop-rds"

```
backup-and-stop-rds/
├── Step 1: Call "create-rds-snapshot" playbook
├── Step 2: Run custom verification script
└── Step 3: Call "stop-rds-instance" playbook
```

**Characteristics**:
- Mix of playbook calls + inline scripts
- Most flexible type

---

## Storage Architecture

### Normalized Structure (Foreign Key Pattern)

**Key Principle**: Scripts are separate reusable entities, playbooks reference them by ID

---

### Script Library (Separate Storage)

```
s3://escher-tenant-data/tenants/{id}/scripts/
├── create-rds-snapshot/
│   └── v1.0.0/
│       ├── metadata.json           # Name, description, available_types
│       ├── parameters.json         # Input parameters
│       └── code/
│           ├── bash/
│           │   └── script.sh
│           ├── python/
│           │   ├── main.py
│           │   └── requirements.txt
│           ├── node/
│           │   ├── index.js
│           │   └── package.json
│           ├── terraform/
│           │   ├── main.tf
│           │   ├── variables.tf
│           │   └── outputs.tf
│           └── cloudformation/
│               └── template.yaml
│
├── stop-rds-instance/
│   └── v1.0.0/
│       ├── metadata.json
│       ├── parameters.json
│       └── code/
│           ├── bash/
│           │   └── stop.sh
│           └── python/
│               └── stop.py
│
└── verify-backup/
    └── v1.0.0/
        ├── metadata.json
        ├── parameters.json
        └── code/
            ├── bash/
            │   └── verify.sh
            └── python/
                └── verify.py
```

**Script metadata.json**:
```json
{
  "script_id": "create-rds-snapshot",
  "version": "1.0.0",
  "name": "Create RDS Snapshot",
  "description": "Creates a snapshot of an RDS instance",
  "available_types": ["bash", "python", "node"],
  "default_type": "bash",
  "entry_points": {
    "bash": "code/bash/script.sh",
    "python": "code/python/main.py",
    "node": "code/node/index.js"
  },
  "tags": ["rds", "backup", "snapshot"],
  "author": "admin@company.com",
  "created_at": "2024-01-15T10:00:00Z"
}
```

---

### Playbook Storage (References Only)

#### Type 1: Pure Script Playbook
```
s3://escher-tenant-data/tenants/{id}/playbooks/stop-rds-instance/v1.0.0/
├── metadata.json              # Name, description, keywords, status
├── parameters.json            # Parent-level parameters
├── orchestration.json         # References script IDs (Foreign key pattern)
└── explain_plan.json          # (optional) Pre-written plan
```

**orchestration.json for Pure Script Playbook**:
```json
{
  "steps": [
    {
      "step_number": 1,
      "type": "script",
      "title": "Check Instance State",
      "script_ref": {
        "script_id": "check-rds-state",
        "version": "1.0.0",
        "source": "tenant_scripts"
      },
      "parameter_mapping": {
        "instance_id": "${parent.instance_id}",
        "region": "${parent.region}"
      },
      "implementation_choice": "user_selects",  // or "auto_default"
      "blocking": true,
      "on_failure": "abort"
    },
    {
      "step_number": 2,
      "type": "script",
      "title": "Stop RDS Instance",
      "script_ref": {
        "script_id": "stop-rds-instance",
        "version": "1.0.0",
        "source": "tenant_scripts"
      },
      "parameter_mapping": {
        "instance_id": "${parent.instance_id}",
        "region": "${parent.region}"
      },
      "implementation_choice": "user_selects",
      "blocking": true,
      "on_failure": "abort"
    }
  ]
}
```

#### Type 2: Pure Orchestration Playbook
```
s3://escher-tenant-data/tenants/{id}/playbooks/stop-rds-with-snapshot/v1.0.0/
├── metadata.json              # Name, description, keywords, status
├── parameters.json            # Parent-level parameters
├── orchestration.json         # Step definitions with playbook references
└── explain_plan.json          # (optional) High-level overview
```

**orchestration.json example**:
```json
{
  "steps": [
    {
      "step_number": 1,
      "type": "playbook",
      "title": "Create Backup Snapshot",
      "playbook_ref": {
        "playbook_id": "create-rds-snapshot",
        "version": "1.0.0",
        "source": "escher_library"
      },
      "parameter_mapping": {
        "instance_id": "${parent.instance_id}",
        "snapshot_name": "${parent.snapshot_name}"
      },
      "blocking": true,
      "on_failure": "abort"
    },
    {
      "step_number": 2,
      "type": "playbook",
      "title": "Stop RDS Instance",
      "playbook_ref": {
        "playbook_id": "stop-rds-instance",
        "version": "2.0.0",
        "source": "escher_library"
      },
      "parameter_mapping": {
        "instance_id": "${parent.instance_id}"
      },
      "blocking": true,
      "on_failure": "abort"
    }
  ]
}
```

#### Hybrid Playbook
```
s3://escher-tenant-data/tenants/{id}/playbooks/backup-and-stop-rds/v1.0.0/
├── metadata.json
├── parameters.json
├── orchestration.json         # Mix of playbook refs + script refs
├── scripts/
│   └── verify.sh             # Inline script for step 2
└── explain_plan.json
```

**orchestration.json for hybrid**:
```json
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
      "title": "Verify Snapshot",
      "script": {
        "format": "shell",
        "path": "scripts/verify.sh"
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

---

## RAG Structure

### What's Stored in RAG (Qdrant)

**All playbooks** (script, orchestration, hybrid) store the same metadata structure:

```json
{
  "id": "stop-rds-with-snapshot-v1.0.0",
  "vector": [0.123, -0.456, ...],  // 384D embedding
  "payload": {
    // IDENTIFICATION
    "playbook_id": "stop-rds-with-snapshot",
    "version": "1.0.0",
    "type": "orchestration",  // "script", "orchestration", or "hybrid"

    // METADATA
    "name": "Stop RDS with Backup",
    "description": "Creates backup snapshot then stops RDS instance",
    "keywords": ["rds", "stop", "snapshot", "backup", "cost-saving"],
    "cloud_provider": ["aws"],
    "resource_types": ["rds::instance", "db-snapshot"],

    // LIFECYCLE
    "status": "active",
    "author": "tenant_custom",
    "created_at": 1729123456,
    "updated_at": 1729123456,

    // EXECUTION STATS
    "execution_count": 47,
    "success_count": 47,
    "success_rate": 1.0,
    "avg_duration_seconds": 300,

    // STORAGE REFERENCES
    "storage_strategy": "uploaded_trusted",
    "s3_base_path": "s3://escher-tenant-data/tenants/abc123/playbooks/stop-rds-with-snapshot/v1.0.0/",

    // DEPENDENCIES (for orchestration/hybrid playbooks)
    "depends_on": [
      {"playbook_id": "create-rds-snapshot", "version": "1.0.0"},
      {"playbook_id": "stop-rds-instance", "version": "2.0.0"}
    ],

    // FILE REFERENCES
    "files": {
      "metadata": "s3://.../metadata.json",
      "parameters": "s3://.../parameters.json",
      "orchestration": "s3://.../orchestration.json",  // Only for orchestration/hybrid
      "scripts": {  // Only for script/hybrid
        "shell": "s3://.../shell/stop_rds.sh"
      }
    }
  }
}
```

### Search Strategy

**For nested playbooks**:
- Parent playbook keywords include its own + flattened from children
- Example: "stop-rds-with-snapshot" has keywords: ["stop", "rds", "snapshot", "backup"] from itself + ["quota", "verify", "integrity"] from child "create-rds-snapshot"
- This ensures search finds parent when user searches for child functionality

---

## Complete Flow: Success

### 1. User Query

```
User: "Stop my production RDS db-prod-01"
```

### 2. Server Processing

#### Step 2.1: LLM Intent Understanding

```
LLM analyzes prompt → Extracts:
{
  "action": "stop",
  "cloud_provider": "aws",
  "resource_types": ["rds::instance"],
  "extracted_parameters": {
    "instance_id": "db-prod-01",
    "environment": "production"
  }
}
```

#### Step 2.2: RAG Search

```rust
// Search both collections
let query = "stop rds production snapshot backup";
let vector = embedder.embed(query);

// Search escher_library
let escher_results = qdrant.search(
    "escher_library",
    vector,
    Filter::must([
        Filter::match("cloud_provider", "aws"),
        Filter::match("status", "active")
    ])
);

// Search tenant playbooks
let tenant_results = qdrant.search(
    "tenant_abc123_playbooks",
    vector,
    Filter::must([
        Filter::match("cloud_provider", "aws"),
        Filter::match("status", "active")
    ])
);

// Merge results
let candidates = merge(escher_results, tenant_results);
```

**Results**:
```
1. stop-rds-with-snapshot (tenant, orchestration, score: 0.92)
2. stop-rds-instance (escher, script, score: 0.85)
3. stop-rds-weekend-automation (escher, hybrid, score: 0.78)
```

#### Step 2.3: Fetch Playbook Details

Server downloads from S3:
```
For "stop-rds-with-snapshot":
1. Download metadata.json
2. Download parameters.json
3. Download orchestration.json
4. For each step in orchestration:
   - If type="playbook": Download that playbook recursively
   - If type="script": Note script path
```

**Recursive resolution**:
```
stop-rds-with-snapshot/
├── Step 1: create-rds-snapshot (playbook)
│   ├── Sub-step 1.1: check-quota (script)
│   ├── Sub-step 1.2: create-snapshot (script)
│   ├── Sub-step 1.3: wait-completion (script)
│   └── Sub-step 1.4: verify-snapshot (script)
└── Step 2: stop-rds-instance (playbook)
    ├── Sub-step 2.1: check-connections (script)
    ├── Sub-step 2.2: send-stop-command (script)
    └── Sub-step 2.3: wait-stopped (script)
```

#### Step 2.4: LLM Parameter Filling

```
LLM prompt:
"User query: 'Stop my production RDS db-prod-01'
Parameters needed: {instance_id, snapshot_name, region}
Extract and generate values."

LLM output:
{
  "instance_id": "db-prod-01",  // Extracted from prompt
  "snapshot_name": "snap-db-prod-01-2025-10-16-14-30",  // Generated
  "region": "us-east-1"  // From user's estate context
}
```

#### Step 2.5: Fill Scripts with Parameters

For each script in the fully expanded tree:
```
Template: aws rds create-db-snapshot --db-instance-identifier ${instance_id} --db-snapshot-identifier ${snapshot_name}

Filled: aws rds create-db-snapshot --db-instance-identifier db-prod-01 --db-snapshot-identifier snap-db-prod-01-2025-10-16-14-30
```

#### Step 2.6: Generate/Load Explain Plan

**If explain_plan.json exists in S3**:
- Load and fill with parameters

**If NOT exists (dynamic playbook)**:
- LLM generates explain plan from orchestration + scripts

### 3. Server Response (Fully Expanded)

```json
{
  "query": "Stop my production RDS db-prod-01",

  "intent": {
    "action": "stop",
    "cloud_provider": "aws",
    "resource_types": ["rds::instance"],
    "extracted_parameters": {
      "instance_id": "db-prod-01",
      "environment": "production"
    }
  },

  "ranked_playbooks": [
    {
      "rank": 1,
      "playbook_id": "stop-rds-with-snapshot",
      "version": "1.0.0",
      "type": "orchestration",
      "name": "Stop RDS with Backup",
      "status": "active",
      "source": "tenant_custom",
      "confidence": 0.92,
      "reason": "Your team's custom playbook with 100% success rate...",

      "parameters": {
        "instance_id": {
          "value": "db-prod-01",
          "source": "extracted_from_prompt",
          "editable": true
        },
        "snapshot_name": {
          "value": "snap-db-prod-01-2025-10-16-14-30",
          "source": "auto_generated",
          "editable": true
        },
        "region": {
          "value": "us-east-1",
          "source": "user_context",
          "editable": false
        }
      },

      "explain_plan": {
        "overview": "Creates backup snapshot then stops database db-prod-01.",
        "total_steps": 2,
        "total_duration_estimate": "~5 minutes",

        "steps": [
          {
            "step_number": 1,
            "step_type": "playbook",
            "title": "Create Backup Snapshot",
            "description": "Creates snapshot using create-rds-snapshot playbook",
            "estimated_duration": "~3 minutes",

            "playbook_reference": {
              "playbook_id": "create-rds-snapshot",
              "version": "1.0.0",
              "status": "active",
              "source": "escher_library"
            },

            "sub_steps": [
              {
                "sub_step_number": "1.1",
                "title": "Check Snapshot Quota",
                "description": "Verifies available quota",
                "commands": [
                  "aws rds describe-db-snapshots --region us-east-1 --query 'length(DBSnapshots)'"
                ],
                "estimated_duration": "5 seconds",
                "must_succeed": true
              },
              {
                "sub_step_number": "1.2",
                "title": "Create RDS Snapshot",
                "description": "Initiates snapshot creation",
                "commands": [
                  "aws rds create-db-snapshot --db-instance-identifier db-prod-01 --db-snapshot-identifier snap-db-prod-01-2025-10-16-14-30 --region us-east-1"
                ],
                "estimated_duration": "5 seconds",
                "must_succeed": true
              },
              {
                "sub_step_number": "1.3",
                "title": "Wait for Snapshot Completion",
                "description": "Polls AWS until snapshot is ready",
                "commands": [
                  "aws rds wait db-snapshot-available --db-snapshot-identifier snap-db-prod-01-2025-10-16-14-30 --region us-east-1"
                ],
                "estimated_duration": "~2 minutes 30 seconds",
                "must_succeed": true
              },
              {
                "sub_step_number": "1.4",
                "title": "Verify Snapshot Integrity",
                "description": "Checks snapshot status and size",
                "commands": [
                  "aws rds describe-db-snapshots --db-snapshot-identifier snap-db-prod-01-2025-10-16-14-30 --query 'DBSnapshots[0].[Status,AllocatedStorage]'"
                ],
                "estimated_duration": "5 seconds",
                "must_succeed": true
              }
            ],

            "blocking": true,
            "on_failure": "abort"
          },

          {
            "step_number": 2,
            "step_type": "playbook",
            "title": "Stop RDS Instance",
            "description": "Stops database db-prod-01",
            "estimated_duration": "~2 minutes",

            "playbook_reference": {
              "playbook_id": "stop-rds-instance",
              "version": "2.0.0",
              "status": "active",
              "source": "escher_library"
            },

            "sub_steps": [
              {
                "sub_step_number": "2.1",
                "title": "Check Active Connections",
                "description": "Warns if connections exist",
                "commands": [
                  "aws rds describe-db-instances --db-instance-identifier db-prod-01 --query 'DBInstances[0].DBInstanceStatus'"
                ],
                "estimated_duration": "3 seconds",
                "must_succeed": false
              },
              {
                "sub_step_number": "2.2",
                "title": "Send Stop Command",
                "description": "Stops the database",
                "commands": [
                  "aws rds stop-db-instance --db-instance-identifier db-prod-01 --region us-east-1"
                ],
                "estimated_duration": "5 seconds",
                "must_succeed": true
              },
              {
                "sub_step_number": "2.3",
                "title": "Wait for Instance to Stop",
                "description": "Polls until stopped",
                "commands": [
                  "aws rds wait db-instance-stopped --db-instance-identifier db-prod-01 --region us-east-1"
                ],
                "estimated_duration": "~1 minute 50 seconds",
                "must_succeed": true
              }
            ],

            "blocking": true,
            "on_failure": "abort"
          }
        ]
      }
    }
  ]
}
```

### 4. Client Display (Fully Expanded)

```
┌─────────────────────────────────────────────────────────────┐
│ Stop RDS with Backup                                        │
│ Rank 1 - 92% confidence                                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ This will:                                                  │
│                                                             │
│ Step 1: Create Backup Snapshot (~3 min)                    │
│   Uses: create-rds-snapshot v1.0.0 (Escher Library)        │
│                                                             │
│   1.1. Check snapshot quota (5s)                           │
│        aws rds describe-db-snapshots --region us-east-1... │
│                                                             │
│   1.2. Create RDS snapshot (5s)                            │
│        aws rds create-db-snapshot                           │
│          --db-instance-identifier db-prod-01                │
│          --db-snapshot-identifier snap-db-prod-01-...       │
│          --region us-east-1                                 │
│                                                             │
│   1.3. Wait for completion (~2m 30s)                       │
│        aws rds wait db-snapshot-available...                │
│                                                             │
│   1.4. Verify snapshot integrity (5s)                      │
│        aws rds describe-db-snapshots...                     │
│                                                             │
│   ⚠️ All sub-steps must complete before continuing          │
│                                                             │
│ Step 2: Stop RDS Instance (~2 min)                         │
│   Uses: stop-rds-instance v2.0.0 (Escher Library)          │
│                                                             │
│   2.1. Check active connections (3s - optional)            │
│        aws rds describe-db-instances...                     │
│                                                             │
│   2.2. Send stop command (5s)                              │
│        aws rds stop-db-instance                             │
│          --db-instance-identifier db-prod-01                │
│          --region us-east-1                                 │
│                                                             │
│   2.3. Wait for instance to stop (~1m 50s)                 │
│        aws rds wait db-instance-stopped...                  │
│                                                             │
│   ⚠️ All sub-steps must complete                            │
│                                                             │
│ Total time: ~5 minutes                                      │
│                                                             │
│ Parameters (editable):                                      │
│   instance_id: db-prod-01                                   │
│   snapshot_name: snap-db-prod-01-2025-10-16-14-30           │
│   region: us-east-1 (read-only)                             │
│                                                             │
│ [Edit Parameters] [Execute] [Cancel]                       │
└─────────────────────────────────────────────────────────────┘
```

### 5. Client Execution

```
User clicks "Execute"
    ↓
Client downloads all scripts from S3 (if needed)
Client executes locally (Tauri Rust)
    ↓
Step 1: Create Snapshot
  Sub-step 1.1: Check quota... ✅ (5s)
  Sub-step 1.2: Create snapshot... ✅ (4s)
  Sub-step 1.3: Wait completion... ✅ (2m 35s)
  Sub-step 1.4: Verify integrity... ✅ (5s)
Step 1 COMPLETE ✅
    ↓
Step 2: Stop Instance
  Sub-step 2.1: Check connections... ⚠️ (3s - warning logged)
  Sub-step 2.2: Send stop command... ✅ (5s)
  Sub-step 2.3: Wait stopped... ✅ (1m 52s)
Step 2 COMPLETE ✅
    ↓
PLAYBOOK COMPLETE ✅
```

### 6. Success Reporting

**Client → Server**:
```json
{
  "execution_id": "exec-abc123",
  "playbook_id": "stop-rds-with-snapshot",
  "version": "1.0.0",
  "status": "completed",
  "total_duration_seconds": 320,

  "steps_executed": [
    {
      "step_number": 1,
      "title": "Create Backup Snapshot",
      "duration_seconds": 159,
      "sub_steps": [
        {"sub_step": "1.1", "status": "completed", "duration": 5},
        {"sub_step": "1.2", "status": "completed", "duration": 4},
        {"sub_step": "1.3", "status": "completed", "duration": 155},
        {"sub_step": "1.4", "status": "completed", "duration": 5}
      ],
      "output": "Snapshot snap-db-prod-01-2025-10-16-14-30 created. Size: 47.3 GB"
    },
    {
      "step_number": 2,
      "title": "Stop RDS Instance",
      "duration_seconds": 160,
      "sub_steps": [
        {"sub_step": "2.1", "status": "warning", "duration": 3},
        {"sub_step": "2.2", "status": "completed", "duration": 5},
        {"sub_step": "2.3", "status": "completed", "duration": 112}
      ],
      "output": "Instance db-prod-01 stopped"
    }
  ]
}
```

**Server LLM → Client**:
```json
{
  "execution_id": "exec-abc123",
  "summary": "Successfully stopped db-prod-01 after creating backup",

  "key_results": [
    "✅ Snapshot created: snap-db-prod-01-2025-10-16-14-30 (47.3 GB)",
    "✅ Database stopped: db-prod-01",
    "💰 Saving ~$120/month while stopped"
  ],

  "next_actions": [
    {
      "suggestion": "Set reminder to restart Monday morning",
      "playbook_available": "schedule-rds-restart"
    }
  ],

  "insights": {
    "performance": "Completed 20 seconds faster than estimate",
    "pattern": "5th time running this - consider automation"
  }
}
```

### 7. Client Shows Summary

```
┌─────────────────────────────────────────────────────────────┐
│ ✅ Success!                                                 │
│                                                             │
│ Successfully stopped db-prod-01                             │
│                                                             │
│ What happened:                                              │
│ • Snapshot created: snap-db-prod-01-... (47.3 GB)          │
│ • Database stopped: db-prod-01                              │
│ • Completed in 5 min 20 sec (20s faster than expected!)    │
│                                                             │
│ Cost Impact:                                                │
│ 💰 Saving ~$120/month while stopped                        │
│                                                             │
│ Smart Suggestions:                                          │
│ □ Set reminder to restart Monday morning                   │
│   [Schedule Restart]                                        │
│                                                             │
│ 💡 You've run this 5 times. Want to automate?              │
│   [Set Up Schedule]                                         │
│                                                             │
│ [Done]                                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Complete Flow: Failure & Remediation

### 1-4. Same as Success Flow

User queries → Server searches → Returns fully expanded playbook → Client displays

### 5. Execution Fails

```
Client executes:
Step 1: Create Snapshot
  Sub-step 1.1: Check quota... ✅ (5s) - 95/100 used
  Sub-step 1.2: Create snapshot... ❌ FAILED

ERROR: SnapshotQuotaExceeded
"You have reached the maximum number of manual DB snapshots (100)"
```

### 6. Client Reports Failure

**Client → Server**:
```json
{
  "execution_id": "exec-abc123",
  "playbook_id": "stop-rds-with-snapshot",
  "version": "1.0.0",
  "status": "failed",
  "failed_at_step": 1,
  "failed_at_sub_step": "1.2",

  "error": {
    "step_title": "Create Backup Snapshot",
    "sub_step_title": "Create RDS Snapshot",
    "code": "SnapshotQuotaExceeded",
    "message": "You have reached the maximum number of manual DB snapshots allowed (100)",
    "command_failed": "aws rds create-db-snapshot --db-instance-identifier db-prod-01 --db-snapshot-identifier snap-db-prod-01-2025-10-16-14-30",
    "raw_output": "An error occurred (SnapshotQuotaExceeded) when calling the CreateDBSnapshot operation..."
  },

  "completed_sub_steps": [
    {"step": 1, "sub_step": "1.1", "status": "completed"}
  ]
}
```

### 7. Server Auto-Remediation (IMMEDIATE)

**Server LLM analyzes**:
```
Error: SnapshotQuotaExceeded
Context: Failed at step 1.2 (Create snapshot)
User intent: Stop production database safely

Search for ALL fix playbooks:
- delete-old-snapshots
- request-quota-increase
- stop-without-snapshot (risky)
```

**Server → Client**:
```json
{
  "execution_id": "exec-abc123",
  "error_analysis": "Snapshot quota exceeded (100/100 used). Need to free space or increase limit.",

  "remediation_playbooks": [
    {
      "rank": 1,
      "playbook_id": "delete-old-snapshots",
      "version": "1.0.0",
      "type": "script",
      "name": "Clean Up Old Snapshots",
      "reason": "Quick fix - frees quota in 2 minutes",

      "parameters": {
        "region": {
          "value": "us-east-1",
          "source": "parent_context",
          "editable": false
        },
        "older_than_days": {
          "value": 90,
          "source": "default",
          "editable": true
        }
      },

      "explain_plan": {
        "overview": "Lists snapshots older than 90 days and lets you delete them.",
        "steps": [
          {
            "step_number": 1,
            "title": "List Old Snapshots",
            "commands": [
              "aws rds describe-db-snapshots --region us-east-1 --query 'DBSnapshots[?SnapshotCreateTime<=`2025-07-16`]'"
            ]
          },
          {
            "step_number": 2,
            "title": "Delete Selected Snapshots",
            "commands": [
              "aws rds delete-db-snapshot --db-snapshot-identifier {snapshot-id}"
            ]
          }
        ]
      }
    },

    {
      "rank": 2,
      "playbook_id": "request-quota-increase",
      "version": "1.0.0",
      "type": "script",
      "name": "Request Snapshot Quota Increase",
      "reason": "Permanent fix but takes 1-2 hours for AWS approval",

      "explain_plan": {
        "overview": "Opens AWS support case to increase quota from 100 to 200.",
        "steps": [
          {
            "step_number": 1,
            "title": "Create Support Case",
            "commands": [
              "aws support create-case --subject 'Increase RDS Snapshot Quota' --service-code 'amazon-rds' --severity-code 'low' --category-code 'general-guidance'"
            ]
          }
        ]
      }
    },

    {
      "rank": 3,
      "playbook_id": "stop-rds-without-snapshot",
      "version": "1.0.0",
      "type": "script",
      "name": "Stop Without Snapshot (Risky)",
      "reason": "Continue without backup - NOT recommended for production",
      "risk_level": "high",

      "explain_plan": {
        "overview": "Stops database without creating snapshot. ⚠️ No recovery point!",
        "steps": [
          {
            "step_number": 1,
            "title": "Stop RDS Instance",
            "commands": [
              "aws rds stop-db-instance --db-instance-identifier db-prod-01"
            ]
          }
        ]
      }
    }
  ],

  "resume_hint": {
    "can_resume": true,
    "message": "After running any fix, you can resume from Step 1, Sub-step 1.2",
    "resume_action": {
      "type": "resume_from_sub_step",
      "playbook_id": "stop-rds-with-snapshot",
      "version": "1.0.0",
      "from_step": 1,
      "from_sub_step": "1.2"
    }
  }
}
```

### 8. Client Shows Remediation

```
┌─────────────────────────────────────────────────────────────┐
│ ❌ Execution Failed                                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Failed at Step 1.2: Create RDS Snapshot                    │
│                                                             │
│ Error: Snapshot quota exceeded (100/100 used)              │
│                                                             │
│ ────────────────────────────────────────────────────────── │
│                                                             │
│ Fix Options:                                                │
│                                                             │
│ □ Clean Up Old Snapshots (~2 min) ⭐ Recommended           │
│   Deletes snapshots older than 90 days                     │
│   [View Full Details]                                       │
│                                                             │
│ □ Request Quota Increase (~1-2 hours)                      │
│   Opens AWS support case                                   │
│   [View Full Details]                                       │
│                                                             │
│ □ Stop Without Snapshot (⚠️ Risky)                         │
│   Continues without backup - NOT for production            │
│   [View Full Details]                                       │
│                                                             │
│ [Run Selected Fix]                                          │
│                                                             │
│ ────────────────────────────────────────────────────────── │
│                                                             │
│ After fix completes:                                        │
│ [Resume from Step 1.2] [Run Different Playbook]           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 9. User Runs Fix Playbook

```
User selects "Clean Up Old Snapshots"
    ↓
Client shows full expand plan (same format as main playbook)
    ↓
User clicks Execute
    ↓
Client executes fix playbook
    ↓
Step 1: List old snapshots... ✅
Found 8 snapshots older than 90 days
    ↓
Step 2: Delete selected snapshots... ✅
Deleted: snap-2025-01-15, snap-2025-02-01, ...
    ↓
Fix playbook COMPLETE ✅
```

### 10. User Resumes Original

```
┌─────────────────────────────────────────────────────────────┐
│ ✅ Fix Complete!                                            │
│                                                             │
│ Deleted 8 old snapshots                                     │
│ Quota now: 92/100                                           │
│                                                             │
│ Ready to resume original playbook:                          │
│ "Stop RDS with Backup" from Step 1, Sub-step 1.2           │
│                                                             │
│ [Resume Now] [Not Now]                                     │
└─────────────────────────────────────────────────────────────┘

User clicks "Resume Now"
    ↓
Client re-executes from Step 1, Sub-step 1.2
    ↓
Sub-step 1.2: Create snapshot... ✅ (now succeeds!)
Sub-step 1.3: Wait completion... ✅
Sub-step 1.4: Verify integrity... ✅
Step 1 COMPLETE ✅
    ↓
Step 2: Stop Instance...
All sub-steps complete ✅
    ↓
PLAYBOOK COMPLETE ✅
    ↓
Shows success summary
```

---

## Server Response JSON

### Structure Overview

```typescript
interface ServerSearchResponse {
  query: string;
  intent: ExtractedIntent;
  ranked_playbooks: RankedPlaybook[];
}

interface RankedPlaybook {
  // Metadata
  rank: number;
  playbook_id: string;
  version: string;
  type: "script" | "orchestration" | "hybrid";
  name: string;
  status: PlaybookStatus;
  source: "escher_library" | "tenant_custom";
  confidence: number;
  reason: string;

  // Parameters (pre-filled)
  parameters: Record<string, ParameterValue>;

  // Fully expanded explain plan
  explain_plan: ExplainPlan;
}

interface ExplainPlan {
  overview: string;
  total_steps: number;
  total_duration_estimate: string;

  steps: ExecutionStep[];
}

interface ExecutionStep {
  step_number: number;
  step_type: "script" | "playbook";
  title: string;
  description: string;
  estimated_duration: string;

  // If step_type = "playbook"
  playbook_reference?: PlaybookReference;

  // Fully expanded sub-steps
  sub_steps: SubStep[];

  // Execution control
  blocking: boolean;
  on_failure: "abort" | "continue";
}

interface SubStep {
  sub_step_number: string;  // "1.1", "1.2", etc.
  title: string;
  description: string;
  commands: string[];
  estimated_duration: string;
  must_succeed: boolean;
}
```

---

## Open Questions

### 1. Orchestration Storage
**Question**: Store orchestration.json in S3 or RAG payload?
**Current proposal**: S3 (keeps RAG lightweight)

### 2. Explain Plan Generation
**Question**: Pre-written or dynamically generated from sub-playbooks?
**Current proposal**: Hybrid - pre-written if exists, else generate

### 3. Version Pinning
**Question**: Exact version, latest, or semver range?
**Current proposal**: Exact version for stability

### 4. Circular Dependencies
**Question**: How to prevent?
**Current proposal**: Detect at upload time + max depth limit (5 levels)

### 5. Parameter Passing
**Question**: Explicit mapping or auto-match?
**Current proposal**: Explicit mapping in orchestration.json

### 6. RAG Keywords
**Question**: Store parent+child keywords separately or flatten?
**Current proposal**: Flatten (better search)

### 7. Resume Granularity
**Question**: Resume from step or sub-step?
**Current proposal**: Sub-step level for precision

### 8. Success Reporting Frequency
**Question**: Report per step or only final?
**Current proposal**: Only final (no noise during execution)

### 9. Sub-Playbook Depth Limit
**Question**: How many levels deep can orchestration go?
**Current proposal**: 5 levels max to prevent performance issues

### 10. Script Download Timing
**Question**: Pre-fetch all scripts or download on-demand?
**Current proposal**: Download when user clicks Execute (saves bandwidth)

---

## Next Steps

1. ✅ Finalize storage structure (orchestration.json format)
2. ✅ Finalize RAG payload structure
3. ✅ Finalize server response JSON
4. ⏳ Implement recursive playbook resolution algorithm
5. ⏳ Implement parameter mapping/filling logic
6. ⏳ Implement explain plan generation (LLM-based)
7. ⏳ Write complete new playbook-agent.md

---

**End of Working Document**

This document captures the complete design including all scenarios. Once finalized, this will replace the existing playbook-agent.md with a comprehensive, consistent architecture.