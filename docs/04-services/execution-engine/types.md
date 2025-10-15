# Execution Engine Types Reference

**Crate**: `cloudops-execution-engine`

Complete reference for all data types, with Rust definitions and JSON schemas.

---

## Table of Contents

- [Input Types](#input-types)
  - [ExecutionRequest](#executionrequest)
  - [ExecutionPlan](#executionplan)
  - [Command](#command)
  - [ExecutionStrategy](#executionstrategy)
- [Output Types](#output-types)
  - [ExecutionResult](#executionresult)
  - [PlanExecutionResult](#planexecutionresult)
  - [ExecutionStatus](#executionstatus)
  - [ExecutionStats](#executionstats)
- [Event Types](#event-types)
  - [ExecutionEvent](#executionevent)
- [Metadata Types](#metadata-types)
  - [ExecutionMetadata](#executionmetadata)
  - [ExecutionSummary](#executionsummary)
- [Internal Types](#internal-types)
  - [ExecutionState](#executionstate)
- [Type Conversions](#type-conversions)

---

## Input Types

### ExecutionRequest

Single command execution request.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionRequest {
    /// Unique identifier for this request
    pub id: Uuid,

    /// Command to execute
    pub command: Command,

    /// Environment variables
    #[serde(default)]
    pub env: HashMap<String, String>,

    /// Working directory (optional)
    pub working_dir: Option<PathBuf>,

    /// Timeout in milliseconds (optional)
    pub timeout_ms: Option<u64>,

    /// Metadata for tracking
    #[serde(default)]
    pub metadata: ExecutionMetadata,
}
```

#### JSON Schema

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "command": {
    "type": "aws_cli",
    "service": "ec2",
    "operation": "describe-instances",
    "args": ["--region", "us-west-2"],
    "profile": "production",
    "region": "us-west-2"
  },
  "env": {
    "AWS_PROFILE": "production",
    "DEBUG": "true"
  },
  "working_dir": "/tmp",
  "timeout_ms": 300000,
  "metadata": {
    "source": "server",
    "conversation_id": "660e8400-e29b-41d4-a716-446655440000",
    "tags": {
      "environment": "production",
      "region": "us-west-2"
    }
  }
}
```

#### Field Details

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | Uuid | Yes | - | Unique identifier |
| `command` | Command | Yes | - | Command to execute |
| `env` | HashMap<String, String> | No | {} | Environment variables |
| `working_dir` | Option<PathBuf> | No | None | Working directory |
| `timeout_ms` | Option<u64> | No | None | Timeout (uses config default) |
| `metadata` | ExecutionMetadata | No | Default | Tracking metadata |

#### Validation Rules

- `id` must be a valid UUID v4
- `command` must be valid (see [Command validation](#command))
- `timeout_ms` must not exceed `max_timeout_ms` from config
- `working_dir` must exist and be a directory if specified

---

### ExecutionPlan

Plan for executing multiple commands.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionPlan {
    /// Unique identifier for this plan
    pub id: Uuid,

    /// Human-readable description
    pub description: String,

    /// Execution strategy
    pub strategy: ExecutionStrategy,

    /// Commands to execute
    pub commands: Vec<ExecutionRequest>,

    /// Metadata
    #[serde(default)]
    pub metadata: ExecutionMetadata,
}
```

#### JSON Schema

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440000",
  "description": "Deploy application to production",
  "strategy": {
    "type": "serial",
    "stop_on_error": true
  },
  "commands": [
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "command": {
        "type": "script",
        "path": "/tmp/pre-deploy.sh",
        "interpreter": null
      },
      "env": {},
      "timeout_ms": 60000
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440002",
      "command": {
        "type": "aws_cli",
        "service": "ecs",
        "operation": "update-service",
        "args": ["--cluster", "prod", "--service", "api"],
        "profile": "production",
        "region": "us-west-2"
      },
      "env": {},
      "timeout_ms": 300000
    }
  ],
  "metadata": {
    "source": "server",
    "conversation_id": "770e8400-e29b-41d4-a716-446655440000"
  }
}
```

#### Field Details

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | Uuid | Yes | Unique identifier for plan |
| `description` | String | Yes | Human-readable description |
| `strategy` | ExecutionStrategy | Yes | How to execute commands |
| `commands` | Vec<ExecutionRequest> | Yes | Commands to execute |
| `metadata` | ExecutionMetadata | No | Tracking metadata |

---

### Command

Standardized command types.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum Command {
    /// Execute a script file
    Script {
        path: PathBuf,
        interpreter: Option<String>,
    },

    /// Execute command with arguments
    Exec {
        program: String,
        args: Vec<String>,
    },

    /// Execute shell command string
    Shell {
        command: String,
        #[serde(default = "default_shell")]
        shell: String,
    },

    /// AWS CLI command (convenience)
    AwsCli {
        service: String,
        operation: String,
        #[serde(default)]
        args: Vec<String>,
        profile: Option<String>,
        region: Option<String>,
    },
}

fn default_shell() -> String {
    "bash".to_string()
}
```

#### Variants

##### Script

```rust
Command::Script {
    path: PathBuf::from("/tmp/deploy.sh"),
    interpreter: None, // Auto-detect from shebang
}
```

```json
{
  "type": "script",
  "path": "/tmp/deploy.sh",
  "interpreter": null
}
```

**Validation:**
- `path` must exist
- `path` must be executable or have valid shebang

##### Exec

```rust
Command::Exec {
    program: "python3".to_string(),
    args: vec!["script.py".to_string(), "--verbose".to_string()],
}
```

```json
{
  "type": "exec",
  "program": "python3",
  "args": ["script.py", "--verbose"]
}
```

**Validation:**
- `program` must not be empty
- `program` should be in PATH or absolute path

##### Shell

```rust
Command::Shell {
    command: "ls -la | grep .rs".to_string(),
    shell: "bash".to_string(),
}
```

```json
{
  "type": "shell",
  "command": "ls -la | grep .rs",
  "shell": "bash"
}
```

**Validation:**
- `command` must not be empty
- `shell` must be valid shell (bash, sh, zsh, etc.)

##### AwsCli

```rust
Command::AwsCli {
    service: "ec2".to_string(),
    operation: "describe-instances".to_string(),
    args: vec!["--filters".to_string(), "Name=instance-state-name,Values=running".to_string()],
    profile: Some("production".to_string()),
    region: Some("us-west-2".to_string()),
}
```

```json
{
  "type": "aws_cli",
  "service": "ec2",
  "operation": "describe-instances",
  "args": ["--filters", "Name=instance-state-name,Values=running"],
  "profile": "production",
  "region": "us-west-2"
}
```

**Validation:**
- `service` and `operation` must not be empty
- AWS CLI must be available in PATH

---

### ExecutionStrategy

Strategy for executing multiple commands.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ExecutionStrategy {
    /// Execute sequentially, stop on error
    Serial {
        #[serde(default = "default_stop_on_error")]
        stop_on_error: bool,
    },

    /// Execute concurrently
    Parallel {
        max_concurrency: Option<usize>,
    },

    /// Execute based on dependency graph
    DependencyGraph {
        dependencies: HashMap<usize, Vec<usize>>,
    },
}

fn default_stop_on_error() -> bool {
    true
}
```

#### Variants

##### Serial

```json
{
  "type": "serial",
  "stop_on_error": true
}
```

Execute commands one by one. If `stop_on_error` is true, stop on first failure.

##### Parallel

```json
{
  "type": "parallel",
  "max_concurrency": 5
}
```

Execute all commands concurrently. `max_concurrency` limits parallel executions (null = unlimited).

##### DependencyGraph

```json
{
  "type": "dependency_graph",
  "dependencies": {
    "2": [0, 1],
    "3": [2]
  }
}
```

Command at index 2 depends on commands 0 and 1. Command 3 depends on command 2.

---

## Output Types

### ExecutionResult

Result of a single command execution.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionResult {
    /// Execution ID
    pub id: Uuid,

    /// Final status
    pub status: ExecutionStatus,

    /// Process exit code
    pub exit_code: i32,

    /// Standard output
    pub stdout: String,

    /// Standard error
    pub stderr: String,

    /// Execution duration
    #[serde(with = "humantime_serde")]
    pub duration: Duration,

    /// Start timestamp
    pub started_at: DateTime<Utc>,

    /// Completion timestamp
    pub completed_at: Option<DateTime<Utc>>,

    /// Log file path
    pub log_path: PathBuf,

    /// Error message (if failed)
    pub error: Option<String>,
}
```

#### JSON Example

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "exit_code": 0,
  "stdout": "{\n  \"Reservations\": [...]\n}",
  "stderr": "",
  "duration": "2.5s",
  "started_at": "2025-10-09T10:30:00Z",
  "completed_at": "2025-10-09T10:30:02Z",
  "log_path": "/tmp/cloudops-executions/550e8400-e29b-41d4-a716-446655440000.log",
  "error": null
}
```

#### Field Details

| Field | Type | Description |
|-------|------|-------------|
| `id` | Uuid | Execution request ID |
| `status` | ExecutionStatus | Final status |
| `exit_code` | i32 | Process exit code (0 = success) |
| `stdout` | String | Complete standard output |
| `stderr` | String | Complete standard error |
| `duration` | Duration | Total execution time |
| `started_at` | DateTime<Utc> | When execution started |
| `completed_at` | Option<DateTime<Utc>> | When execution completed |
| `log_path` | PathBuf | Path to log file |
| `error` | Option<String> | Error message if failed |

---

### PlanExecutionResult

Result of executing a plan.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PlanExecutionResult {
    /// Plan ID
    pub plan_id: Uuid,

    /// Overall status
    pub status: ExecutionStatus,

    /// Results for each command
    pub results: Vec<ExecutionResult>,

    /// Total duration
    #[serde(with = "humantime_serde")]
    pub total_duration: Duration,

    /// Statistics
    pub stats: ExecutionStats,
}
```

#### JSON Example

```json
{
  "plan_id": "660e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "results": [
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "status": "completed",
      "exit_code": 0,
      "stdout": "Pre-deploy checks passed",
      "stderr": "",
      "duration": "1.2s",
      "started_at": "2025-10-09T10:30:00Z",
      "completed_at": "2025-10-09T10:30:01Z",
      "log_path": "/tmp/cloudops-executions/660e8400-e29b-41d4-a716-446655440001.log",
      "error": null
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440002",
      "status": "completed",
      "exit_code": 0,
      "stdout": "Service updated successfully",
      "stderr": "",
      "duration": "15.3s",
      "started_at": "2025-10-09T10:30:01Z",
      "completed_at": "2025-10-09T10:30:16Z",
      "log_path": "/tmp/cloudops-executions/660e8400-e29b-41d4-a716-446655440002.log",
      "error": null
    }
  ],
  "total_duration": "16.5s",
  "stats": {
    "total": 2,
    "completed": 2,
    "failed": 0,
    "cancelled": 0,
    "timeout": 0
  }
}
```

---

### ExecutionStatus

Status enum for executions.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
pub enum ExecutionStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Cancelled,
    Timeout,
}
```

#### Variants

| Variant | Description | Terminal State |
|---------|-------------|----------------|
| `Pending` | Created but not started | No |
| `Running` | Currently executing | No |
| `Completed` | Finished successfully | Yes |
| `Failed` | Failed with error | Yes |
| `Cancelled` | Cancelled by user | Yes |
| `Timeout` | Exceeded timeout | Yes |

#### State Transitions

```
Pending → Running → Completed
                 → Failed
                 → Cancelled
                 → Timeout
```

#### JSON Values

```json
"pending"
"running"
"completed"
"failed"
"cancelled"
"timeout"
```

---

### ExecutionStats

Statistics for plan execution.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionStats {
    pub total: usize,
    pub completed: usize,
    pub failed: usize,
    pub cancelled: usize,
    pub timeout: usize,
}
```

#### JSON Example

```json
{
  "total": 10,
  "completed": 7,
  "failed": 2,
  "cancelled": 1,
  "timeout": 0
}
```

---

## Event Types

### ExecutionEvent

Events emitted during execution.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "event_type", rename_all = "snake_case")]
pub enum ExecutionEvent {
    Started {
        execution_id: Uuid,
        command: String,
        timestamp: DateTime<Utc>,
    },
    Stdout {
        execution_id: Uuid,
        line: String,
        timestamp: DateTime<Utc>,
    },
    Stderr {
        execution_id: Uuid,
        line: String,
        timestamp: DateTime<Utc>,
    },
    Completed {
        execution_id: Uuid,
        result: ExecutionResult,
        timestamp: DateTime<Utc>,
    },
    Failed {
        execution_id: Uuid,
        error: String,
        timestamp: DateTime<Utc>,
    },
    Cancelled {
        execution_id: Uuid,
        timestamp: DateTime<Utc>,
    },
    Progress {
        plan_id: Uuid,
        completed: usize,
        total: usize,
        current_command: Option<String>,
        timestamp: DateTime<Utc>,
    },
}
```

#### JSON Examples

##### Started Event

```json
{
  "event_type": "started",
  "execution_id": "550e8400-e29b-41d4-a716-446655440000",
  "command": "aws ec2 describe-instances",
  "timestamp": "2025-10-09T10:30:00Z"
}
```

##### Stdout Event

```json
{
  "event_type": "stdout",
  "execution_id": "550e8400-e29b-41d4-a716-446655440000",
  "line": "{\"Reservations\": [",
  "timestamp": "2025-10-09T10:30:01Z"
}
```

##### Completed Event

```json
{
  "event_type": "completed",
  "execution_id": "550e8400-e29b-41d4-a716-446655440000",
  "result": { /* ExecutionResult */ },
  "timestamp": "2025-10-09T10:30:02Z"
}
```

##### Progress Event

```json
{
  "event_type": "progress",
  "plan_id": "660e8400-e29b-41d4-a716-446655440000",
  "completed": 3,
  "total": 5,
  "current_command": "aws ecs update-service",
  "timestamp": "2025-10-09T10:30:05Z"
}
```

---

## Metadata Types

### ExecutionMetadata

Metadata for tracking and debugging.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct ExecutionMetadata {
    /// Source of request
    pub source: Option<String>,

    /// Related conversation ID
    pub conversation_id: Option<Uuid>,

    /// Custom tags
    #[serde(default)]
    pub tags: HashMap<String, String>,
}
```

#### JSON Example

```json
{
  "source": "server",
  "conversation_id": "770e8400-e29b-41d4-a716-446655440000",
  "tags": {
    "environment": "production",
    "region": "us-west-2",
    "user": "admin@example.com"
  }
}
```

---

### ExecutionSummary

Summary information for listing executions.

#### Rust Definition

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionSummary {
    pub id: Uuid,
    pub status: ExecutionStatus,
    pub started_at: DateTime<Utc>,
    pub duration: Option<Duration>,
}
```

#### JSON Example

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "started_at": "2025-10-09T10:30:00Z",
  "duration": "2.5s"
}
```

---

## Internal Types

### ExecutionState

Internal state tracking (not serialized to JSON).

#### Rust Definition

```rust
#[derive(Debug, Clone)]
pub struct ExecutionState {
    pub id: Uuid,
    pub status: ExecutionStatus,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub result: Option<ExecutionResult>,
    pub cancel_token: CancellationToken,
    pub log_path: PathBuf,
}
```

**Note:** This type is internal and not exposed via public API.

---

## Type Conversions

### From ExecutionRequest to JSON

```rust
let request = ExecutionRequest {
    id: Uuid::new_v4(),
    command: Command::AwsCli {
        service: "ec2".to_string(),
        operation: "describe-instances".to_string(),
        args: vec![],
        profile: Some("prod".to_string()),
        region: Some("us-west-2".to_string()),
    },
    env: HashMap::new(),
    working_dir: None,
    timeout_ms: None,
    metadata: Default::default(),
};

let json = serde_json::to_string(&request)?;
```

### From JSON to ExecutionRequest

```rust
let json = r#"{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "command": {
    "type": "aws_cli",
    "service": "ec2",
    "operation": "describe-instances",
    "args": [],
    "profile": "prod",
    "region": "us-west-2"
  },
  "env": {},
  "working_dir": null,
  "timeout_ms": null,
  "metadata": {}
}"#;

let request: ExecutionRequest = serde_json::from_str(json)?;
```

### Duration Serialization

Duration is serialized using `humantime_serde`:

```rust
Duration::from_secs(150) // → "2m 30s" in JSON
```

---

## Type Safety Guarantees

### Compile-Time Safety

- ✅ All types are strongly typed with Rust
- ✅ Serde ensures serialization correctness
- ✅ Enum variants prevent invalid states
- ✅ Option types for nullable fields

### Runtime Validation

- ✅ `ExecutionRequest::validate()` checks constraints
- ✅ `Command::validate()` verifies command validity
- ✅ Timeout bounds checked against config
- ✅ Path existence validated

### JSON Schema Generation

Generate JSON schemas for external consumers:

```rust
use schemars::{schema_for, JsonSchema};

#[derive(JsonSchema)]
pub struct ExecutionRequest { /* ... */ }

let schema = schema_for!(ExecutionRequest);
println!("{}", serde_json::to_string_pretty(&schema)?);
```

---

## Best Practices

### 1. Always Validate Inputs

```rust
request.validate(&config)?;
```

### 2. Use Type-Safe Builders

```rust
let request = ExecutionRequestBuilder::new()
    .command(Command::AwsCli { /* ... */ })
    .timeout_ms(300_000)
    .env("AWS_PROFILE", "prod")
    .build()?;
```

### 3. Handle All Status Variants

```rust
match status {
    ExecutionStatus::Completed => { /* success */ },
    ExecutionStatus::Failed => { /* handle error */ },
    ExecutionStatus::Timeout => { /* handle timeout */ },
    ExecutionStatus::Cancelled => { /* handle cancellation */ },
    _ => { /* unexpected */ },
}
```

### 4. Check Exit Codes

```rust
if result.exit_code == 0 {
    // Success
} else {
    // Check stderr for error details
    eprintln!("Error: {}", result.stderr);
}
```

---

## Related Documents

- [API Reference](api.md)
- [Architecture](architecture.md)
- [Usage Examples](usage.md)