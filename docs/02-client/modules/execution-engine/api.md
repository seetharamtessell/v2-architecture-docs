# Execution Engine API Reference

**Crate**: `cloudops-execution-engine`

## Core Types

### ExecutionPlan

Represents a complete execution plan from the server.

```rust
pub struct ExecutionPlan {
    /// Unique identifier for this execution plan
    pub id: Uuid,

    /// Human-readable description
    pub description: String,

    /// Commands to execute
    pub commands: Vec<Command>,

    /// Execution mode (sequential, parallel, dependency-based)
    pub mode: ExecutionMode,

    /// Risk assessment
    pub risk_level: RiskLevel,

    /// Optional rollback strategy
    pub rollback_strategy: Option<RollbackStrategy>,

    /// Estimated execution time
    pub estimated_duration: Option<Duration>,
}

pub enum ExecutionMode {
    Sequential,
    Parallel,
    DependencyGraph { dependencies: HashMap<Uuid, Vec<Uuid>> },
}

pub enum RiskLevel {
    ReadOnly,
    Modify,
    Destructive,
}
```

### Command

Represents a single executable command.

```rust
pub struct Command {
    /// Unique identifier
    pub id: Uuid,

    /// Command type (CLI or SDK)
    pub command_type: CommandType,

    /// Human-readable description
    pub description: String,

    /// Risk level for this specific command
    pub risk_level: RiskLevel,

    /// AWS profile to use
    pub profile: String,

    /// AWS region
    pub region: String,
}

pub enum CommandType {
    Cli(CliCommand),
    Sdk(SdkCommand),
}

pub struct CliCommand {
    /// AWS CLI command (e.g., "aws ec2 describe-instances")
    pub command: String,

    /// Arguments
    pub args: Vec<String>,

    /// Support dry-run?
    pub supports_dry_run: bool,
}

pub struct SdkCommand {
    /// Service name (e.g., "ec2", "s3")
    pub service: String,

    /// Operation name (e.g., "DescribeInstances")
    pub operation: String,

    /// Serialized parameters (JSON)
    pub parameters: serde_json::Value,
}
```

### ExecutionResult

Result of executing a plan.

```rust
pub struct ExecutionResult {
    /// Execution plan ID
    pub plan_id: Uuid,

    /// Overall status
    pub status: ExecutionStatus,

    /// Results for each command
    pub command_results: Vec<CommandResult>,

    /// Total execution time
    pub duration: Duration,

    /// Error if failed
    pub error: Option<ExecutionError>,
}

pub enum ExecutionStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Cancelled,
    RolledBack,
}

pub struct CommandResult {
    pub command_id: Uuid,
    pub status: CommandStatus,
    pub output: String,
    pub error: Option<String>,
    pub duration: Duration,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}

pub enum CommandStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Skipped,
    RolledBack,
}
```

## Main API

### ExecutionEngine

The main entry point for the execution engine.

```rust
pub struct ExecutionEngine {
    storage: Arc<StorageService>,
    config: ExecutionConfig,
}

impl ExecutionEngine {
    /// Create a new execution engine
    pub fn new(storage: Arc<StorageService>, config: ExecutionConfig) -> Self {
        Self { storage, config }
    }

    /// Execute a plan with approval
    /// Returns execution ID for tracking
    pub async fn execute_with_approval(
        &self,
        plan: ExecutionPlan,
        approval: ApprovalDecision,
    ) -> Result<Uuid> {
        // Implementation
    }

    /// Execute a plan without approval (for non-destructive operations)
    pub async fn execute(
        &self,
        plan: ExecutionPlan,
    ) -> Result<ExecutionResult> {
        // Implementation
    }

    /// Execute in dry-run mode
    pub async fn dry_run(
        &self,
        plan: ExecutionPlan,
    ) -> Result<DryRunResult> {
        // Implementation
    }

    /// Get execution status
    pub async fn get_status(&self, execution_id: Uuid) -> Result<ExecutionStatus> {
        // Implementation
    }

    /// Get execution result
    pub async fn get_result(&self, execution_id: Uuid) -> Result<ExecutionResult> {
        // Implementation
    }

    /// Cancel a running execution
    pub async fn cancel(&self, execution_id: Uuid) -> Result<()> {
        // Implementation
    }

    /// Rollback a completed execution
    pub async fn rollback(&self, execution_id: Uuid) -> Result<ExecutionResult> {
        // Implementation
    }

    /// List execution history
    pub async fn list_executions(
        &self,
        filter: ExecutionFilter,
    ) -> Result<Vec<ExecutionRecord>> {
        // Implementation
    }

    /// Search execution history
    pub async fn search_executions(
        &self,
        query: &str,
    ) -> Result<Vec<ExecutionRecord>> {
        // Implementation
    }
}
```

### ExecutionConfig

Configuration for the execution engine.

```rust
pub struct ExecutionConfig {
    /// Default executor type
    pub default_executor: ExecutorType,

    /// Retry configuration
    pub retry_config: RetryConfig,

    /// Timeout for commands
    pub command_timeout: Duration,

    /// Enable output streaming
    pub stream_output: bool,

    /// Store execution history
    pub store_history: bool,
}

pub enum ExecutorType {
    Cli,
    Sdk,
    Auto, // Choose based on command
}

impl Default for ExecutionConfig {
    fn default() -> Self {
        Self {
            default_executor: ExecutorType::Auto,
            retry_config: RetryConfig::default(),
            command_timeout: Duration::from_secs(300), // 5 minutes
            stream_output: true,
            store_history: true,
        }
    }
}
```

## Approval Workflow

### ApprovalWorkflow

Manages the approval process.

```rust
pub struct ApprovalWorkflow {
    config: ApprovalConfig,
}

impl ApprovalWorkflow {
    /// Create new approval workflow
    pub fn new(config: ApprovalConfig) -> Self {
        Self { config }
    }

    /// Assess risk level of a plan
    pub fn assess_risk(&self, plan: &ExecutionPlan) -> RiskLevel {
        // Implementation
    }

    /// Check if approval is required
    pub fn requires_approval(&self, plan: &ExecutionPlan) -> bool {
        // Implementation
    }

    /// Create approval request
    pub fn create_approval_request(
        &self,
        plan: &ExecutionPlan,
    ) -> ApprovalRequest {
        // Implementation
    }

    /// Process approval decision
    pub fn process_decision(
        &self,
        request: &ApprovalRequest,
        decision: ApprovalDecision,
    ) -> Result<ProcessedApproval> {
        // Implementation
    }
}

pub struct ApprovalRequest {
    pub plan_id: Uuid,
    pub description: String,
    pub risk_level: RiskLevel,
    pub commands: Vec<CommandSummary>,
    pub estimated_duration: Option<Duration>,
    pub created_at: DateTime<Utc>,
}

pub struct CommandSummary {
    pub id: Uuid,
    pub description: String,
    pub risk_level: RiskLevel,
    pub dry_run_result: Option<String>,
}

pub enum ApprovalDecision {
    Approve,
    Reject { reason: String },
    ModifyAndApprove { modified_plan: ExecutionPlan },
}

pub struct ProcessedApproval {
    pub approved: bool,
    pub plan: ExecutionPlan,
    pub metadata: ApprovalMetadata,
}

pub struct ApprovalMetadata {
    pub approved_by: String, // User identifier
    pub approved_at: DateTime<Utc>,
    pub decision: ApprovalDecision,
    pub notes: Option<String>,
}
```

### ApprovalConfig

Configuration for approval workflow.

```rust
pub struct ApprovalConfig {
    /// Auto-approve read-only operations
    pub auto_approve_readonly: bool,

    /// Require approval for modify operations
    pub require_approval_modify: bool,

    /// Require approval for destructive operations
    pub require_approval_destructive: bool,

    /// Enable dry-run before approval
    pub dry_run_before_approval: bool,
}

impl Default for ApprovalConfig {
    fn default() -> Self {
        Self {
            auto_approve_readonly: false,
            require_approval_modify: true,
            require_approval_destructive: true,
            dry_run_before_approval: true,
        }
    }
}
```

## Executors

### Executor Trait

Base trait for all executors.

```rust
#[async_trait]
pub trait Executor: Send + Sync {
    /// Execute a command
    async fn execute(&self, command: &Command) -> Result<CommandOutput>;

    /// Execute in dry-run mode
    async fn dry_run(&self, command: &Command) -> Result<DryRunOutput>;

    /// Check if this executor supports the command
    fn supports(&self, command: &Command) -> bool;

    /// Get executor type
    fn executor_type(&self) -> ExecutorType;
}

pub struct CommandOutput {
    pub stdout: String,
    pub stderr: String,
    pub exit_code: i32,
    pub duration: Duration,
}

pub struct DryRunOutput {
    pub description: String,
    pub changes: Vec<String>,
    pub warnings: Vec<String>,
}
```

### CliExecutor

Executes commands via AWS CLI.

```rust
pub struct CliExecutor {
    config: CliExecutorConfig,
}

impl CliExecutor {
    pub fn new(config: CliExecutorConfig) -> Self {
        Self { config }
    }

    /// Execute AWS CLI command
    pub async fn execute_cli(
        &self,
        command: &CliCommand,
        profile: &str,
        region: &str,
    ) -> Result<CommandOutput> {
        // Implementation
    }

    /// Stream output line-by-line
    pub async fn execute_streaming<F>(
        &self,
        command: &CliCommand,
        profile: &str,
        region: &str,
        callback: F,
    ) -> Result<CommandOutput>
    where
        F: Fn(String) + Send + 'static,
    {
        // Implementation
    }
}

pub struct CliExecutorConfig {
    /// Path to AWS CLI binary
    pub cli_path: PathBuf,

    /// Timeout for commands
    pub timeout: Duration,

    /// Output format (json, text, table)
    pub output_format: String,
}

impl Default for CliExecutorConfig {
    fn default() -> Self {
        Self {
            cli_path: PathBuf::from("aws"),
            timeout: Duration::from_secs(300),
            output_format: "json".to_string(),
        }
    }
}
```

### SdkExecutor

Executes commands via AWS SDK for Rust.

```rust
pub struct SdkExecutor {
    clients: HashMap<String, Box<dyn Any + Send + Sync>>,
}

impl SdkExecutor {
    pub fn new() -> Self {
        Self {
            clients: HashMap::new(),
        }
    }

    /// Execute SDK command
    pub async fn execute_sdk(
        &self,
        command: &SdkCommand,
        profile: &str,
        region: &str,
    ) -> Result<CommandOutput> {
        // Implementation
    }

    /// Get or create SDK client for service
    async fn get_client<T>(
        &mut self,
        service: &str,
        profile: &str,
        region: &str,
    ) -> Result<&T>
    where
        T: 'static + Send + Sync,
    {
        // Implementation
    }
}
```

## History Management

### ExecutionHistory

Manages execution history.

```rust
pub struct ExecutionHistory {
    storage: Arc<StorageService>,
}

impl ExecutionHistory {
    pub fn new(storage: Arc<StorageService>) -> Self {
        Self { storage }
    }

    /// Store execution record
    pub async fn store(
        &self,
        record: ExecutionRecord,
    ) -> Result<()> {
        // Implementation
    }

    /// Get execution record
    pub async fn get(
        &self,
        execution_id: Uuid,
    ) -> Result<ExecutionRecord> {
        // Implementation
    }

    /// List executions with filter
    pub async fn list(
        &self,
        filter: ExecutionFilter,
    ) -> Result<Vec<ExecutionRecord>> {
        // Implementation
    }

    /// Search executions
    pub async fn search(
        &self,
        query: &str,
    ) -> Result<Vec<ExecutionRecord>> {
        // Implementation
    }

    /// Get executions for conversation
    pub async fn get_by_conversation(
        &self,
        conversation_id: Uuid,
    ) -> Result<Vec<ExecutionRecord>> {
        // Implementation
    }

    /// Delete old executions
    pub async fn cleanup(
        &self,
        older_than: DateTime<Utc>,
    ) -> Result<usize> {
        // Implementation
    }
}

pub struct ExecutionRecord {
    pub id: Uuid,
    pub conversation_id: Uuid,
    pub timestamp: DateTime<Utc>,
    pub user_prompt: String,
    pub execution_plan: ExecutionPlan,
    pub approval_metadata: Option<ApprovalMetadata>,
    pub result: ExecutionResult,
    pub duration: Duration,
}

pub struct ExecutionFilter {
    pub status: Option<ExecutionStatus>,
    pub risk_level: Option<RiskLevel>,
    pub conversation_id: Option<Uuid>,
    pub from_date: Option<DateTime<Utc>>,
    pub to_date: Option<DateTime<Utc>>,
    pub limit: usize,
    pub offset: usize,
}
```

## Error Handling

### Error Types

```rust
#[derive(Debug, thiserror::Error)]
pub enum ExecutionError {
    #[error("Command execution failed: {0}")]
    CommandFailed(String),

    #[error("Approval required but not provided")]
    ApprovalRequired,

    #[error("Execution timeout after {0:?}")]
    Timeout(Duration),

    #[error("Execution cancelled by user")]
    Cancelled,

    #[error("Rollback failed: {0}")]
    RollbackFailed(String),

    #[error("Invalid command: {0}")]
    InvalidCommand(String),

    #[error("Executor not available: {0}")]
    ExecutorNotAvailable(String),

    #[error("Storage error: {0}")]
    Storage(#[from] StorageError),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("AWS SDK error: {0}")]
    AwsSdk(String),

    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

pub type Result<T> = std::result::Result<T, ExecutionError>;
```

### Retry Configuration

```rust
pub struct RetryConfig {
    pub max_attempts: u32,
    pub backoff: BackoffStrategy,
    pub retryable_errors: Vec<String>,
}

pub enum BackoffStrategy {
    Fixed(Duration),
    Exponential { base: Duration, max: Duration },
    Linear { increment: Duration, max: Duration },
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_attempts: 3,
            backoff: BackoffStrategy::Exponential {
                base: Duration::from_secs(1),
                max: Duration::from_secs(30),
            },
            retryable_errors: vec![
                "ThrottlingException".to_string(),
                "RequestLimitExceeded".to_string(),
                "ServiceUnavailable".to_string(),
            ],
        }
    }
}
```

## Event System

### Events

The execution engine emits events for real-time updates.

```rust
pub enum ExecutionEvent {
    /// Execution started
    Started {
        execution_id: Uuid,
        plan: ExecutionPlan,
    },

    /// Command started
    CommandStarted {
        execution_id: Uuid,
        command_id: Uuid,
    },

    /// Command output (streaming)
    CommandOutput {
        execution_id: Uuid,
        command_id: Uuid,
        line: String,
    },

    /// Command completed
    CommandCompleted {
        execution_id: Uuid,
        command_id: Uuid,
        result: CommandResult,
    },

    /// Command failed
    CommandFailed {
        execution_id: Uuid,
        command_id: Uuid,
        error: String,
    },

    /// Execution completed
    Completed {
        execution_id: Uuid,
        result: ExecutionResult,
    },

    /// Execution failed
    Failed {
        execution_id: Uuid,
        error: ExecutionError,
    },

    /// Execution cancelled
    Cancelled {
        execution_id: Uuid,
    },

    /// Rollback started
    RollbackStarted {
        execution_id: Uuid,
    },

    /// Rollback completed
    RollbackCompleted {
        execution_id: Uuid,
        result: ExecutionResult,
    },
}

/// Event listener trait
#[async_trait]
pub trait ExecutionEventListener: Send + Sync {
    async fn on_event(&self, event: ExecutionEvent);
}
```

## Usage Examples

### Basic Execution

```rust
use cloudops_execution_engine::{ExecutionEngine, ExecutionPlan, ExecutionConfig};

#[tokio::main]
async fn main() -> Result<()> {
    let storage = Arc::new(StorageService::new(storage_config).await?);
    let engine = ExecutionEngine::new(storage, ExecutionConfig::default());

    let plan = ExecutionPlan {
        id: Uuid::new_v4(),
        description: "List EC2 instances".to_string(),
        commands: vec![/* ... */],
        mode: ExecutionMode::Sequential,
        risk_level: RiskLevel::ReadOnly,
        rollback_strategy: None,
        estimated_duration: None,
    };

    let result = engine.execute(plan).await?;
    println!("Execution completed: {:?}", result);

    Ok(())
}
```

### Execution with Approval

```rust
use cloudops_execution_engine::{ApprovalWorkflow, ApprovalDecision};

let workflow = ApprovalWorkflow::new(ApprovalConfig::default());
let approval_request = workflow.create_approval_request(&plan);

// Present to user and get decision
let decision = get_user_decision(&approval_request).await?;

let execution_id = engine.execute_with_approval(plan, decision).await?;
println!("Execution started: {}", execution_id);
```

### Streaming Output

```rust
use tauri::Manager;

let app_handle = app.handle();
let execution_id = Uuid::new_v4();

// Listen for events
app_handle.listen_global("execution:output", move |event| {
    println!("Output: {:?}", event.payload());
});

// Execute with streaming
engine.execute(plan).await?;
```

## Tauri Integration

### Tauri Commands

```rust
use tauri::command;

#[command]
pub async fn execute_plan(
    state: tauri::State<'_, Arc<ExecutionEngine>>,
    plan: ExecutionPlan,
) -> Result<Uuid, String> {
    state.execute(plan)
        .await
        .map_err(|e| e.to_string())
}

#[command]
pub async fn get_execution_status(
    state: tauri::State<'_, Arc<ExecutionEngine>>,
    execution_id: Uuid,
) -> Result<ExecutionStatus, String> {
    state.get_status(execution_id)
        .await
        .map_err(|e| e.to_string())
}

#[command]
pub async fn cancel_execution(
    state: tauri::State<'_, Arc<ExecutionEngine>>,
    execution_id: Uuid,
) -> Result<(), String> {
    state.cancel(execution_id)
        .await
        .map_err(|e| e.to_string())
}

#[command]
pub async fn list_execution_history(
    state: tauri::State<'_, Arc<ExecutionEngine>>,
    filter: ExecutionFilter,
) -> Result<Vec<ExecutionRecord>, String> {
    state.list_executions(filter)
        .await
        .map_err(|e| e.to_string())
}
```

## Related Documents

- [Architecture](architecture.md)
- [Execution Strategies](execution-strategies.md)
- [Approval Workflow](approval-workflow.md)