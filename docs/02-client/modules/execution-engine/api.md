# Execution Engine API Reference

**Crate**: `cloudops-execution-engine`
**Version**: 0.1.0

Complete API documentation for the Execution Engine module.

---

## Table of Contents

- [Core API](#core-api)
  - [ExecutionEngine](#executionengine)
  - [ExecutionConfig](#executionconfig)
- [Execution Types](#execution-types)
  - [ExecutionRequest](#executionrequest)
  - [ExecutionPlan](#executionplan)
  - [Command](#command)
  - [ExecutionStrategy](#executionstrategy)
- [Result Types](#result-types)
  - [ExecutionResult](#executionresult)
  - [PlanExecutionResult](#planexecutionresult)
  - [ExecutionStatus](#executionstatus)
- [Event System](#event-system)
  - [EventHandler](#eventhandler)
  - [ExecutionEvent](#executionevent)
- [Error Types](#error-types)
  - [ExecutionError](#executionerror)
  - [ValidationError](#validationerror)
- [Metadata Types](#metadata-types)

---

## Core API

### ExecutionEngine

Main entry point for command execution.

```rust
pub struct ExecutionEngine {
    config: ExecutionConfig,
    executions: Arc<RwLock<HashMap<Uuid, ExecutionState>>>,
    event_handler: Option<Arc<dyn EventHandler>>,
}
```

#### Constructor

##### `new`

```rust
pub fn new(config: ExecutionConfig) -> Self
```

Create a new execution engine with the given configuration.

**Parameters:**
- `config`: ExecutionConfig - Configuration for the engine

**Returns:** ExecutionEngine instance

**Example:**
```rust
let engine = ExecutionEngine::new(ExecutionConfig::default());
```

##### `with_event_handler`

```rust
pub fn with_event_handler(
    self,
    handler: Arc<dyn EventHandler>
) -> Self
```

Add a custom event handler for real-time updates.

**Parameters:**
- `handler`: Arc<dyn EventHandler> - Event handler implementation

**Returns:** Modified ExecutionEngine instance

**Example:**
```rust
let engine = ExecutionEngine::new(config)
    .with_event_handler(Arc::new(MyEventHandler));
```

#### Execution Methods

##### `execute`

```rust
pub async fn execute(
    &self,
    request: ExecutionRequest
) -> Result<Uuid, ExecutionError>
```

Execute a single command in the background.

**Parameters:**
- `request`: ExecutionRequest - Command execution request

**Returns:**
- `Ok(Uuid)`: Execution ID for tracking
- `Err(ExecutionError)`: Validation or execution error

**Example:**
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

let execution_id = engine.execute(request).await?;
```

##### `execute_plan`

```rust
pub async fn execute_plan(
    &self,
    plan: ExecutionPlan
) -> Result<Uuid, ExecutionError>
```

Execute multiple commands according to a plan (serial, parallel, or dependency-based).

**Parameters:**
- `plan`: ExecutionPlan - Execution plan with multiple commands

**Returns:**
- `Ok(Uuid)`: Plan execution ID for tracking
- `Err(ExecutionError)`: Validation or execution error

**Example:**
```rust
let plan = ExecutionPlan {
    id: Uuid::new_v4(),
    description: "Deploy application".to_string(),
    strategy: ExecutionStrategy::Serial {
        stop_on_error: true,
    },
    commands: vec![request1, request2, request3],
    metadata: Default::default(),
};

let plan_id = engine.execute_plan(plan).await?;
```

#### Query Methods

##### `get_status`

```rust
pub async fn get_status(
    &self,
    execution_id: Uuid
) -> Result<ExecutionStatus, ExecutionError>
```

Get the current status of an execution.

**Parameters:**
- `execution_id`: Uuid - Execution identifier

**Returns:**
- `Ok(ExecutionStatus)`: Current status
- `Err(ExecutionError::NotFound)`: Execution not found

**Example:**
```rust
let status = engine.get_status(execution_id).await?;
if status == ExecutionStatus::Completed {
    println!("Execution completed!");
}
```

##### `get_result`

```rust
pub async fn get_result(
    &self,
    execution_id: Uuid
) -> Result<ExecutionResult, ExecutionError>
```

Get the result of a completed execution.

**Parameters:**
- `execution_id`: Uuid - Execution identifier

**Returns:**
- `Ok(ExecutionResult)`: Execution result with output
- `Err(ExecutionError::NotFound)`: Execution not found
- `Err(ExecutionError)`: Execution not yet completed

**Example:**
```rust
let result = engine.get_result(execution_id).await?;
println!("Exit code: {}", result.exit_code);
println!("Output: {}", result.stdout);
```

##### `list_executions`

```rust
pub async fn list_executions(&self) -> Vec<ExecutionSummary>
```

List all executions tracked by this engine.

**Returns:** Vector of execution summaries

**Example:**
```rust
let executions = engine.list_executions().await;
for exec in executions {
    println!("{}: {:?}", exec.id, exec.status);
}
```

#### Control Methods

##### `cancel`

```rust
pub async fn cancel(
    &self,
    execution_id: Uuid
) -> Result<(), ExecutionError>
```

Cancel a running execution.

**Parameters:**
- `execution_id`: Uuid - Execution identifier

**Returns:**
- `Ok(())`: Cancellation initiated
- `Err(ExecutionError::NotFound)`: Execution not found
- `Err(ExecutionError)`: Execution not running

**Example:**
```rust
engine.cancel(execution_id).await?;
```

#### Log Methods

##### `read_logs`

```rust
pub async fn read_logs(
    &self,
    execution_id: Uuid
) -> Result<String, ExecutionError>
```

Read the complete log file for an execution.

**Parameters:**
- `execution_id`: Uuid - Execution identifier

**Returns:**
- `Ok(String)`: Complete log contents
- `Err(ExecutionError)`: Log file not found or IO error

**Example:**
```rust
let logs = engine.read_logs(execution_id).await?;
println!("Logs:\n{}", logs);
```

##### `get_log_path`

```rust
pub fn get_log_path(&self, execution_id: Uuid) -> PathBuf
```

Get the path to the log file for an execution.

**Parameters:**
- `execution_id`: Uuid - Execution identifier

**Returns:** PathBuf to log file location

**Example:**
```rust
let log_path = engine.get_log_path(execution_id);
println!("Logs at: {:?}", log_path);
```

---

## ExecutionConfig

Configuration for the execution engine.

```rust
pub struct ExecutionConfig {
    pub default_timeout_ms: u64,
    pub max_timeout_ms: u64,
    pub stream_output: bool,
    pub log_dir: Option<PathBuf>,
    pub max_concurrent_executions: usize,
}
```

### Fields

- **`default_timeout_ms`**: Default timeout in milliseconds (default: 120,000 = 2 minutes)
- **`max_timeout_ms`**: Maximum allowed timeout (default: 600,000 = 10 minutes)
- **`stream_output`**: Enable real-time output streaming (default: true)
- **`log_dir`**: Custom log directory (default: None = system temp dir)
- **`max_concurrent_executions`**: Maximum concurrent executions (default: 10)

### Methods

#### `default`

```rust
impl Default for ExecutionConfig {
    fn default() -> Self
}
```

Create configuration with default values.

**Example:**
```rust
let config = ExecutionConfig::default();
```

#### Custom Configuration

```rust
let config = ExecutionConfig {
    default_timeout_ms: 300_000, // 5 minutes
    max_timeout_ms: 1_800_000,   // 30 minutes
    stream_output: true,
    log_dir: Some(PathBuf::from("/var/log/cloudops")),
    max_concurrent_executions: 20,
};
```

---

## Execution Types

### ExecutionRequest

Request to execute a single command.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionRequest {
    pub id: Uuid,
    pub command: Command,
    pub env: HashMap<String, String>,
    pub working_dir: Option<PathBuf>,
    pub timeout_ms: Option<u64>,
    pub metadata: ExecutionMetadata,
}
```

### Fields

- **`id`**: Unique identifier for this request
- **`command`**: Command to execute (see [Command](#command))
- **`env`**: Environment variables (default: empty)
- **`working_dir`**: Working directory (default: current directory)
- **`timeout_ms`**: Timeout in milliseconds (default: uses config default)
- **`metadata`**: Metadata for tracking and debugging

#### Methods

##### `validate`

```rust
pub fn validate(&self, config: &ExecutionConfig) -> Result<(), ValidationError>
```

Validate the execution request.

**Returns:**
- `Ok(())`: Request is valid
- `Err(ValidationError)`: Validation failed

---

### ExecutionPlan

Plan for executing multiple commands.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionPlan {
    pub id: Uuid,
    pub description: String,
    pub strategy: ExecutionStrategy,
    pub commands: Vec<ExecutionRequest>,
    pub metadata: ExecutionMetadata,
}
```

### Fields

- **`id`**: Unique identifier for this plan
- **`description`**: Human-readable description
- **`strategy`**: Execution strategy (serial, parallel, etc.)
- **`commands`**: List of commands to execute
- **`metadata`**: Metadata for tracking

---

### Command

Standardized command types.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum Command {
    Script {
        path: PathBuf,
        interpreter: Option<String>,
    },
    Exec {
        program: String,
        args: Vec<String>,
    },
    Shell {
        command: String,
        shell: String,
    },
    AwsCli {
        service: String,
        operation: String,
        args: Vec<String>,
        profile: Option<String>,
        region: Option<String>,
    },
}
```

### Variants

#### `Script`

Execute a script file.

**Fields:**
- `path`: PathBuf - Path to script file
- `interpreter`: Option<String> - Interpreter to use (auto-detected if None)

**Example:**
```rust
Command::Script {
    path: PathBuf::from("/tmp/deploy.sh"),
    interpreter: None,
}
```

**JSON:**
```json
{
  "type": "script",
  "path": "/tmp/deploy.sh",
  "interpreter": null
}
```

#### `Exec`

Execute a command with arguments.

**Fields:**
- `program`: String - Program to execute
- `args`: Vec<String> - Arguments

**Example:**
```rust
Command::Exec {
    program: "python3".to_string(),
    args: vec!["script.py".to_string(), "--input".to_string(), "data.json".to_string()],
}
```

**JSON:**
```json
{
  "type": "exec",
  "program": "python3",
  "args": ["script.py", "--input", "data.json"]
}
```

#### `Shell`

Execute a shell command string.

**Fields:**
- `command`: String - Command string
- `shell`: String - Shell to use (default: "bash")

**Example:**
```rust
Command::Shell {
    command: "aws ec2 describe-instances | jq '.Reservations[]'".to_string(),
    shell: "bash".to_string(),
}
```

**JSON:**
```json
{
  "type": "shell",
  "command": "aws ec2 describe-instances | jq '.Reservations[]'",
  "shell": "bash"
}
```

#### `AwsCli`

Execute AWS CLI command (convenience wrapper).

**Fields:**
- `service`: String - AWS service (e.g., "ec2", "s3")
- `operation`: String - Operation (e.g., "describe-instances")
- `args`: Vec<String> - Additional arguments
- `profile`: Option<String> - AWS profile
- `region`: Option<String> - AWS region

**Example:**
```rust
Command::AwsCli {
    service: "ec2".to_string(),
    operation: "describe-instances".to_string(),
    args: vec!["--region".to_string(), "us-west-2".to_string()],
    profile: Some("production".to_string()),
    region: Some("us-west-2".to_string()),
}
```

**JSON:**
```json
{
  "type": "aws_cli",
  "service": "ec2",
  "operation": "describe-instances",
  "args": ["--region", "us-west-2"],
  "profile": "production",
  "region": "us-west-2"
}
```

#### Methods

##### `validate`

```rust
pub fn validate(&self) -> Result<(), ValidationError>
```

Validate the command.

---

### ExecutionStrategy

Strategy for executing multiple commands.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ExecutionStrategy {
    Serial {
        stop_on_error: bool,
    },
    Parallel {
        max_concurrency: Option<usize>,
    },
    DependencyGraph {
        dependencies: HashMap<usize, Vec<usize>>,
    },
}
```

### Variants

#### `Serial`

Execute commands one by one.

**Fields:**
- `stop_on_error`: bool - Stop on first error (default: true)

**Example:**
```rust
ExecutionStrategy::Serial {
    stop_on_error: true,
}
```

#### `Parallel`

Execute commands concurrently.

**Fields:**
- `max_concurrency`: Option<usize> - Maximum concurrent executions (None = unlimited)

**Example:**
```rust
ExecutionStrategy::Parallel {
    max_concurrency: Some(5),
}
```

#### `DependencyGraph`

Execute based on dependency graph.

**Fields:**
- `dependencies`: HashMap<usize, Vec<usize>> - Map of command index to dependency indices

**Example:**
```rust
// Command 2 depends on commands 0 and 1
ExecutionStrategy::DependencyGraph {
    dependencies: HashMap::from([
        (2, vec![0, 1]),
    ]),
}
```

---

## Result Types

### ExecutionResult

Result of a single command execution.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionResult {
    pub id: Uuid,
    pub status: ExecutionStatus,
    pub exit_code: i32,
    pub stdout: String,
    pub stderr: String,
    pub duration: Duration,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub log_path: PathBuf,
    pub error: Option<String>,
}
```

### Fields

- **`id`**: Execution request ID
- **`status`**: Final status
- **`exit_code`**: Process exit code
- **`stdout`**: Standard output
- **`stderr`**: Standard error
- **`duration`**: Execution duration
- **`started_at`**: Start timestamp
- **`completed_at`**: Completion timestamp
- **`log_path`**: Path to log file
- **`error`**: Error message if failed

---

### PlanExecutionResult

Result of executing a plan.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PlanExecutionResult {
    pub plan_id: Uuid,
    pub status: ExecutionStatus,
    pub results: Vec<ExecutionResult>,
    pub total_duration: Duration,
    pub stats: ExecutionStats,
}
```

### Fields

- **`plan_id`**: Plan identifier
- **`status`**: Overall status
- **`results`**: Results for each command
- **`total_duration`**: Total execution time
- **`stats`**: Execution statistics

---

### ExecutionStatus

Status of an execution.

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

### Variants

- **`Pending`**: Execution created but not started
- **`Running`**: Currently executing
- **`Completed`**: Completed successfully
- **`Failed`**: Failed with error
- **`Cancelled`**: Cancelled by user
- **`Timeout`**: Timed out

---

## Event System

### EventHandler

Trait for custom event handlers.

```rust
#[async_trait::async_trait]
pub trait EventHandler: Send + Sync {
    async fn on_event(&self, event: ExecutionEvent);
}
```

### Method

#### `on_event`

```rust
async fn on_event(&self, event: ExecutionEvent)
```

Handle an execution event.

**Parameters:**
- `event`: ExecutionEvent - Event to handle

**Example Implementation:**
```rust
struct MyEventHandler;

#[async_trait]
impl EventHandler for MyEventHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match event {
            ExecutionEvent::Stdout { execution_id, line } => {
                println!("[{}] {}", execution_id, line);
            }
            ExecutionEvent::Completed { execution_id, result } => {
                println!("Execution {} completed", execution_id);
            }
            _ => {}
        }
    }
}
```

---

### ExecutionEvent

Events emitted during execution.

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

### Variants

- **`Started`**: Execution started
- **`Stdout`**: Line of stdout output
- **`Stderr`**: Line of stderr output
- **`Completed`**: Execution completed
- **`Failed`**: Execution failed
- **`Cancelled`**: Execution cancelled
- **`Progress`**: Progress update for plans

---

## Error Types

### ExecutionError

Main error type for execution operations.

```rust
#[derive(Debug, thiserror::Error)]
pub enum ExecutionError {
    #[error("Validation error: {0}")]
    Validation(#[from] ValidationError),

    #[error("Execution not found: {0}")]
    NotFound(Uuid),

    #[error("Execution timeout after {0}ms")]
    Timeout(u64),

    #[error("Execution cancelled")]
    Cancelled,

    #[error("Command failed with exit code {0}")]
    CommandFailed(i32),

    #[error("Process spawn failed: {0}")]
    SpawnFailed(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),
}
```

---

### ValidationError

Validation error types.

```rust
#[derive(Debug, thiserror::Error)]
pub enum ValidationError {
    #[error("Invalid command format: {0}")]
    InvalidCommand(String),

    #[error("Script file not found: {0}")]
    ScriptNotFound(PathBuf),

    #[error("Timeout exceeds maximum allowed: {0}ms > {1}ms")]
    TimeoutTooLarge(u64, u64),

    #[error("Invalid working directory: {0}")]
    InvalidWorkingDir(PathBuf),

    #[error("Missing required field: {0}")]
    MissingField(String),

    #[error("Invalid execution plan: {0}")]
    InvalidPlan(String),
}
```

---

## Metadata Types

### ExecutionMetadata

Metadata for tracking and debugging.

```rust
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct ExecutionMetadata {
    pub source: Option<String>,
    pub conversation_id: Option<Uuid>,
    pub tags: HashMap<String, String>,
}
```

### Fields

- **`source`**: Source of request (e.g., "server", "ui", "test")
- **`conversation_id`**: Related conversation ID
- **`tags`**: Custom tags

---

### ExecutionSummary

Summary information about an execution.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionSummary {
    pub id: Uuid,
    pub status: ExecutionStatus,
    pub started_at: DateTime<Utc>,
    pub duration: Option<Duration>,
}
```

---

### ExecutionStats

Statistics for plan execution.

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

---

## Type Aliases

```rust
pub type Result<T> = std::result::Result<T, ExecutionError>;
```

---

## Related Documents

- [Architecture](architecture.md)
- [Types Reference](types.md)
- [Usage Examples](usage.md)
- [Event Handlers](event-handlers.md)