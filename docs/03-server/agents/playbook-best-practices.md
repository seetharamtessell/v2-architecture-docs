# Playbook Best Practices

**Purpose**: Guidelines for writing high-quality, maintainable, and reliable playbooks

**Audience**: DevOps engineers, SREs, and automation developers creating custom playbooks

**Enforced By**: Playbook Agent validation during publish + LLM evaluation during search/ranking

---

## Table of Contents

1. [Core Principles](#core-principles)
2. [Metadata Best Practices](#metadata-best-practices)
3. [Parameter Design](#parameter-design)
4. [Script Quality Standards](#script-quality-standards)
5. [Error Handling](#error-handling)
6. [Testing Requirements](#testing-requirements)
7. [Documentation Standards](#documentation-standards)
8. [Security Guidelines](#security-guidelines)
9. [Performance Optimization](#performance-optimization)
10. [Validation Rules](#validation-rules)
11. [Agent Integration](#agent-integration)

---

## Core Principles

### 1. Single Responsibility

**DO**: Each playbook should do ONE thing well
```json
{
  "playbook_id": "stop-rds-instance",
  "name": "Stop RDS Instance",
  "description": "Safely stops an RDS instance with pre-flight checks"
}
```

**DON'T**: Mix multiple unrelated operations
```json
{
  "playbook_id": "stop-rds-and-backup-s3",
  "name": "Stop RDS and Backup S3",
  "description": "Stops RDS, backs up S3, and sends Slack notification"
}
```

**Why**: Single responsibility makes playbooks:
- Easier to understand and maintain
- More reusable across different workflows
- Simpler to test and debug
- Better for composition (use Pure Orchestration playbooks to combine)

---

### 2. Idempotency

**DO**: Playbooks should be safe to run multiple times
```bash
#!/bin/bash
# Check if RDS instance is already stopped
INSTANCE_STATE=$(aws rds describe-db-instances \
  --db-instance-identifier "$INSTANCE_ID" \
  --query 'DBInstances[0].DBInstanceStatus' \
  --output text)

if [ "$INSTANCE_STATE" = "stopped" ]; then
  echo "Instance already stopped"
  exit 0
fi

# Stop instance
aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"
```

**DON'T**: Fail if already in desired state
```bash
#!/bin/bash
# This will fail if instance is already stopped
aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"
```

**Why**: Idempotency ensures:
- Safe retries after failures
- Predictable behavior
- No unintended side effects
- Resume capability works correctly

---

### 3. Fail Fast with Clear Messages

**DO**: Validate inputs and prerequisites early
```bash
#!/bin/bash
set -e

# Validate inputs
if [ -z "$INSTANCE_ID" ]; then
  echo "ERROR: INSTANCE_ID is required"
  exit 1
fi

# Check prerequisites
if ! command -v aws &> /dev/null; then
  echo "ERROR: AWS CLI not installed"
  exit 1
fi

# Verify instance exists
if ! aws rds describe-db-instances --db-instance-identifier "$INSTANCE_ID" &> /dev/null; then
  echo "ERROR: RDS instance '$INSTANCE_ID' not found"
  exit 1
fi

# Proceed with operation
aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"
```

**Why**: Early validation:
- Saves time and resources
- Provides clear error messages
- Prevents partial execution
- Easier to debug

---

### 4. Comprehensive Logging

**DO**: Log all important actions and decisions
```bash
#!/bin/bash
echo "[INFO] Starting RDS instance stop process"
echo "[INFO] Instance ID: $INSTANCE_ID"
echo "[INFO] Region: $REGION"

echo "[INFO] Checking instance state..."
INSTANCE_STATE=$(aws rds describe-db-instances ...)
echo "[INFO] Current state: $INSTANCE_STATE"

if [ "$INSTANCE_STATE" = "stopped" ]; then
  echo "[INFO] Instance already stopped, nothing to do"
  exit 0
fi

echo "[INFO] Stopping instance..."
aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"
echo "[SUCCESS] Stop command issued successfully"

echo "[INFO] Waiting for instance to reach 'stopped' state..."
aws rds wait db-instance-stopped --db-instance-identifier "$INSTANCE_ID"
echo "[SUCCESS] Instance stopped successfully"
```

**Why**: Good logging:
- Helps debug failures
- Provides audit trail
- Enables monitoring
- Improves user confidence

---

## Metadata Best Practices

### 1. Descriptive Naming

**DO**: Use clear, descriptive names
```json
{
  "playbook_id": "stop-rds-with-snapshot",
  "name": "Stop RDS Instance with Snapshot",
  "description": "Creates a snapshot of the RDS instance before stopping it to ensure data safety"
}
```

**DON'T**: Use vague or cryptic names
```json
{
  "playbook_id": "rds-op-1",
  "name": "RDS Thing",
  "description": "Does stuff with RDS"
}
```

---

### 2. Rich Keywords

**DO**: Include comprehensive keywords for search
```json
{
  "keywords": [
    "rds",
    "database",
    "stop",
    "shutdown",
    "halt",
    "cost-optimization",
    "cost-saving",
    "weekend",
    "scheduled",
    "snapshot",
    "backup",
    "production",
    "aws"
  ]
}
```

**DON'T**: Use minimal keywords
```json
{
  "keywords": ["rds", "stop"]
}
```

**Why**: Rich keywords improve:
- Search discoverability
- RAG vector matching
- LLM understanding
- User experience

---

### 3. Accurate Use Cases

**DO**: List specific, real-world use cases
```json
{
  "use_cases": [
    "Stop non-production databases during nights and weekends to save costs",
    "Shut down RDS instances before scheduled maintenance windows",
    "Temporarily halt databases during low-traffic periods",
    "Automated cost optimization for development environments"
  ]
}
```

**DON'T**: Be generic
```json
{
  "use_cases": [
    "Stop RDS"
  ]
}
```

---

### 4. Realistic Estimated Impact

**DO**: Provide accurate estimates
```json
{
  "estimated_impact": {
    "downtime_minutes": 10,
    "cost_savings_monthly_usd": 120,
    "cost_savings_calculation": "db.t3.medium running 16 hours/day (weekdays 8am-6pm) vs 24/7: ~$0.068/hr * 8 saved hours/day * 30 days = $16.32/mo per instance, scaled to 7 instances",
    "affected_resources": "7 RDS instances in development environment",
    "reversible": true,
    "data_loss_risk": "none"
  }
}
```

**DON'T**: Be vague
```json
{
  "estimated_impact": {
    "downtime_minutes": 5,
    "cost_savings_monthly_usd": 1000,
    "affected_resources": "some databases"
  }
}
```

---

### 5. Complete Prerequisites

**DO**: List all prerequisites
```json
{
  "prerequisites": [
    "RDS instance must be in 'available' state",
    "No active database connections (checked automatically)",
    "IAM permissions: rds:StopDBInstance, rds:DescribeDBInstances, rds:CreateDBSnapshot",
    "CloudWatch alarms configured for post-shutdown monitoring",
    "Automated snapshot policy enabled (for backup safety)",
    "Instance must have at least 1 recent snapshot (< 24 hours old)"
  ]
}
```

**DON'T**: Skip prerequisites
```json
{
  "prerequisites": [
    "Valid RDS instance"
  ]
}
```

---

## Parameter Design

### 1. Clear Parameter Names

**DO**: Use self-explanatory names
```json
{
  "name": "instance_id",
  "type": "string",
  "description": "RDS instance identifier (e.g., 'prod-db-1', 'staging-mysql')",
  "required": true
}
```

**DON'T**: Use abbreviations
```json
{
  "name": "inst_id",
  "type": "string",
  "description": "RDS ID",
  "required": true
}
```

---

### 2. Validation Rules

**DO**: Add comprehensive validation
```json
{
  "name": "instance_id",
  "type": "string",
  "description": "RDS instance identifier",
  "required": true,
  "validation": {
    "pattern": "^[a-zA-Z][a-zA-Z0-9-]{0,62}$",
    "min_length": 1,
    "max_length": 63,
    "error_message": "Instance ID must start with a letter, contain only alphanumeric characters and hyphens, and be 1-63 characters long"
  }
}
```

---

### 3. Smart Auto-Fill Strategies

**DO**: Provide helpful auto-fill strategies
```json
{
  "name": "instance_id",
  "type": "string",
  "description": "RDS instance identifier",
  "required": true,
  "auto_fill_strategy": {
    "source": "user_estate",
    "estate_query": {
      "resource_type": "rds::instance",
      "filters": {
        "tags.environment": "production",
        "state": "available",
        "tags.auto_stop_enabled": "true"
      },
      "sort_by": "created_at",
      "sort_order": "desc"
    },
    "display_field": "name",
    "value_field": "identifier",
    "display_template": "{name} ({identifier}) - {engine} {engine_version}"
  }
}
```

---

### 4. Safe Defaults

**DO**: Provide safe, conservative defaults
```json
{
  "name": "skip_snapshot",
  "type": "boolean",
  "description": "Skip creating snapshot before shutdown (NOT RECOMMENDED for production)",
  "required": false,
  "default": false,
  "validation": {
    "warning_if_true": "⚠️ Skipping snapshot is risky for production databases. You will lose the ability to restore to the pre-shutdown state."
  }
}
```

**DON'T**: Default to risky behavior
```json
{
  "name": "skip_snapshot",
  "type": "boolean",
  "default": true
}
```

---

## Script Quality Standards

### 1. Error Handling

**DO**: Handle errors gracefully
```bash
#!/bin/bash
set -euo pipefail  # Exit on error, undefined variables, pipe failures

# Function to handle errors
error_exit() {
  echo "[ERROR] $1" >&2
  exit "${2:-1}"
}

# Function to cleanup on exit
cleanup() {
  local exit_code=$?
  if [ $exit_code -ne 0 ]; then
    echo "[ERROR] Script failed with exit code $exit_code"
    echo "[INFO] Rolling back changes..."
    # Rollback logic here
  fi
}
trap cleanup EXIT

# Main logic with error handling
if ! aws rds describe-db-instances --db-instance-identifier "$INSTANCE_ID" &> /dev/null; then
  error_exit "RDS instance '$INSTANCE_ID' not found" 1
fi

echo "[INFO] Stopping instance..."
if ! aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"; then
  error_exit "Failed to stop instance" 2
fi

echo "[SUCCESS] Instance stopped successfully"
```

---

### 2. Structured Output

**DO**: Return structured JSON output
```bash
#!/bin/bash

# ... script logic ...

# Return structured output for next steps
cat <<EOF
{
  "snapshot_id": "$SNAPSHOT_ID",
  "snapshot_arn": "$SNAPSHOT_ARN",
  "instance_id": "$INSTANCE_ID",
  "stopped_at": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "estimated_monthly_savings_usd": 120
}
EOF
```

**Why**: Structured output enables:
- Chaining steps with parameter mapping
- Progress tracking
- Result validation
- Data-driven decisions

---

### 3. Timeout Handling

**DO**: Set appropriate timeouts
```json
{
  "step_number": 2,
  "name": "Create RDS snapshot",
  "type": "script",
  "script_ref": {
    "script_id": "create-rds-snapshot",
    "version": "2.1.0"
  },
  "timeout_seconds": 600,
  "continue_on_error": false
}
```

**Timeout Guidelines**:
- Quick checks: 30 seconds
- API calls: 60 seconds
- Snapshot creation: 600 seconds (10 minutes)
- Instance state changes: 300 seconds (5 minutes)
- Large data operations: 3600 seconds (1 hour)

---

### 4. Retry Logic

**DO**: Implement retry for transient failures
```bash
#!/bin/bash

# Retry function
retry_with_backoff() {
  local max_attempts=$1
  shift
  local command=("$@")
  local attempt=1
  local delay=1

  while [ $attempt -le $max_attempts ]; do
    echo "[INFO] Attempt $attempt/$max_attempts: ${command[*]}"

    if "${command[@]}"; then
      echo "[SUCCESS] Command succeeded"
      return 0
    fi

    if [ $attempt -lt $max_attempts ]; then
      echo "[WARN] Command failed, retrying in ${delay}s..."
      sleep $delay
      delay=$((delay * 2))  # Exponential backoff
      attempt=$((attempt + 1))
    else
      echo "[ERROR] Command failed after $max_attempts attempts"
      return 1
    fi
  done
}

# Usage
retry_with_backoff 3 aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"
```

---

## Error Handling

### 1. Pre-flight Checks

**DO**: Validate state before operations
```bash
#!/bin/bash

echo "[INFO] Running pre-flight checks..."

# Check 1: Instance exists
if ! aws rds describe-db-instances --db-instance-identifier "$INSTANCE_ID" &> /dev/null; then
  echo "[ERROR] Instance not found"
  exit 1
fi

# Check 2: Instance is in correct state
INSTANCE_STATE=$(aws rds describe-db-instances \
  --db-instance-identifier "$INSTANCE_ID" \
  --query 'DBInstances[0].DBInstanceStatus' \
  --output text)

if [ "$INSTANCE_STATE" != "available" ]; then
  echo "[ERROR] Instance must be in 'available' state, currently: $INSTANCE_STATE"
  exit 1
fi

# Check 3: No active connections (for stop operations)
ACTIVE_CONNECTIONS=$(aws rds describe-db-instances \
  --db-instance-identifier "$INSTANCE_ID" \
  --query 'DBInstances[0].DBInstanceStatus' \
  --output text)

if [ "$ACTIVE_CONNECTIONS" -gt 0 ]; then
  echo "[WARN] Instance has $ACTIVE_CONNECTIONS active connections"
  echo "[INFO] Proceeding anyway, but connections will be terminated"
fi

echo "[SUCCESS] Pre-flight checks passed"
```

---

### 2. Rollback Strategy

**DO**: Define clear rollback logic
```json
{
  "explain_plan": {
    "rollback_strategy": "If snapshot creation fails, no changes are made. If stop command fails after snapshot, the snapshot is preserved for manual recovery. To rollback: start the instance using 'start-rds-instance' playbook.",
    "rollback_steps": [
      {
        "condition": "snapshot_creation_failed",
        "action": "No rollback needed - no changes made"
      },
      {
        "condition": "stop_command_failed",
        "action": "Delete incomplete snapshot if it was created"
      },
      {
        "condition": "user_requests_rollback",
        "action": "Start instance using latest snapshot"
      }
    ]
  }
}
```

---

### 3. Auto-Remediation Configuration

**DO**: Configure auto-remediation for known errors
```json
{
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
        "error_pattern": "ThrottlingException",
        "action": "exponential_backoff",
        "parameters": {
          "initial_delay": 5,
          "max_delay": 60
        }
      }
    ],
    "fallback": "notify_user"
  }
}
```

---

## Testing Requirements

### 1. Test Coverage

**DO**: Test all scenarios
```markdown
## Test Plan

### Happy Path Tests
- ✅ Stop available instance successfully
- ✅ Create snapshot before stop
- ✅ Verify instance reaches 'stopped' state
- ✅ Validate output JSON structure

### Edge Case Tests
- ✅ Instance already stopped (idempotency)
- ✅ Instance in 'stopping' state (wait and verify)
- ✅ Instance has active connections (warn and proceed)
- ✅ Invalid instance ID (fail fast)

### Failure Tests
- ✅ Snapshot creation fails (rollback)
- ✅ Stop command fails (preserve snapshot)
- ✅ Timeout exceeded (cleanup and fail)
- ✅ Network error (retry logic)
- ✅ Insufficient permissions (clear error)

### Performance Tests
- ✅ Single instance: < 3 minutes
- ✅ Multiple instances (parallel): < 5 minutes
- ✅ Large instance (db.r5.24xlarge): < 10 minutes
```

---

### 2. Dry Run Mode

**DO**: Implement dry run mode
```bash
#!/bin/bash

DRY_RUN=${DRY_RUN:-false}

if [ "$DRY_RUN" = "true" ]; then
  echo "[DRY RUN] Would create snapshot: $SNAPSHOT_NAME"
  echo "[DRY RUN] Would stop instance: $INSTANCE_ID"
  echo "[DRY RUN] Estimated savings: \$120/month"
  exit 0
fi

# Actual implementation
echo "[INFO] Creating snapshot..."
aws rds create-db-snapshot ...
```

---

## Documentation Standards

### 1. Explain Plan

**DO**: Write clear, detailed explain plans
```json
{
  "explain_plan": {
    "what_it_does": "Creates a snapshot of the RDS instance and stops it gracefully to save costs during low-traffic periods",

    "why_use_it": [
      "Save 67% on RDS costs during nights and weekends",
      "Maintain data safety with automatic snapshots before shutdown",
      "Zero risk of data loss (snapshot taken before stop)",
      "Easy restart when needed (< 5 minutes to bring back online)"
    ],

    "what_happens": [
      {
        "step": 1,
        "action": "Verify RDS instance state",
        "details": "Checks that instance is in 'available' state, has no active connections, and meets prerequisites",
        "duration_seconds": 5,
        "can_fail": true,
        "rollback": null
      },
      {
        "step": 2,
        "action": "Create manual snapshot",
        "details": "Creates a snapshot named 'weekend-shutdown-{instance}-{timestamp}' for recovery if needed",
        "duration_seconds": 30,
        "can_fail": true,
        "rollback": "Delete incomplete snapshot"
      },
      {
        "step": 3,
        "action": "Stop RDS instance",
        "details": "Issues StopDBInstance command and waits for instance to reach 'stopped' state",
        "duration_seconds": 120,
        "can_fail": true,
        "rollback": "Restart instance if snapshot was created"
      }
    ],

    "risks": [
      {
        "risk": "Active connections disrupted",
        "severity": "medium",
        "probability": "low",
        "mitigation": "Pre-flight check warns if connections exist. Schedule during low-traffic periods."
      },
      {
        "risk": "Snapshot creation failure",
        "severity": "low",
        "probability": "very-low",
        "mitigation": "Playbook aborts immediately, instance remains running. No impact."
      }
    ],

    "success_criteria": [
      "Instance state is 'stopped'",
      "Snapshot created successfully with status 'available'",
      "No errors in CloudWatch logs",
      "Output JSON contains snapshot_id and instance_id"
    ]
  }
}
```

---

### 2. README Files

**DO**: Include README for each script
```markdown
# Create RDS Snapshot Script

## Overview
Creates a manual snapshot of an RDS instance with verification.

## Prerequisites
- AWS CLI v2.x
- IAM permissions: rds:CreateDBSnapshot, rds:DescribeDBSnapshots
- Python 3.8+ (for Python implementation)

## Usage

### Bash
```bash
./script.sh <instance_id> <snapshot_name> <region>
```

### Python
```bash
python main.py <instance_id> <snapshot_name> <region>
```

## Parameters
- `instance_id`: RDS instance identifier (required)
- `snapshot_name`: Name for snapshot (required)
- `region`: AWS region (required)

## Output
Returns JSON with snapshot details:
```json
{
  "snapshot_id": "snap-123",
  "snapshot_arn": "arn:aws:rds:...",
  "created_at": "2025-10-16T10:30:00Z"
}
```

## Error Codes
- 0: Success
- 1: Invalid parameters
- 2: Instance not found
- 3: Snapshot creation failed

## Examples
```bash
# Create snapshot
./script.sh prod-db-1 weekend-backup-20251016 us-east-1

# With error handling
./script.sh prod-db-1 backup || echo "Failed: $?"
```
```

---

## Security Guidelines

### 1. No Hardcoded Credentials

**DON'T**: Hardcode credentials
```bash
#!/bin/bash
AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"  # ❌ NEVER DO THIS
AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"  # ❌
```

**DO**: Use IAM roles or environment variables
```bash
#!/bin/bash
# Relies on IAM role or AWS CLI configuration
aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"
```

---

### 2. Input Sanitization

**DO**: Sanitize all inputs
```bash
#!/bin/bash

# Sanitize instance ID
INSTANCE_ID=$(echo "$INSTANCE_ID" | tr -cd 'a-zA-Z0-9-')

# Validate format
if ! [[ "$INSTANCE_ID" =~ ^[a-zA-Z][a-zA-Z0-9-]{0,62}$ ]]; then
  echo "[ERROR] Invalid instance ID format"
  exit 1
fi
```

---

### 3. Least Privilege

**DO**: Document minimum required permissions
```json
{
  "permissions_required": [
    "rds:StopDBInstance",
    "rds:DescribeDBInstances",
    "rds:CreateDBSnapshot",
    "rds:DescribeDBSnapshots"
  ],
  "permissions_optional": [
    "cloudwatch:PutMetricData"
  ]
}
```

---

### 4. Audit Logging

**DO**: Log security-relevant actions
```bash
#!/bin/bash

# Log action to CloudWatch or syslog
log_audit() {
  local action=$1
  local resource=$2
  logger -t "escher-playbook" "ACTION=$action RESOURCE=$resource USER=$USER TIMESTAMP=$(date -u +%s)"
}

log_audit "rds_stop_initiated" "$INSTANCE_ID"
aws rds stop-db-instance --db-instance-identifier "$INSTANCE_ID"
log_audit "rds_stop_completed" "$INSTANCE_ID"
```

---

## Performance Optimization

### 1. Parallel Execution

**DO**: Use parallel execution when safe
```bash
#!/bin/bash

# Stop multiple instances in parallel
INSTANCE_IDS=("prod-db-1" "prod-db-2" "prod-db-3")

for instance in "${INSTANCE_IDS[@]}"; do
  (
    echo "[INFO] Stopping $instance..."
    aws rds stop-db-instance --db-instance-identifier "$instance"
  ) &
done

wait
echo "[INFO] All instances stopped"
```

---

### 2. Efficient API Usage

**DO**: Minimize API calls
```bash
#!/bin/bash

# ❌ DON'T: Multiple API calls
for instance in "${INSTANCES[@]}"; do
  aws rds describe-db-instances --db-instance-identifier "$instance"
done

# ✅ DO: Single API call
aws rds describe-db-instances \
  --query "DBInstances[?DBInstanceIdentifier=='prod-db-1' || DBInstanceIdentifier=='prod-db-2']"
```

---

### 3. Caching

**DO**: Cache expensive operations
```bash
#!/bin/bash

CACHE_FILE="/tmp/rds-instances-cache-$(date +%Y%m%d).json"
CACHE_EXPIRY_SECONDS=3600

# Use cache if fresh
if [ -f "$CACHE_FILE" ] && [ $(($(date +%s) - $(stat -f %m "$CACHE_FILE"))) -lt $CACHE_EXPIRY_SECONDS ]; then
  echo "[INFO] Using cached instance list"
  cat "$CACHE_FILE"
else
  echo "[INFO] Refreshing instance list..."
  aws rds describe-db-instances > "$CACHE_FILE"
  cat "$CACHE_FILE"
fi
```

---

## Validation Rules

### Automated Validation

The Playbook Agent automatically validates playbooks during publish. Here are the rules:

#### 1. Metadata Validation

```typescript
interface ValidationRules {
  playbook_id: {
    required: true,
    pattern: /^[a-z0-9-]+$/,
    max_length: 100
  },
  name: {
    required: true,
    min_length: 5,
    max_length: 100
  },
  description: {
    required: true,
    min_length: 20,
    max_length: 500
  },
  keywords: {
    required: true,
    min_items: 5,
    max_items: 20
  },
  use_cases: {
    required: true,
    min_items: 2,
    max_items: 10
  },
  prerequisites: {
    required: true,
    min_items: 1
  }
}
```

#### 2. Parameter Validation

```typescript
interface ParameterRules {
  name: {
    required: true,
    pattern: /^[a-z_][a-z0-9_]*$/
  },
  description: {
    required: true,
    min_length: 10
  },
  validation: {
    required_if_user_input: true  // Must have validation if no auto_fill
  }
}
```

#### 3. Script Validation

```typescript
interface ScriptRules {
  shebang: {
    required: true,
    pattern: /^#!\/(bin\/(ba)?sh|usr\/bin\/env (python|node))/
  },
  error_handling: {
    required: true,
    checks: [
      'set -e or equivalent',
      'error exit function',
      'trap for cleanup'
    ]
  },
  output: {
    required: true,
    format: 'json',
    validation: 'must be valid JSON'
  }
}
```

#### 4. Explain Plan Validation

```typescript
interface ExplainPlanRules {
  what_it_does: {
    required: true,
    min_length: 30
  },
  what_happens: {
    required: true,
    min_items: 1,
    each_step_must_have: [
      'step',
      'action',
      'details',
      'duration_seconds'
    ]
  },
  risks: {
    required: true,
    min_items: 1,
    each_risk_must_have: [
      'risk',
      'severity',
      'mitigation'
    ]
  }
}
```

---

## Agent Integration

### How the Agent Uses Best Practices

#### 1. Validation During Publish

When a user publishes a playbook, the Playbook Agent validates it:

```rust
pub async fn publish_metadata(
    &self,
    tenant_id: &str,
    metadata: &PlaybookMetadata,
) -> Result<ValidationResult> {
    // Step 1: Validate metadata
    let validation_result = validate_metadata(metadata)?;
    if !validation_result.is_valid {
        return Err(Error::ValidationFailed(validation_result.errors));
    }

    // Step 2: Validate parameters
    let param_validation = validate_parameters(&metadata.parameters)?;
    if !param_validation.is_valid {
        return Err(Error::ValidationFailed(param_validation.errors));
    }

    // Step 3: Validate scripts (if embedded)
    let script_validation = validate_scripts(&metadata.scripts)?;
    if !script_validation.is_valid {
        return Err(Error::ValidationFailed(script_validation.errors));
    }

    // Step 4: Calculate quality score
    let quality_score = calculate_quality_score(metadata);

    // Step 5: Publish with quality score
    self.publish_with_score(tenant_id, metadata, quality_score).await
}
```

#### 2. Quality Scoring

The agent calculates a quality score (0-100) based on best practices:

```rust
fn calculate_quality_score(metadata: &PlaybookMetadata) -> f32 {
    let mut score = 0.0;

    // Metadata completeness (30 points)
    score += score_metadata_completeness(metadata) * 30.0;

    // Documentation quality (25 points)
    score += score_documentation_quality(metadata) * 25.0;

    // Parameter design (20 points)
    score += score_parameter_design(metadata) * 20.0;

    // Error handling (15 points)
    score += score_error_handling(metadata) * 15.0;

    // Testing coverage (10 points)
    score += score_testing_coverage(metadata) * 10.0;

    score.min(100.0)
}
```

**Quality Score Impact**:
- **90-100**: Featured in search, higher ranking bonus
- **70-89**: Normal ranking
- **50-69**: Lower ranking, warning shown to user
- **< 50**: Rejected or marked as "needs improvement"

#### 3. LLM Evaluation During Ranking

When ranking playbooks, the LLM considers quality:

```
LLM Prompt:
"Evaluate these playbooks for the user's need:

Playbook 1: user-weekend-shutdown
- Quality Score: 95/100
- Best Practices: ✅ Idempotent, ✅ Error handling, ✅ Clear docs
- Documentation: Excellent explain plan with rollback strategy
- Testing: Comprehensive test coverage documented

Playbook 2: user-old-shutdown
- Quality Score: 62/100
- Best Practices: ⚠️ No error handling, ⚠️ Minimal docs
- Documentation: Basic explain plan
- Testing: No test coverage documented

Rank playbooks considering quality scores and best practices adherence."
```

#### 4. User Feedback

The agent provides feedback during publish:

```json
{
  "status": "published",
  "quality_score": 87,
  "feedback": {
    "strengths": [
      "Excellent error handling with rollback strategy",
      "Comprehensive explain plan with clear risks",
      "Good parameter validation with auto-fill",
      "Idempotent design - safe to retry"
    ],
    "improvements": [
      "Add more keywords for better searchability (currently 6, recommended 10+)",
      "Include estimated impact calculations",
      "Add test coverage documentation"
    ],
    "warnings": [],
    "required_fixes": []
  }
}
```

---

## Checklist for Playbook Authors

Use this checklist before publishing a playbook:

### Metadata
- [ ] Descriptive playbook_id (kebab-case)
- [ ] Clear name (5-100 chars)
- [ ] Detailed description (20-500 chars)
- [ ] 5+ keywords for search
- [ ] 2+ specific use cases
- [ ] Complete prerequisites list
- [ ] Accurate estimated impact

### Parameters
- [ ] Clear parameter names (snake_case)
- [ ] Detailed descriptions
- [ ] Validation rules for user inputs
- [ ] Auto-fill strategies where applicable
- [ ] Safe defaults
- [ ] Warning for risky options

### Scripts
- [ ] Error handling (set -e, error functions)
- [ ] Pre-flight checks
- [ ] Idempotent behavior
- [ ] Comprehensive logging
- [ ] Structured JSON output
- [ ] Retry logic for transient failures
- [ ] Timeout handling
- [ ] No hardcoded credentials
- [ ] Input sanitization

### Documentation
- [ ] Complete explain plan
- [ ] What it does (30+ chars)
- [ ] Why use it (2+ reasons)
- [ ] What happens (all steps documented)
- [ ] Risks and mitigations
- [ ] Rollback strategy
- [ ] Success criteria
- [ ] README for each script

### Testing
- [ ] Happy path tested
- [ ] Edge cases tested
- [ ] Failure scenarios tested
- [ ] Idempotency tested
- [ ] Dry run mode implemented
- [ ] Test coverage documented

### Security
- [ ] No hardcoded credentials
- [ ] Input sanitization
- [ ] Minimum permissions documented
- [ ] Audit logging

### Performance
- [ ] Appropriate timeouts
- [ ] Efficient API usage
- [ ] Parallel execution where safe
- [ ] Caching for expensive operations

---

## Related Documentation

- [Playbook Agent](playbook-agent.md) - Main agent documentation
- [Playbook Structure](playbook-agent.md#playbook-structure) - Detailed structure reference
- [Storage Strategies](playbook-agent.md#storage-strategies) - Publishing and sharing playbooks