# Execution Engine Architecture

**Crate**: `cloudops-execution-engine`
**Pattern**: Tokio + Streaming (inspired by Claude Code)

## Design Principles

1. **Pure Rust Crate** - No framework dependencies (Tauri/Axum optional via traits)
2. **Async-First** - Built on Tokio for non-blocking execution
3. **Reusable** - Can be used in clients, servers, CLI tools
4. **Type-Safe** - Strongly typed with serde for serialization
5. **Event-Driven** - Pluggable event system for real-time updates
6. **Resilient** - Timeout handling, cancellation, error recovery
7. **Observable** - Complete logging and execution history

---

## Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                     User Application                          │
│            (Tauri Client / Server / CLI Tool)                 │
└───────────────────────────────────────────────────────────────┘
                            ↓
                 ┌─────────────────────┐
                 │  ExecutionEngine    │
                 │  (Public API)       │
                 └─────────────────────┘
                            ↓
        ┌───────────────────┴───────────────────┐
        ↓                                       ↓
┌─────────────────┐                  ┌─────────────────┐
│  State Manager  │                  │ Event Emitter   │
│  (In-Memory)    │                  │  (Pluggable)    │
└─────────────────┘                  └─────────────────┘
        ↓                                       ↓
┌─────────────────────────────────────────────────────┐
│              Execution Orchestrator                 │
│  • Validate request                                 │
│  • Spawn background task                            │
│  • Track execution state                            │
└─────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────┐
│                Command Executor                     │
│  • tokio::process::Command                          │
│  • Stream stdout/stderr (BufReader + AsyncBufRead)  │
│  • Handle timeouts (tokio::time::timeout)           │
│  • Cancellation support (CancellationToken)         │
└─────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────┐
│             OS Process (bash/aws/sh)                │
└─────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────┐
│          Temp Folder Logs (/tmp/cloudops-*)         │
└─────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. ExecutionEngine (Public API)

**Purpose**: Main entry point for all execution operations.

**Responsibilities**:
- Provide high-level API for command execution
- Validate input requests
- Delegate to internal components
- Return execution IDs for tracking

**Key Methods**:
```rust
pub struct ExecutionEngine {
    config: ExecutionConfig,
    executions: Arc<RwLock<HashMap<Uuid, ExecutionState>>>,
    event_handler: Option<Arc<dyn EventHandler>>,
}

impl ExecutionEngine {
    pub fn new(config: ExecutionConfig) -> Self;
    pub fn with_event_handler(self, handler: Arc<dyn EventHandler>) -> Self;

    pub async fn execute(&self, request: ExecutionRequest) -> Result<Uuid>;
    pub async fn execute_plan(&self, plan: ExecutionPlan) -> Result<Uuid>;
    pub async fn get_status(&self, id: Uuid) -> Result<ExecutionStatus>;
    pub async fn get_result(&self, id: Uuid) -> Result<ExecutionResult>;
    pub async fn cancel(&self, id: Uuid) -> Result<()>;
    pub async fn read_logs(&self, id: Uuid) -> Result<String>;
    pub async fn list_executions(&self) -> Vec<ExecutionSummary>;
}
```

**Design Pattern**: Facade pattern - provides simple interface over complex subsystem.

---

### 2. State Manager

**Purpose**: Track execution state in memory.

**Data Structure**:
```rust
type ExecutionStore = Arc<RwLock<HashMap<Uuid, ExecutionState>>>;

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

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ExecutionStatus {
    Pending,
    Running,
    Completed,
    Failed,
    Cancelled,
    Timeout,
}
```

**Concurrency**: Uses `Arc<RwLock<>>` for thread-safe shared state.

**State Transitions**:
```
Pending → Running → [Completed | Failed | Cancelled | Timeout]
```

**Lifetime**: Executions remain in memory until process restart. Logs persist in temp folder.

---

### 3. Event System

**Purpose**: Provide real-time updates about execution progress.

**Event Handler Trait**:
```rust
#[async_trait::async_trait]
pub trait EventHandler: Send + Sync {
    async fn on_event(&self, event: ExecutionEvent);
}

#[derive(Debug, Clone, Serialize, Deserialize)]
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

**Design Pattern**: Observer pattern - decouple execution from event consumption.

**Flexibility**: Users implement `EventHandler` trait for custom behavior (Tauri events, WebSocket, logging, etc.).

---

### 4. Execution Orchestrator

**Purpose**: Coordinate execution of single commands or plans.

**Responsibilities**:
- Validate execution requests
- Create execution state
- Spawn background tasks
- Handle serial/parallel execution strategies
- Update state on completion

**Single Command Flow**:
```
1. Validate ExecutionRequest
2. Generate execution_id
3. Create ExecutionState (Pending)
4. Store in state manager
5. Spawn tokio background task
6. Return execution_id immediately
7. Background task:
   a. Update status to Running
   b. Execute command
   c. Stream output
   d. Update status to Completed/Failed
   e. Store result
```

**Plan Execution Flow**:
```
Serial:
  For each command:
    - Execute
    - Wait for completion
    - If error and stop_on_error: break
    - Otherwise: continue

Parallel:
  Spawn tokio task for each command
  tokio::spawn for all
  Wait for all tasks to complete (join_all)
  Aggregate results

Dependency Graph:
  Build DAG from dependencies
  Execute in topological order
  Wait for dependencies before starting
```

---

### 5. Command Executor (Tokio)

**Purpose**: Execute individual commands using Tokio async processes.

**Key Implementation**:
```rust
async fn execute_command_internal(
    &self,
    execution_id: Uuid,
    request: ExecutionRequest,
    cancel_token: CancellationToken,
) -> Result<ExecutionResult> {
    let start = Instant::now();

    // Build tokio::process::Command
    let mut cmd = self.build_command(&request)?;

    // Spawn process with piped I/O
    let mut child = cmd
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()?;

    let stdout = child.stdout.take().unwrap();
    let stderr = child.stderr.take().unwrap();

    // Stream stdout in background task
    let stdout_task = self.stream_output(
        execution_id,
        stdout,
        OutputType::Stdout,
    );

    // Stream stderr in background task
    let stderr_task = self.stream_output(
        execution_id,
        stderr,
        OutputType::Stderr,
    );

    // Wait with timeout and cancellation
    let status = self.wait_with_timeout_and_cancel(
        child,
        request.timeout_ms,
        cancel_token,
    ).await?;

    // Join streaming tasks
    let stdout_output = stdout_task.await?;
    let stderr_output = stderr_task.await?;

    Ok(ExecutionResult {
        execution_id,
        exit_code: status.code().unwrap_or(-1),
        success: status.success(),
        stdout: stdout_output,
        stderr: stderr_output,
        duration: start.elapsed(),
        log_path: self.get_log_path(execution_id),
        // ...
    })
}
```

**Streaming Pattern**:
```rust
async fn stream_output(
    &self,
    execution_id: Uuid,
    output: ChildStdout, // or ChildStderr
    output_type: OutputType,
) -> Result<String> {
    let reader = BufReader::new(output);
    let mut lines = reader.lines();
    let mut collected = Vec::new();

    while let Some(line) = lines.next_line().await? {
        // Write to log file
        self.write_log(execution_id, &line).await?;

        // Emit event
        if let Some(handler) = &self.event_handler {
            let event = match output_type {
                OutputType::Stdout => ExecutionEvent::Stdout {
                    execution_id,
                    line: line.clone(),
                    timestamp: Utc::now(),
                },
                OutputType::Stderr => ExecutionEvent::Stderr {
                    execution_id,
                    line: line.clone(),
                    timestamp: Utc::now(),
                },
            };
            handler.on_event(event).await;
        }

        collected.push(line);
    }

    Ok(collected.join("\n"))
}
```

**Timeout + Cancellation Pattern**:
```rust
use tokio::time::{timeout, Duration};
use tokio_util::sync::CancellationToken;

async fn wait_with_timeout_and_cancel(
    &self,
    mut child: Child,
    timeout_ms: Option<u64>,
    cancel_token: CancellationToken,
) -> Result<ExitStatus> {
    let timeout_duration = Duration::from_millis(
        timeout_ms.unwrap_or(self.config.default_timeout_ms)
    );

    tokio::select! {
        // Normal completion
        result = child.wait() => {
            result.map_err(Into::into)
        }

        // Timeout
        _ = tokio::time::sleep(timeout_duration) => {
            child.kill().await?;
            Err(ExecutionError::Timeout(timeout_duration.as_millis() as u64))
        }

        // Cancellation
        _ = cancel_token.cancelled() => {
            child.kill().await?;
            Err(ExecutionError::Cancelled)
        }
    }
}
```

---

### 6. Command Builder

**Purpose**: Convert `Command` enum to `tokio::process::Command`.

```rust
fn build_command(&self, request: &ExecutionRequest) -> Result<tokio::process::Command> {
    let mut cmd = match &request.command {
        Command::Script { path, interpreter } => {
            let mut c = if let Some(interp) = interpreter {
                tokio::process::Command::new(interp)
            } else {
                // Auto-detect from shebang or use bash
                tokio::process::Command::new("bash")
            };
            c.arg(path);
            c
        }

        Command::Exec { program, args } => {
            let mut c = tokio::process::Command::new(program);
            c.args(args);
            c
        }

        Command::Shell { command, shell } => {
            let mut c = tokio::process::Command::new(shell);
            c.arg("-c").arg(command);
            c
        }

        Command::AwsCli { service, operation, args, profile, region } => {
            let mut c = tokio::process::Command::new("aws");
            c.arg(service).arg(operation);
            c.args(args);

            if let Some(p) = profile {
                c.env("AWS_PROFILE", p);
            }
            if let Some(r) = region {
                c.env("AWS_REGION", r);
            }
            c.env("AWS_OUTPUT", "json");
            c
        }
    };

    // Set environment variables
    for (key, val) in &request.env {
        cmd.env(key, val);
    }

    // Set working directory
    if let Some(ref dir) = request.working_dir {
        cmd.current_dir(dir);
    }

    Ok(cmd)
}
```

---

### 7. Log Management

**Purpose**: Store execution logs in temp folder.

```rust
impl ExecutionEngine {
    fn get_log_path(&self, execution_id: Uuid) -> PathBuf {
        let temp_dir = std::env::temp_dir();
        let log_dir = temp_dir.join("cloudops-executions");

        // Create directory if needed
        std::fs::create_dir_all(&log_dir).ok();

        log_dir.join(format!("{}.log", execution_id))
    }

    async fn write_log(&self, execution_id: Uuid, line: &str) -> Result<()> {
        let log_path = self.get_log_path(execution_id);

        let mut file = tokio::fs::OpenOptions::new()
            .create(true)
            .append(true)
            .open(log_path)
            .await?;

        use tokio::io::AsyncWriteExt;
        file.write_all(format!("{}\n", line).as_bytes()).await?;

        Ok(())
    }

    pub async fn read_logs(&self, execution_id: Uuid) -> Result<String> {
        let log_path = self.get_log_path(execution_id);
        tokio::fs::read_to_string(log_path)
            .await
            .map_err(Into::into)
    }
}
```

**Log Cleanup**: OS automatically cleans temp folder periodically. Logs are ephemeral.

---

## Execution Patterns

### Pattern 1: Background Execution

```rust
pub async fn execute(&self, request: ExecutionRequest) -> Result<Uuid> {
    // Validate
    request.validate(&self.config)?;

    let execution_id = Uuid::new_v4();
    let cancel_token = CancellationToken::new();
    let log_path = self.get_log_path(execution_id);

    // Create initial state
    let state = ExecutionState {
        id: execution_id,
        status: ExecutionStatus::Pending,
        started_at: Utc::now(),
        completed_at: None,
        result: None,
        cancel_token: cancel_token.clone(),
        log_path,
    };

    // Store state
    self.executions.write().await.insert(execution_id, state);

    // Spawn background task
    let engine = self.clone();
    tokio::spawn(async move {
        engine.execute_internal(execution_id, request, cancel_token).await
    });

    Ok(execution_id)
}
```

### Pattern 2: Serial Execution

```rust
async fn execute_serial(
    &self,
    plan: ExecutionPlan,
    stop_on_error: bool,
) -> Result<PlanExecutionResult> {
    let mut results = Vec::new();

    for (index, request) in plan.commands.into_iter().enumerate() {
        // Emit progress event
        self.emit_progress_event(&plan, index, plan.commands.len());

        // Execute command
        let execution_id = self.execute(request).await?;

        // Wait for completion
        let result = self.wait_for_completion(execution_id).await;

        match result {
            Ok(res) => {
                results.push(res);
            }
            Err(e) => {
                results.push(create_error_result(execution_id, e));

                if stop_on_error {
                    break; // Stop on first error
                }
            }
        }
    }

    Ok(PlanExecutionResult {
        plan_id: plan.id,
        results,
        // ...
    })
}
```

### Pattern 3: Parallel Execution

```rust
async fn execute_parallel(
    &self,
    plan: ExecutionPlan,
    max_concurrency: Option<usize>,
) -> Result<PlanExecutionResult> {
    let semaphore = max_concurrency.map(|n| Arc::new(Semaphore::new(n)));

    let mut tasks = Vec::new();

    for request in plan.commands {
        let engine = self.clone();
        let sem = semaphore.clone();

        let task = tokio::spawn(async move {
            // Acquire semaphore if limited concurrency
            let _permit = if let Some(ref s) = sem {
                Some(s.acquire().await.unwrap())
            } else {
                None
            };

            let execution_id = engine.execute(request).await?;
            engine.wait_for_completion(execution_id).await
        });

        tasks.push(task);
    }

    // Wait for all to complete
    let results = futures::future::join_all(tasks).await;

    Ok(PlanExecutionResult {
        plan_id: plan.id,
        results: results.into_iter().map(|r| r.unwrap()).collect(),
        // ...
    })
}
```

---

## Concurrency Model

### Thread Safety
- **State**: `Arc<RwLock<HashMap>>` - Multiple readers, single writer
- **Event Handler**: `Arc<dyn EventHandler>` - Shared across tasks
- **Execution**: Each execution runs in separate Tokio task

### Tokio Runtime
- Requires Tokio runtime (`#[tokio::main]` or manual runtime)
- All async functions use `.await`
- Background tasks via `tokio::spawn`

### Resource Limits
- Configurable `max_concurrent_executions`
- Semaphore for parallel execution throttling
- OS limits on file descriptors and processes apply

---

## Error Handling Strategy

### Error Types
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
}
```

### Recovery Strategies
- **Timeout**: Kill process, return timeout error
- **Cancellation**: Kill process, update state to Cancelled
- **Command Failure**: Capture stderr, store in result
- **Spawn Failure**: Return error immediately
- **IO Error**: Propagate error to caller

---

## Performance Considerations

### Memory
- Execution state stored in memory (HashMap)
- Output streams buffered line-by-line (low memory)
- Log files stored on disk (not in memory)

### CPU
- Tokio async runtime uses thread pool
- Process spawning is OS-level overhead
- Minimal CPU usage in waiting/streaming

### I/O
- Async I/O via Tokio (non-blocking)
- Log writes are async (buffered)
- Process pipes handled by OS

### Scalability
- Can handle hundreds of concurrent executions
- Limited by OS file descriptors and process limits
- Configurable concurrency limits

---

## Security Considerations

1. **Command Injection**: Input validation, no shell interpolation
2. **Environment Variables**: User-controlled, validated
3. **File Paths**: Validated before use
4. **Process Isolation**: Each command runs in separate process
5. **Credentials**: Never logged (user responsibility to avoid in commands)
6. **Logs**: Stored in OS temp folder (ephemeral)

---

## Dependencies

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-util = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
async-trait = "0.1"
futures = "0.3"
```

---

## Related Documents

- [API Reference](api.md)
- [Types Reference](types.md)
- [Cargo Integration](cargo-integration.md)
- [Event Handlers](event-handlers.md)
- [Configuration](configuration.md)