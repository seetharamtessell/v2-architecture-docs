# Execution Engine Architecture

**Crate Name**: `cloudops-execution-engine`
**Status**: Design Phase 🔄

## Overview

The Execution Engine is responsible for executing AWS operations with user approval, managing execution history, and providing real-time feedback. It acts as the bridge between the LLM-generated commands and actual AWS infrastructure changes.

## Design Principles

1. **Safety First**: All destructive operations require explicit user approval
2. **Transparency**: Real-time output streaming and complete audit trail
3. **Flexibility**: Support both AWS CLI and AWS SDK for Rust
4. **Reliability**: Robust error handling, retries, and rollback capabilities
5. **Auditability**: Complete execution history with context

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React)                         │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│              Execution Engine (Tauri Command)               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          Approval Workflow Manager                    │  │
│  │  • Parse execution plan                               │  │
│  │  • Risk assessment                                    │  │
│  │  • Present to user for approval                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          Execution Orchestrator                       │  │
│  │  • Execute commands sequentially                      │  │
│  │  • Handle dependencies                                │  │
│  │  • Stream output                                      │  │
│  │  • Error handling & rollback                          │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────┬────────────────────┐                  │
│  │   CLI Executor   │   SDK Executor     │                  │
│  │  • Spawn process │  • Native calls    │                  │
│  │  • Parse output  │  • Type safety     │                  │
│  │  • Handle stderr │  • Better errors   │                  │
│  └──────────────────┴────────────────────┘                  │
│                            ↓                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          Execution History Manager                    │  │
│  │  • Store execution records                            │  │
│  │  • Link to chat context                               │  │
│  │  • Provide audit trail                                │  │
│  │  • Enable replay/rollback                             │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│              Storage Service (Qdrant)                       │
│  • Execution history                                        │
│  • Linked chat context                                      │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Approval Workflow Manager

**Purpose**: Present execution plans to users and collect approval decisions.

**Responsibilities**:
- Parse server response (execution plan with commands)
- Assess risk level (read-only, modify, destructive)
- Present formatted plan to user
- Collect approval/rejection/modification
- Track approval metadata (who, when, what)

**Risk Levels**:
```rust
pub enum RiskLevel {
    ReadOnly,      // List, describe, get operations
    Modify,        // Update, modify operations
    Destructive,   // Delete, terminate, destroy operations
}
```

**Approval Flow**:
```
Server Response → Parse Plan → Assess Risk → Present to User
                                                    ↓
                                            [Approve/Reject/Edit]
                                                    ↓
                                    Approved → Execute
                                    Rejected → Cancel & Store Decision
                                    Edit → Modify & Re-approve
```

### 2. Execution Orchestrator

**Purpose**: Coordinate execution of commands with dependencies, error handling, and rollback.

**Responsibilities**:
- Execute commands sequentially or in parallel (based on dependencies)
- Stream output in real-time to frontend
- Handle errors and retries
- Coordinate rollback on failure
- Track execution state

**Execution Modes**:
```rust
pub enum ExecutionMode {
    Sequential,           // Execute one by one
    Parallel,             // Execute concurrently (independent commands)
    DependencyGraph,      // Execute based on dependency DAG
}
```

**State Machine**:
```
Pending → Running → [Completed | Failed | Cancelled]
                         ↓           ↓
                    [Success]   [Rollback?]
```

### 3. CLI Executor

**Purpose**: Execute AWS CLI commands via subprocess.

**Responsibilities**:
- Spawn AWS CLI process with credentials from `~/.aws/credentials`
- Stream stdout/stderr in real-time
- Parse CLI output (JSON or text)
- Handle CLI errors
- Support dry-run mode

**Advantages**:
- Simple to implement
- Supports all AWS services
- Easy debugging (copy-paste commands)

**Disadvantages**:
- Slower (process spawn overhead)
- Output parsing required
- Less type-safe

### 4. SDK Executor

**Purpose**: Execute AWS operations using AWS SDK for Rust.

**Responsibilities**:
- Call AWS SDK directly (no subprocess)
- Use native Rust types
- Handle SDK errors with better context
- Support credential providers
- Enable better retry logic

**Advantages**:
- Faster (no process spawn)
- Type-safe
- Better error messages
- More control over retries

**Disadvantages**:
- Requires SDK support for service
- More complex implementation
- Debugging harder

### 5. Execution History Manager

**Purpose**: Store and retrieve execution history with complete audit trail.

**Responsibilities**:
- Store execution records with metadata
- Link executions to chat context (conversation ID)
- Provide search/filter over history
- Enable replay of past executions
- Support rollback operations

**Storage Schema**:
```rust
pub struct ExecutionRecord {
    pub id: Uuid,
    pub conversation_id: Uuid,        // Link to chat
    pub timestamp: DateTime<Utc>,
    pub user_prompt: String,          // Original user request
    pub execution_plan: ExecutionPlan, // What was approved
    pub approval_metadata: ApprovalMetadata,
    pub commands: Vec<CommandRecord>,
    pub status: ExecutionStatus,
    pub duration_ms: u64,
}

pub struct CommandRecord {
    pub id: Uuid,
    pub command: String,
    pub executor_type: ExecutorType,  // CLI or SDK
    pub status: CommandStatus,
    pub output: String,
    pub error: Option<String>,
    pub duration_ms: u64,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
}
```

## Execution Flow

### Happy Path
```
1. User approves execution plan
2. Orchestrator receives plan
3. For each command:
   a. Select executor (CLI or SDK)
   b. Execute command
   c. Stream output to frontend
   d. Store result
4. Mark execution as completed
5. Store complete execution record
6. Link to chat context
```

### Error Path
```
1. Command fails during execution
2. Capture error details
3. Decide: retry or rollback?
4. If rollback:
   a. Execute rollback commands (if provided)
   b. Mark execution as failed + rolled back
5. Store execution record with error
6. Notify user
```

## Credential Management

**Approach**: Use AWS credentials from `~/.aws/credentials` (OS file system).

**For CLI Executor**:
```rust
// Set AWS_PROFILE environment variable
std::env::set_var("AWS_PROFILE", profile_name);
// Spawn aws cli process (inherits env)
```

**For SDK Executor**:
```rust
// Load credentials from profile
let config = aws_config::from_env()
    .profile_name(profile_name)
    .load()
    .await;
```

**Security**:
- Never store credentials in application storage
- Never send credentials to server
- Use OS file permissions for `~/.aws/credentials`
- Support MFA/SSO workflows

## Output Streaming

**Real-time Updates**:
```rust
// Frontend listens to Tauri events
tauri.event.listen("execution:output", (event) => {
  const { command_id, line } = event.payload;
  appendOutput(command_id, line);
});

// Backend emits events
async fn execute_cli_command(app_handle: &AppHandle, cmd: &Command) {
    let mut child = std::process::Command::new("aws")
        .args(&cmd.args)
        .stdout(Stdio::piped())
        .spawn()?;

    let stdout = child.stdout.take().unwrap();
    let reader = BufReader::new(stdout);

    for line in reader.lines() {
        let line = line?;
        app_handle.emit_all("execution:output", json!({
            "command_id": cmd.id,
            "line": line
        }))?;
    }
}
```

## Error Handling & Retries

**Retry Strategy**:
```rust
pub struct RetryConfig {
    pub max_attempts: u32,
    pub backoff: BackoffStrategy,
    pub retryable_errors: Vec<String>, // "ThrottlingException", etc.
}

pub enum BackoffStrategy {
    Fixed(Duration),
    Exponential { base: Duration, max: Duration },
    Linear { increment: Duration, max: Duration },
}
```

**Error Categories**:
1. **Retryable**: Throttling, network errors, temporary failures
2. **Non-retryable**: Permission denied, resource not found, invalid parameters
3. **Rollback-required**: Partial success in multi-command execution

## Rollback Support

**Rollback Strategies**:
```rust
pub enum RollbackStrategy {
    None,                              // No rollback (read-only)
    CommandBased(Vec<Command>),        // Execute rollback commands
    StateBased(Box<dyn StateRestorer>), // Restore previous state
}
```

**Example**:
```rust
// Original execution
ExecutionPlan {
    commands: [
        Command::Sdk(ec2::terminate_instances(instance_ids)),
    ],
    rollback_strategy: RollbackStrategy::CommandBased(vec![
        Command::Sdk(ec2::start_instances(instance_ids)),
    ]),
}
```

## Dry-Run Support

**Purpose**: Preview changes without executing.

**Implementation**:
```rust
pub enum ExecutionMode {
    DryRun,  // Show what would happen
    Execute, // Actually execute
}

// Many AWS CLI commands support --dry-run
aws ec2 run-instances --dry-run ...

// SDK also supports DryRun
let req = ec2::RunInstancesRequest {
    dry_run: Some(true),
    ...
};
```

## Testing Strategy

**Unit Tests**:
- Test approval workflow logic
- Test risk assessment
- Test command parsing
- Test retry logic

**Integration Tests**:
- Test CLI executor with AWS CLI
- Test SDK executor with AWS SDK
- Test output streaming
- Test error handling

**Mocking**:
```rust
// Mock executor for testing
pub struct MockExecutor {
    responses: HashMap<String, Result<String, String>>,
}

impl Executor for MockExecutor {
    async fn execute(&self, cmd: &Command) -> Result<Output> {
        // Return predefined response
    }
}
```

## Performance Considerations

1. **Async Execution**: Use tokio for concurrent operations
2. **Streaming**: Stream output instead of buffering
3. **Connection Pooling**: Reuse SDK clients
4. **Caching**: Cache SDK clients per profile

## Security Considerations

1. **Command Injection**: Sanitize all inputs (even from LLM)
2. **Credential Isolation**: Per-profile execution
3. **Audit Trail**: Complete execution history
4. **Approval Required**: No auto-execution of destructive operations

## Future Enhancements

1. **Execution Templates**: Reusable execution patterns
2. **Scheduled Execution**: Execute at specific time
3. **Conditional Execution**: Execute if condition met
4. **Parallel Execution**: Concurrent command execution
5. **Progress Estimation**: Show estimated completion time
6. **Resource Locking**: Prevent concurrent modifications

## Dependencies

```toml
[dependencies]
# Core
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = { version = "0.4", features = ["serde"] }

# AWS SDK
aws-config = "1"
aws-sdk-ec2 = "1"
aws-sdk-s3 = "1"
# ... other AWS services

# CLI execution
tokio-process = "0.2"

# Storage
cloudops-storage-service = { path = "../storage-service" }

# Error handling
anyhow = "1"
thiserror = "1"
```

## Related Documents

- [API Reference](api.md)
- [Execution Strategies](execution-strategies.md)
- [Approval Workflow](approval-workflow.md)
- [Storage Service](../storage-service/README.md)