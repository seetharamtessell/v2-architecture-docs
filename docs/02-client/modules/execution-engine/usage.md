# Execution Engine Usage Guide

**Crate**: `cloudops-execution-engine`

Practical examples and usage patterns for the Execution Engine.

---

## Table of Contents

- [Basic Usage](#basic-usage)
- [Single Command Execution](#single-command-execution)
- [Multiple Commands](#multiple-commands)
- [Event Streaming](#event-streaming)
- [Error Handling](#error-handling)
- [Advanced Patterns](#advanced-patterns)
- [Integration Examples](#integration-examples)

---

## Basic Usage

### Minimal Example

```rust
use cloudops_execution_engine::{
    ExecutionEngine, ExecutionRequest, Command, ExecutionConfig, ExecutionStatus,
};
use std::collections::HashMap;
use uuid::Uuid;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create engine
    let engine = ExecutionEngine::new(ExecutionConfig::default());

    // Create request
    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::Shell {
            command: "echo 'Hello, World!'".to_string(),
            shell: "bash".to_string(),
        },
        env: HashMap::new(),
        working_dir: None,
        timeout_ms: None,
        metadata: Default::default(),
    };

    // Execute
    let execution_id = engine.execute(request).await?;

    // Wait for completion
    loop {
        let status = engine.get_status(execution_id).await?;
        if status == ExecutionStatus::Completed || status == ExecutionStatus::Failed {
            break;
        }
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    }

    // Get result
    let result = engine.get_result(execution_id).await?;
    println!("Output: {}", result.stdout);
    println!("Exit code: {}", result.exit_code);

    Ok(())
}
```

---

## Single Command Execution

### AWS CLI Command

```rust
use cloudops_execution_engine::{ExecutionRequest, Command};

async fn describe_ec2_instances(engine: &ExecutionEngine) -> Result<String, ExecutionError> {
    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::AwsCli {
            service: "ec2".to_string(),
            operation: "describe-instances".to_string(),
            args: vec![
                "--filters".to_string(),
                "Name=instance-state-name,Values=running".to_string(),
            ],
            profile: Some("production".to_string()),
            region: Some("us-west-2".to_string()),
        },
        env: HashMap::new(),
        working_dir: None,
        timeout_ms: Some(60_000), // 1 minute
        metadata: Default::default(),
    };

    let execution_id = engine.execute(request).await?;

    // Wait for completion
    wait_for_completion(&engine, execution_id).await?;

    let result = engine.get_result(execution_id).await?;

    if result.exit_code == 0 {
        Ok(result.stdout)
    } else {
        Err(ExecutionError::CommandFailed(result.exit_code))
    }
}

// Helper function to wait for completion
async fn wait_for_completion(
    engine: &ExecutionEngine,
    execution_id: Uuid,
) -> Result<(), ExecutionError> {
    loop {
        let status = engine.get_status(execution_id).await?;
        match status {
            ExecutionStatus::Completed => return Ok(()),
            ExecutionStatus::Failed | ExecutionStatus::Timeout | ExecutionStatus::Cancelled => {
                let result = engine.get_result(execution_id).await?;
                return Err(ExecutionError::CommandFailed(result.exit_code));
            }
            _ => {
                tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
            }
        }
    }
}
```

### Execute Script File

```rust
async fn run_deployment_script(
    engine: &ExecutionEngine,
    script_path: &str,
) -> Result<ExecutionResult, ExecutionError> {
    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::Script {
            path: PathBuf::from(script_path),
            interpreter: None, // Auto-detect from shebang
        },
        env: HashMap::from([
            ("ENVIRONMENT".to_string(), "production".to_string()),
            ("REGION".to_string(), "us-west-2".to_string()),
        ]),
        working_dir: Some(PathBuf::from("/opt/deployments")),
        timeout_ms: Some(600_000), // 10 minutes
        metadata: ExecutionMetadata {
            source: Some("deployment-system".to_string()),
            conversation_id: None,
            tags: HashMap::from([
                ("type".to_string(), "deployment".to_string()),
            ]),
        },
    };

    let execution_id = engine.execute(request).await?;
    wait_for_completion(&engine, execution_id).await?;
    engine.get_result(execution_id).await
}
```

### Execute Shell Command

```rust
async fn check_disk_space(
    engine: &ExecutionEngine,
) -> Result<String, ExecutionError> {
    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::Shell {
            command: "df -h | grep '/dev/sda1'".to_string(),
            shell: "bash".to_string(),
        },
        env: HashMap::new(),
        working_dir: None,
        timeout_ms: Some(5_000),
        metadata: Default::default(),
    };

    let execution_id = engine.execute(request).await?;
    wait_for_completion(&engine, execution_id).await?;

    let result = engine.get_result(execution_id).await?;
    Ok(result.stdout)
}
```

### Execute Program with Arguments

```rust
async fn analyze_logs(
    engine: &ExecutionEngine,
    log_file: &str,
) -> Result<String, ExecutionError> {
    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::Exec {
            program: "python3".to_string(),
            args: vec![
                "/opt/scripts/analyze.py".to_string(),
                "--input".to_string(),
                log_file.to_string(),
                "--format".to_string(),
                "json".to_string(),
            ],
        },
        env: HashMap::from([
            ("PYTHONPATH".to_string(), "/opt/lib".to_string()),
        ]),
        working_dir: Some(PathBuf::from("/opt/scripts")),
        timeout_ms: Some(300_000), // 5 minutes
        metadata: Default::default(),
    };

    let execution_id = engine.execute(request).await?;
    wait_for_completion(&engine, execution_id).await?;

    let result = engine.get_result(execution_id).await?;
    Ok(result.stdout)
}
```

---

## Multiple Commands

### Serial Execution

Execute commands one by one, stopping on first error.

```rust
async fn deploy_application(
    engine: &ExecutionEngine,
) -> Result<PlanExecutionResult, ExecutionError> {
    let plan = ExecutionPlan {
        id: Uuid::new_v4(),
        description: "Deploy application to production".to_string(),
        strategy: ExecutionStrategy::Serial {
            stop_on_error: true,
        },
        commands: vec![
            // Step 1: Pre-deployment checks
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::Script {
                    path: PathBuf::from("/opt/scripts/pre-deploy-check.sh"),
                    interpreter: None,
                },
                env: HashMap::new(),
                working_dir: None,
                timeout_ms: Some(60_000),
                metadata: Default::default(),
            },
            // Step 2: Pull latest code
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::Shell {
                    command: "git pull origin main".to_string(),
                    shell: "bash".to_string(),
                },
                env: HashMap::new(),
                working_dir: Some(PathBuf::from("/opt/app")),
                timeout_ms: Some(30_000),
                metadata: Default::default(),
            },
            // Step 3: Build application
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::Exec {
                    program: "cargo".to_string(),
                    args: vec!["build".to_string(), "--release".to_string()],
                },
                env: HashMap::new(),
                working_dir: Some(PathBuf::from("/opt/app")),
                timeout_ms: Some(600_000), // 10 minutes
                metadata: Default::default(),
            },
            // Step 4: Update ECS service
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::AwsCli {
                    service: "ecs".to_string(),
                    operation: "update-service".to_string(),
                    args: vec![
                        "--cluster".to_string(),
                        "production".to_string(),
                        "--service".to_string(),
                        "api".to_string(),
                        "--force-new-deployment".to_string(),
                    ],
                    profile: Some("production".to_string()),
                    region: Some("us-west-2".to_string()),
                },
                env: HashMap::new(),
                working_dir: None,
                timeout_ms: Some(300_000),
                metadata: Default::default(),
            },
            // Step 5: Health check
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::Shell {
                    command: "curl -f https://api.example.com/health".to_string(),
                    shell: "bash".to_string(),
                },
                env: HashMap::new(),
                working_dir: None,
                timeout_ms: Some(10_000),
                metadata: Default::default(),
            },
        ],
        metadata: ExecutionMetadata {
            source: Some("deployment-pipeline".to_string()),
            conversation_id: None,
            tags: HashMap::from([
                ("environment".to_string(), "production".to_string()),
            ]),
        },
    };

    let plan_id = engine.execute_plan(plan).await?;
    wait_for_completion(&engine, plan_id).await?;

    // Get plan result (would need to be implemented)
    // For now, return empty result
    Ok(PlanExecutionResult {
        plan_id,
        status: ExecutionStatus::Completed,
        results: vec![],
        total_duration: Duration::from_secs(0),
        stats: ExecutionStats {
            total: 5,
            completed: 5,
            failed: 0,
            cancelled: 0,
            timeout: 0,
        },
    })
}
```

### Parallel Execution

Execute multiple commands concurrently.

```rust
async fn scan_all_regions(
    engine: &ExecutionEngine,
) -> Result<PlanExecutionResult, ExecutionError> {
    let regions = vec!["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"];

    let commands: Vec<ExecutionRequest> = regions
        .into_iter()
        .map(|region| ExecutionRequest {
            id: Uuid::new_v4(),
            command: Command::AwsCli {
                service: "ec2".to_string(),
                operation: "describe-instances".to_string(),
                args: vec![],
                profile: Some("production".to_string()),
                region: Some(region.to_string()),
            },
            env: HashMap::new(),
            working_dir: None,
            timeout_ms: Some(60_000),
            metadata: ExecutionMetadata {
                tags: HashMap::from([
                    ("region".to_string(), region.to_string()),
                ]),
                ..Default::default()
            },
        })
        .collect();

    let plan = ExecutionPlan {
        id: Uuid::new_v4(),
        description: "Scan EC2 instances in all regions".to_string(),
        strategy: ExecutionStrategy::Parallel {
            max_concurrency: Some(4), // Max 4 concurrent scans
        },
        commands,
        metadata: Default::default(),
    };

    let plan_id = engine.execute_plan(plan).await?;
    wait_for_completion(&engine, plan_id).await?;

    // Return result
    Ok(PlanExecutionResult {
        plan_id,
        status: ExecutionStatus::Completed,
        results: vec![],
        total_duration: Duration::from_secs(0),
        stats: ExecutionStats {
            total: 4,
            completed: 4,
            failed: 0,
            cancelled: 0,
            timeout: 0,
        },
    })
}
```

### Continue on Error

Execute all commands even if some fail.

```rust
async fn cleanup_resources(
    engine: &ExecutionEngine,
) -> Result<PlanExecutionResult, ExecutionError> {
    let plan = ExecutionPlan {
        id: Uuid::new_v4(),
        description: "Cleanup temporary resources".to_string(),
        strategy: ExecutionStrategy::Serial {
            stop_on_error: false, // Continue even if some fail
        },
        commands: vec![
            // Cleanup temp files
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::Shell {
                    command: "rm -rf /tmp/app-*".to_string(),
                    shell: "bash".to_string(),
                },
                env: HashMap::new(),
                working_dir: None,
                timeout_ms: Some(10_000),
                metadata: Default::default(),
            },
            // Stop unused containers
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::Shell {
                    command: "docker ps -a | grep 'Exited' | awk '{print $1}' | xargs docker rm".to_string(),
                    shell: "bash".to_string(),
                },
                env: HashMap::new(),
                working_dir: None,
                timeout_ms: Some(30_000),
                metadata: Default::default(),
            },
            // Clear old logs
            ExecutionRequest {
                id: Uuid::new_v4(),
                command: Command::Shell {
                    command: "find /var/log -name '*.log' -mtime +7 -delete".to_string(),
                    shell: "bash".to_string(),
                },
                env: HashMap::new(),
                working_dir: None,
                timeout_ms: Some(30_000),
                metadata: Default::default(),
            },
        ],
        metadata: Default::default(),
    };

    let plan_id = engine.execute_plan(plan).await?;
    wait_for_completion(&engine, plan_id).await?;

    Ok(PlanExecutionResult {
        plan_id,
        status: ExecutionStatus::Completed,
        results: vec![],
        total_duration: Duration::from_secs(0),
        stats: ExecutionStats {
            total: 3,
            completed: 2,
            failed: 1,
            cancelled: 0,
            timeout: 0,
        },
    })
}
```

---

## Event Streaming

### Basic Event Handler

```rust
use cloudops_execution_engine::{EventHandler, ExecutionEvent};
use async_trait::async_trait;

struct ConsoleEventHandler;

#[async_trait]
impl EventHandler for ConsoleEventHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match event {
            ExecutionEvent::Started { execution_id, command, timestamp } => {
                println!("[{}] Started: {}", execution_id, command);
            }
            ExecutionEvent::Stdout { execution_id, line, .. } => {
                println!("[{}] {}", execution_id, line);
            }
            ExecutionEvent::Stderr { execution_id, line, .. } => {
                eprintln!("[{}] ERROR: {}", execution_id, line);
            }
            ExecutionEvent::Completed { execution_id, result, .. } => {
                println!("[{}] Completed with exit code {}",
                    execution_id, result.exit_code);
            }
            ExecutionEvent::Failed { execution_id, error, .. } => {
                eprintln!("[{}] Failed: {}", execution_id, error);
            }
            ExecutionEvent::Cancelled { execution_id, .. } => {
                println!("[{}] Cancelled", execution_id);
            }
            ExecutionEvent::Progress { plan_id, completed, total, current_command, .. } => {
                println!("[{}] Progress: {}/{} - {:?}",
                    plan_id, completed, total, current_command);
            }
        }
    }
}

// Use with engine
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = ExecutionEngine::new(ExecutionConfig::default())
        .with_event_handler(Arc::new(ConsoleEventHandler));

    // Now all executions will emit events to console
    let request = /* ... */;
    let execution_id = engine.execute(request).await?;

    Ok(())
}
```

### File Logging Event Handler

```rust
use tokio::sync::Mutex;
use tokio::fs::OpenOptions;
use tokio::io::AsyncWriteExt;

struct FileLoggerEventHandler {
    log_file: Arc<Mutex<tokio::fs::File>>,
}

impl FileLoggerEventHandler {
    async fn new(log_path: &str) -> Result<Self, std::io::Error> {
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(log_path)
            .await?;

        Ok(Self {
            log_file: Arc::new(Mutex::new(file)),
        })
    }
}

#[async_trait]
impl EventHandler for FileLoggerEventHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        let log_line = match event {
            ExecutionEvent::Stdout { execution_id, line, timestamp } => {
                format!("[{}] [{}] {}\n", timestamp, execution_id, line)
            }
            ExecutionEvent::Stderr { execution_id, line, timestamp } => {
                format!("[{}] [{}] ERROR: {}\n", timestamp, execution_id, line)
            }
            _ => return,
        };

        let mut file = self.log_file.lock().await;
        file.write_all(log_line.as_bytes()).await.ok();
    }
}
```

---

## Error Handling

### Handle All Error Types

```rust
async fn execute_with_error_handling(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> Result<String, String> {
    match engine.execute(request).await {
        Ok(execution_id) => {
            // Wait for completion
            match wait_for_completion(&engine, execution_id).await {
                Ok(_) => {
                    // Get result
                    match engine.get_result(execution_id).await {
                        Ok(result) => {
                            if result.exit_code == 0 {
                                Ok(result.stdout)
                            } else {
                                Err(format!("Command failed with exit code {}: {}",
                                    result.exit_code, result.stderr))
                            }
                        }
                        Err(e) => Err(format!("Failed to get result: {}", e)),
                    }
                }
                Err(e) => Err(format!("Execution failed: {}", e)),
            }
        }
        Err(ExecutionError::Validation(e)) => {
            Err(format!("Invalid request: {}", e))
        }
        Err(ExecutionError::Timeout(ms)) => {
            Err(format!("Execution timed out after {}ms", ms))
        }
        Err(e) => Err(format!("Execution error: {}", e)),
    }
}
```

### Retry Logic

```rust
async fn execute_with_retry(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
    max_retries: u32,
) -> Result<ExecutionResult, ExecutionError> {
    let mut attempts = 0;

    loop {
        attempts += 1;

        let execution_id = engine.execute(request.clone()).await?;

        match wait_for_completion(&engine, execution_id).await {
            Ok(_) => {
                let result = engine.get_result(execution_id).await?;

                if result.exit_code == 0 {
                    return Ok(result);
                }

                if attempts >= max_retries {
                    return Ok(result); // Return failed result
                }

                // Wait before retry
                tokio::time::sleep(tokio::time::Duration::from_secs(2_u64.pow(attempts))).await;
            }
            Err(ExecutionError::Timeout(_)) if attempts < max_retries => {
                // Retry on timeout
                continue;
            }
            Err(e) => return Err(e),
        }
    }
}
```

---

## Advanced Patterns

### Cancellation

```rust
async fn execute_with_cancellation(
    engine: Arc<ExecutionEngine>,
    request: ExecutionRequest,
    cancel_signal: tokio::sync::oneshot::Receiver<()>,
) -> Result<ExecutionResult, String> {
    let execution_id = engine.execute(request).await
        .map_err(|e| e.to_string())?;

    tokio::select! {
        _ = cancel_signal => {
            // Cancel execution
            engine.cancel(execution_id).await
                .map_err(|e| e.to_string())?;
            Err("Execution cancelled by user".to_string())
        }
        result = wait_for_completion(&engine, execution_id) => {
            result.map_err(|e| e.to_string())?;
            engine.get_result(execution_id).await
                .map_err(|e| e.to_string())
        }
    }
}

// Usage
let (cancel_tx, cancel_rx) = tokio::sync::oneshot::channel();

// Spawn execution
let handle = tokio::spawn(async move {
    execute_with_cancellation(engine, request, cancel_rx).await
});

// Cancel after 5 seconds
tokio::time::sleep(tokio::time::Duration::from_secs(5)).await;
cancel_tx.send(()).ok();
```

### Progress Tracking

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

struct ProgressTracker {
    completed: Arc<AtomicUsize>,
    total: usize,
}

impl ProgressTracker {
    fn new(total: usize) -> Self {
        Self {
            completed: Arc::new(AtomicUsize::new(0)),
            total,
        }
    }

    fn increment(&self) {
        self.completed.fetch_add(1, Ordering::SeqCst);
    }

    fn progress(&self) -> f32 {
        let completed = self.completed.load(Ordering::SeqCst);
        (completed as f32 / self.total as f32) * 100.0
    }
}

async fn execute_with_progress(
    engine: &ExecutionEngine,
    requests: Vec<ExecutionRequest>,
) -> Result<Vec<ExecutionResult>, ExecutionError> {
    let tracker = Arc::new(ProgressTracker::new(requests.len()));
    let mut results = Vec::new();

    for request in requests {
        let execution_id = engine.execute(request).await?;
        wait_for_completion(&engine, execution_id).await?;

        let result = engine.get_result(execution_id).await?;
        results.push(result);

        tracker.increment();
        println!("Progress: {:.1}%", tracker.progress());
    }

    Ok(results)
}
```

---

## Integration Examples

See separate example files:
- [Tauri Integration](examples/tauri-integration.md)
- [Server Integration](examples/server-integration.md)
- [CLI Tool](examples/cli-tool.md)

---

## Related Documents

- [API Reference](api.md)
- [Event Handlers](event-handlers.md)
- [Error Handling](error-handling.md)
- [Configuration](configuration.md)