# Playbook Agent

**Role**: LLM + RAG intelligent playbook discovery, ranking, and orchestration engine
**Type**: Server-side Rust service using storage-service crate + LLM integration (Claude/GPT)
**Location**: Runs on Escher server (ECS/EC2 with EBS/EFS volume)

---

## Table of Contents

1. [What It Does](#what-it-does)
2. [Core Architecture Principles](#core-architecture-principles)
3. [Three Playbook Types](#three-playbook-types)
4. [How It Works: The Intelligence Flow](#how-it-works-the-intelligence-flow)
5. [Playbook Lifecycle & States](#playbook-lifecycle-states)
6. [Playbook Structure](#playbook-structure)
7. [Nested Sub-Steps in Explain Plan](#nested-sub-steps-in-explain-plan)
8. [Auto-Remediation Flow](#auto-remediation-flow)
9. [Resume Capability](#resume-capability)
10. [Storage Strategies](#storage-strategies)
11. [S3 Storage Structure](#s3-storage-structure)
12. [Architecture](#architecture)
13. [Data Flow](#data-flow)
14. [API](#api)
15. [Related Documentation](#related-documentation)

---

## What It Does

The Playbook Agent is an **intelligent search and recommendation engine** powered by LLM + RAG. It:

1. **Understands intent** using LLM (Claude/GPT) - not just keywords
2. **Finds candidates** using RAG vector search across Escher + Tenant playbooks
3. **Ranks intelligently** using LLM reasoning + precedence rules
4. **Returns complete playbooks** with explain plan, code, and reasoning

**Key Insight**: It's not just search - it's an AI agent that understands what you're trying to do and recommends the best automation.

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
User Query
    ↓
┌─────────────────────────────────────────────────────┐
│ STEP 1: LLM Intent Understanding                    │
│  LLM analyzes query to extract intent               │
├─────────────────────────────────────────────────────┤
│ STEP 2: RAG Vector Search                           │
│  Find candidate playbooks using embeddings          │
├─────────────────────────────────────────────────────┤
│ STEP 3: LLM Ranking & Reasoning                     │
│  LLM evaluates each candidate with context          │
├─────────────────────────────────────────────────────┤
│ STEP 4: Package & Return                            │
│  Send playbooks with explain plan + code            │
└─────────────────────────────────────────────────────┘
    ↓
Client receives ranked playbooks ready to execute
```

---

### STEP 1: LLM Intent Understanding

**Purpose**: Understand WHAT the user wants to do and WHY

**Input**: Raw user query
```
"Stop production RDS for weekend"
```

**LLM Prompt**:
```
Analyze this cloud operations request and extract structured intent:

Query: "Stop production RDS for weekend"

Extract:
1. Primary action (stop, start, scale, backup, etc.)
2. Cloud provider (aws, azure, gcp)
3. Resource types (ec2, rds, s3, etc.)
4. Environment/filters (production, staging, tags)
5. Use case category (cost-optimization, maintenance, disaster-recovery)
6. Time context (weekend, scheduled, immediate)

Return JSON:
{
  "action": "stop",
  "cloud_provider": "aws",
  "resource_types": ["rds::instance"],
  "filters": {"environment": "production"},
  "use_case": "cost-optimization",
  "time_based": true,
  "keywords": ["weekend", "cost-saving", "automated-shutdown"]
}
```

**LLM Output** (Claude/GPT):
```json
{
  "action": "stop",
  "cloud_provider": "aws",
  "resource_types": ["rds::instance"],
  "filters": {"environment": "production"},
  "use_case": "cost-optimization",
  "time_based": true,
  "keywords": ["weekend", "cost-saving", "automated-shutdown", "snapshot"]
}
```

**Why LLM?**:
- Understands synonyms: "turn off" = "stop" = "shutdown"
- Infers context: "weekend" → cost optimization use case
- Extracts implicit requirements: production → needs snapshot before stop

---

### STEP 2: RAG Vector Search

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

### STEP 3: LLM Ranking & Reasoning

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

### STEP 4: Package & Return to Client

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
          "auto_fill_strategy": {
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
- ✅ Parameters with auto-fill hints
- ✅ LLM-generated reasoning
- ✅ Confidence scores

**Client's Next Steps**:
1. User reviews explain plans
2. Selects preferred playbook (usually rank 1)
3. Reviews/fills parameters (auto-fill helps)
4. Execution Engine runs the script

---

## Why LLM + RAG Works Better Than Just Search

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
          └──────────┘
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

## Playbook Structure

### Complete Playbook Format

A playbook consists of:
1. **metadata.json** - Core metadata and configuration
2. **orchestration.json** - Step definitions with script/playbook references (NEW)
3. **parameters.json** - Input parameters with auto-fill strategies
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

```json
{
  "parameters": [
    {
      "name": "instance_id",
      "type": "string",
      "description": "RDS instance identifier",
      "required": true,
      "default": null,
      "validation": {
        "pattern": "^[a-zA-Z][a-zA-Z0-9-]{0,62}$",
        "min_length": 1,
        "max_length": 63
      },
      "auto_fill_strategy": {
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
      "required": false,
      "default": "auto-snapshot-{timestamp}",
      "auto_fill_strategy": {
        "source": "generated",
        "template": "weekend-shutdown-{instance_id}-{timestamp}"
      }
    },
    {
      "name": "skip_snapshot",
      "type": "boolean",
      "description": "Skip creating snapshot before shutdown",
      "required": false,
      "default": false,
      "validation": {
        "warning_if_true": "Skipping snapshot is risky for production databases"
      }
    },
    {
      "name": "region",
      "type": "string",
      "description": "AWS region",
      "required": true,
      "auto_fill_strategy": {
        "source": "context",
        "context_field": "resource.region"
      }
    },
    {
      "name": "dry_run",
      "type": "boolean",
      "description": "Simulate without making changes",
      "required": false,
      "default": false
    }
  ]
}
```

#### Auto-Fill Strategy Types

| Source | Description | Example |
|--------|-------------|---------|
| `user_estate` | Query user's cloud resources | Find RDS instances matching filters |
| `context` | Extract from execution context | Get region from selected resource |
| `generated` | Generate using template | Create timestamp-based names |
| `static` | Use fixed value | Default region "us-west-2" |
| `prompt` | Ask user (no auto-fill) | Custom notes or confirmation |

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

### 5. Complete Playbook Structure on Disk

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

## Auto-Remediation Flow (NEW)

**Key Concept**: When a step fails during playbook execution, the system can automatically attempt to fix the issue and retry the failed step.

### How Auto-Remediation Works

```
Playbook Execution
    ↓
Step 2 Fails (e.g., "Create RDS snapshot")
    ↓
┌────────────────────────────────────────────────────┐
│ Execution Engine Detects Failure                   │
├────────────────────────────────────────────────────┤
│ Error: "InsufficientStorageException:              │
│  Not enough disk space to create snapshot"         │
└────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────┐
│ Auto-Remediation Agent (LLM-Powered)               │
├────────────────────────────────────────────────────┤
│ 1. Analyze error message                           │
│ 2. Check playbook context                          │
│ 3. Determine if auto-fixable                       │
│ 4. Generate remediation plan                       │
└────────────────────────────────────────────────────┘
    ↓
LLM Determines: "Fixable - Need to free disk space"
    ↓
┌────────────────────────────────────────────────────┐
│ Remediation Steps                                   │
├────────────────────────────────────────────────────┤
│ 1. Delete old snapshots (retention policy)         │
│ 2. Wait 30 seconds for space reclamation           │
│ 3. Retry Step 2                                    │
└────────────────────────────────────────────────────┘
    ↓
Step 2 Retried → Success ✓
    ↓
Continue to Step 3
```

### Remediation Configuration in orchestration.json

```json
{
  "steps": [
    {
      "step_number": 2,
      "name": "Create RDS snapshot",
      "type": "script",
      "script_ref": {
        "script_id": "create-rds-snapshot",
        "version": "2.1.0",
        "implementation": "python"
      },
      "auto_remediation": {
        "enabled": true,
        "max_attempts": 3,
        "retry_delay_seconds": 30,
        "remediation_strategies": [
          {
            "error_pattern": "InsufficientStorageException",
            "action": "cleanup_old_snapshots",
            "parameters": {
              "retention_days": 7
            }
          },
          {
            "error_pattern": "SnapshotQuotaExceededException",
            "action": "delete_oldest_snapshot",
            "parameters": {
              "count": 1
            }
          },
          {
            "error_pattern": "ThrottlingException",
            "action": "exponential_backoff",
            "parameters": {
              "initial_delay": 5,
              "max_delay": 60
            }
          }
        ],
        "fallback": "notify_user"
      },
      "parameter_mapping": {
        "instance_id": "${playbook.instance_id}",
        "snapshot_name": "${playbook.snapshot_name}"
      }
    }
  ]
}
```

### Auto-Remediation Process

**1. Error Detection**:
```rust
match step_result {
    Err(error) => {
        if step.auto_remediation.enabled {
            attempt_auto_remediation(step, error).await?;
        } else {
            return Err(error); // Fail immediately
        }
    }
    Ok(output) => continue_to_next_step(output),
}
```

**2. LLM Analysis** (Optional - for unknown errors):
```
LLM Prompt:
"Analyze this error and suggest remediation if possible:

Error: InsufficientStorageException - Not enough disk space to create snapshot
Context: Creating RDS snapshot for instance prod-db-1
Step: create-rds-snapshot (step 2 of 5)

Can this be automatically fixed? If yes, provide:
1. Remediation action
2. Expected success rate
3. Risk level (low/medium/high)

Return JSON."
```

**LLM Response**:
```json
{
  "auto_fixable": true,
  "remediation_action": "cleanup_old_snapshots",
  "parameters": {
    "retention_days": 7,
    "delete_count": 3
  },
  "success_probability": 0.85,
  "risk_level": "low",
  "explanation": "Delete snapshots older than 7 days to free space. Low risk since snapshots are redundant backups."
}
```

**3. Execute Remediation**:
```rust
async fn execute_remediation(action: &str, params: &Value) -> Result<()> {
    match action {
        "cleanup_old_snapshots" => {
            let retention_days = params["retention_days"].as_i64().unwrap();
            delete_old_snapshots(retention_days).await?;
            tokio::time::sleep(Duration::from_secs(30)).await; // Wait for cleanup
            Ok(())
        },
        "delete_oldest_snapshot" => {
            let count = params["count"].as_i64().unwrap();
            delete_oldest_snapshots(count).await?;
            Ok(())
        },
        "exponential_backoff" => {
            let initial_delay = params["initial_delay"].as_i64().unwrap();
            tokio::time::sleep(Duration::from_secs(initial_delay)).await;
            Ok(())
        },
        _ => Err(Error::UnknownRemediation(action.to_string())),
    }
}
```

**4. Retry Failed Step**:
```rust
for attempt in 1..=max_attempts {
    match execute_step(step).await {
        Ok(output) => return Ok(output), // Success!
        Err(error) => {
            if attempt < max_attempts {
                // Try remediation
                if let Some(remediation) = find_remediation(&error, &step) {
                    execute_remediation(&remediation.action, &remediation.params).await?;
                    continue; // Retry
                }
            }
            return Err(error); // Give up
        }
    }
}
```

### Common Remediation Strategies

| Error Type | Remediation Action | Success Rate |
|------------|-------------------|--------------|
| `InsufficientStorageException` | Delete old snapshots | 85% |
| `SnapshotQuotaExceededException` | Delete oldest snapshot | 90% |
| `ThrottlingException` | Exponential backoff | 95% |
| `InvalidParameterException` | LLM suggests parameter fix | 60% |
| `ResourceNotFoundException` | Create missing resource | 70% |
| `AccessDeniedException` | Notify user (not auto-fixable) | 0% |

### UI Display During Remediation

```
Weekend RDS Automation
├─ Step 1: Create RDS snapshot ✓ (35s)
├─ Step 2: Stop RDS instance ⚠️ FAILED → REMEDIATING
│  └─ Error: InsufficientStorageException
│  └─ Remediation: Deleting old snapshots (retention: 7 days)
│  └─ Deleted 3 old snapshots (freed 15 GB)
│  └─ Retrying step 2... ⏳
│  └─ Retry successful! ✓
└─ Step 3: Send notification ⏳ (waiting...)
```

### Remediation Logs

All remediation attempts are logged for auditing:

```json
{
  "execution_id": "exec-abc123",
  "playbook_id": "weekend-rds-automation",
  "version": "1.5.0",
  "step_number": 2,
  "remediation_log": [
    {
      "attempt": 1,
      "timestamp": "2025-10-16T10:15:30Z",
      "error": "InsufficientStorageException: Not enough disk space",
      "remediation_action": "cleanup_old_snapshots",
      "remediation_params": {
        "retention_days": 7,
        "deleted_count": 3,
        "space_freed_gb": 15
      },
      "remediation_duration_seconds": 45,
      "retry_result": "success"
    }
  ],
  "final_status": "success",
  "total_attempts": 2
}
```

### Benefits

- **Reliability**: 70-90% of transient failures auto-fixed
- **User Experience**: No manual intervention required
- **Transparency**: Full audit trail of remediation attempts
- **Intelligence**: LLM can handle novel errors
- **Safety**: Max attempts limit prevents infinite loops

---

## Resume Capability (NEW)

**Key Concept**: If a playbook execution fails mid-way, users can resume from the last successful step instead of starting over.

### How Resume Works

```
Playbook Execution Started
├─ Step 1: Create RDS snapshot ✓ (completed)
├─ Step 2: Verify backup integrity ✓ (completed)
├─ Step 3: Stop RDS instance ❌ (failed)
└─ Step 4: Send notification ⏸️ (not started)

┌────────────────────────────────────────────────────┐
│ Execution State Saved                               │
├────────────────────────────────────────────────────┤
│ - Last successful step: 2                           │
│ - Step outputs: {                                   │
│     step1: {snapshot_id: "snap-123"},              │
│     step2: {verification_status: "passed"}         │
│   }                                                 │
│ - Failed step: 3                                    │
│ - Error: "ActiveConnectionsException"              │
└────────────────────────────────────────────────────┘

User Reviews Error → Fixes Issue (closes connections)

User Clicks "Resume Playbook"
    ↓
┌────────────────────────────────────────────────────┐
│ Resume from Step 3                                  │
├────────────────────────────────────────────────────┤
│ - Skip steps 1 & 2 (already completed)             │
│ - Load outputs from steps 1 & 2                    │
│ - Retry step 3 with saved parameters               │
│ - Continue to step 4 if successful                 │
└────────────────────────────────────────────────────┘
    ↓
├─ Step 3: Stop RDS instance ✓ (retried & succeeded)
└─ Step 4: Send notification ✓ (completed)

Playbook Completed Successfully!
```

### Execution State Storage

**State File Format** (`~/.escher/execution-state/{execution_id}.json`):

```json
{
  "execution_id": "exec-abc123",
  "playbook_id": "weekend-rds-automation",
  "version": "1.5.0",
  "tenant_id": "abc123",
  "user_id": "user@example.com",

  "started_at": "2025-10-16T10:00:00Z",
  "paused_at": "2025-10-16T10:05:30Z",
  "status": "paused",

  "parameters": {
    "instance_id": "prod-db-1",
    "snapshot_name": "weekend-backup-20251016",
    "region": "us-east-1"
  },

  "completed_steps": [
    {
      "step_number": 1,
      "name": "Create RDS snapshot",
      "status": "completed",
      "started_at": "2025-10-16T10:00:00Z",
      "completed_at": "2025-10-16T10:00:35Z",
      "duration_seconds": 35,
      "output": {
        "snapshot_id": "snap-abc123",
        "snapshot_arn": "arn:aws:rds:us-east-1:123456789:snapshot:snap-abc123"
      }
    },
    {
      "step_number": 2,
      "name": "Verify backup integrity",
      "status": "completed",
      "started_at": "2025-10-16T10:00:35Z",
      "completed_at": "2025-10-16T10:05:25Z",
      "duration_seconds": 290,
      "output": {
        "verification_status": "passed",
        "integrity_score": 1.0
      }
    }
  ],

  "failed_step": {
    "step_number": 3,
    "name": "Stop RDS instance",
    "status": "failed",
    "started_at": "2025-10-16T10:05:25Z",
    "failed_at": "2025-10-16T10:05:30Z",
    "duration_seconds": 5,
    "error": {
      "type": "ActiveConnectionsException",
      "message": "Cannot stop instance with 5 active connections",
      "code": "RDS-1003"
    },
    "retry_count": 0
  },

  "pending_steps": [4],

  "resume_from_step": 3,
  "resumable": true
}
```

### Resume API

**Client-Side API**:

```rust
// Check if execution can be resumed
pub async fn can_resume(execution_id: &str) -> Result<bool> {
    let state = load_execution_state(execution_id).await?;
    Ok(state.status == "paused" && state.resumable)
}

// Resume execution
pub async fn resume_execution(execution_id: &str) -> Result<ExecutionResult> {
    // Load saved state
    let mut state = load_execution_state(execution_id).await?;

    // Load playbook
    let playbook = load_playbook(&state.playbook_id, &state.version).await?;

    // Skip completed steps, start from failed/next step
    let start_from = state.resume_from_step;

    // Execute remaining steps
    for step_num in start_from..=playbook.steps.len() {
        let step = &playbook.steps[step_num - 1];

        // Use outputs from previous steps
        let context = build_execution_context(&state.completed_steps);

        match execute_step(step, &context).await {
            Ok(output) => {
                // Save progress
                state.completed_steps.push(StepResult {
                    step_number: step_num,
                    status: "completed",
                    output,
                    ..Default::default()
                });
                save_execution_state(&state).await?;
            },
            Err(error) => {
                // Save failure and pause
                state.failed_step = Some(StepResult {
                    step_number: step_num,
                    status: "failed",
                    error: Some(error.clone()),
                    ..Default::default()
                });
                state.status = "paused";
                save_execution_state(&state).await?;
                return Err(error);
            }
        }
    }

    // All steps completed
    state.status = "completed";
    state.completed_at = Some(SystemTime::now());
    save_execution_state(&state).await?;

    Ok(ExecutionResult::Success)
}
```

### UI for Resume

**Execution History View**:

```
┌─────────────────────────────────────────────────────┐
│ Paused Executions                                   │
├─────────────────────────────────────────────────────┤
│ 🔶 Weekend RDS Automation v1.5.0                    │
│    Started: 2025-10-16 10:00 AM                     │
│    Paused:  2025-10-16 10:05 AM (5 minutes ago)     │
│    Status:  Failed at Step 3                        │
│    Error:   ActiveConnectionsException              │
│                                                      │
│    Progress: ▓▓▓▓▓▓░░░░ 2/4 steps (50%)             │
│                                                      │
│    [Resume] [View Details] [Delete]                 │
├─────────────────────────────────────────────────────┤
│ 🔶 Backup Production Databases v2.0.0               │
│    Started: 2025-10-15 11:30 PM                     │
│    Paused:  2025-10-15 11:45 PM (12 hours ago)      │
│    Status:  Failed at Step 5                        │
│                                                      │
│    [Resume] [View Details] [Delete]                 │
└─────────────────────────────────────────────────────┘
```

**Resume Confirmation Dialog**:

```
┌─────────────────────────────────────────────────────┐
│ Resume Playbook Execution?                          │
├─────────────────────────────────────────────────────┤
│                                                      │
│ Playbook: Weekend RDS Automation v1.5.0             │
│ Execution ID: exec-abc123                           │
│                                                      │
│ Completed Steps:                                    │
│ ✓ Step 1: Create RDS snapshot                       │
│ ✓ Step 2: Verify backup integrity                   │
│                                                      │
│ Will Resume From:                                   │
│ ↻ Step 3: Stop RDS instance                         │
│                                                      │
│ Remaining Steps:                                    │
│ ⏸️ Step 4: Send notification                         │
│                                                      │
│ Previous Error:                                     │
│ ActiveConnectionsException: Cannot stop instance    │
│ with 5 active connections                           │
│                                                      │
│ ⚠️ Make sure the issue has been resolved before     │
│    resuming.                                        │
│                                                      │
│          [Cancel]  [Resume Execution]               │
└─────────────────────────────────────────────────────┘
```

### Resume Limitations

**Cannot Resume If**:
- Playbook has been modified (version changed)
- Completed steps have expired state (> 24 hours)
- External resources changed (e.g., snapshot deleted)
- Step dependencies broken

**Validation Before Resume**:

```rust
pub fn validate_resume_state(state: &ExecutionState) -> Result<()> {
    // Check state age
    let state_age = SystemTime::now()
        .duration_since(state.paused_at)?
        .as_secs();
    if state_age > 86400 {  // 24 hours
        return Err(Error::StateExpired);
    }

    // Check playbook version still exists
    let playbook = load_playbook(&state.playbook_id, &state.version).await?;

    // Verify outputs from completed steps are still valid
    for step in &state.completed_steps {
        if let Some(snapshot_id) = step.output.get("snapshot_id") {
            if !verify_snapshot_exists(snapshot_id).await? {
                return Err(Error::ResourceNoLongerExists);
            }
        }
    }

    Ok(())
}
```

### Benefits

- **Time Saving**: Don't repeat long-running steps
- **Cost Saving**: Avoid redundant cloud operations
- **Debugging**: Fix issues and continue
- **Flexibility**: Pause and resume later
- **Recovery**: Handle transient failures gracefully

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

### File Download Flow

#### Client ← S3 Download

```
User selects playbook to execute
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Client)                          │
├────────────────────────────────────────────────────┤
│ 1. Check if script exists locally                  │
│    → NOT FOUND                                     │
│                                                     │
│ 2. Request download URL from server                │
│    GET /api/script/download?                       │
│        playbook_id=user-weekend-shutdown&          │
│        version=1.2.0&                              │
│        format=shell&                               │
│        tenant_id=abc123                            │
└────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Agent (Server)                            │
├────────────────────────────────────────────────────┤
│ 1. Lookup playbook metadata from RAG               │
│    storage.get_point(                              │
│      "tenant_abc123_playbooks",                    │
│      "user-weekend-shutdown-v1.2.0"                │
│    )                                               │
│    → Get S3 path from payload                      │
│                                                     │
│ 2. Generate pre-signed download URL                │
│    s3_client.generate_presigned_url(               │
│      'get_object',                                 │
│      Bucket='escher-tenant-data',                  │
│      Key='tenants/abc123/playbooks/.../main.sh',   │
│      ExpiresIn=3600  # 1 hour                      │
│    )                                               │
│                                                     │
│ 3. Return download URL + checksum                  │
│    {                                               │
│      "url": "https://s3...?signature",             │
│      "checksum": "sha256:abc123...",               │
│      "size_bytes": 4096,                           │
│      "expires_at": 1728394762                      │
│    }                                               │
└────────────────────────────────────────────────────┘
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Client)                          │
├────────────────────────────────────────────────────┤
│ 1. Download from S3                                │
│    GET https://s3...?signature                     │
│                                                     │
│ 2. Verify checksum                                 │
│    sha256(downloaded_file) == expected_checksum    │
│                                                     │
│ 3. Cache locally                                   │
│    ~/.escher/cache/scripts/                        │
│      user-weekend-shutdown-v1.2.0/shell/main.sh    │
│                                                     │
│ 4. Return local path to execution engine           │
└────────────────────────────────────────────────────┘
```

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

### Flow 2: Client Resolves Playbook Scripts

```
User selects: "user-weekend-shutdown-v1.2.0"
    ↓
Frontend → Playbook Service (Rust, client-side):
    resolve_script("user-weekend-shutdown", "1.2.0", "shell")
    ↓
┌────────────────────────────────────────────────────┐
│ Playbook Service (Client)                          │
├──────────────────────────��─────────────────────────┤
│ 1. Check local storage                             │
│    path = ~/.escher/playbooks/synced/              │
│           user-weekend-shutdown/v1.2.0/shell/      │
│    → FOUND! Return local path                      │
│                                                     │
│ (If NOT found locally)                             │
│ 2. Check cache                                     │
│    path = ~/.escher/cache/scripts/                 │
│           user-weekend-shutdown-v1.2.0/            │
│    → If found & fresh → Return cached path         │
│                                                     │
│ 3. Download from S3                                │
│    GET /api/script/download?                       │
│        s3_path=s3://escher-tenant-data/...         │
│    → Server returns pre-signed URL                 │
│    → Download & cache                              │
│    → Return cached path                            │
└────────────────────────────────────────────────────┘
    ↓
Execution Engine executes script at resolved path
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

## Related Documentation

- [Playbook Service (Client)](../../04-services/playbook-service/README.md) - Client-side playbook management
- [Storage Service](../../04-services/storage-service/README.md) - RAG module used by agent
- [Storage Service Collections](../../04-services/storage-service/collections.md) - user_playbooks schema