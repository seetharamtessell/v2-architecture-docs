# Error Handling Guide

**Crate**: `cloudops-execution-engine`

Complete guide for handling errors in the Execution Engine.

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

pub type Result<T> = std::result::Result<T, ExecutionError>;
```

### ValidationError

Validation-specific errors.

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

## Error Handling Patterns

### Basic Error Handling

```rust
async fn execute_command(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> Result<String, String> {
    match engine.execute(request).await {
        Ok(execution_id) => {
            // Wait for completion
            let result = wait_and_get_result(&engine, execution_id).await?;

            if result.exit_code == 0 {
                Ok(result.stdout)
            } else {
                Err(format!("Command failed: {}", result.stderr))
            }
        }
        Err(e) => Err(format!("Execution error: {}", e)),
    }
}
```

### Match on Error Variants

```rust
async fn execute_with_specific_handling(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> Result<ExecutionResult, String> {
    match engine.execute(request).await {
        Ok(execution_id) => {
            match wait_for_completion(&engine, execution_id).await {
                Ok(_) => engine.get_result(execution_id).await
                    .map_err(|e| e.to_string()),
                Err(ExecutionError::Timeout(ms)) => {
                    Err(format!("Execution timed out after {}ms", ms))
                }
                Err(ExecutionError::Cancelled) => {
                    Err("Execution was cancelled by user".to_string())
                }
                Err(e) => Err(format!("Execution failed: {}", e)),
            }
        }
        Err(ExecutionError::Validation(e)) => {
            Err(format!("Invalid request: {}", e))
        }
        Err(ExecutionError::NotFound(id)) => {
            Err(format!("Execution {} not found", id))
        }
        Err(e) => Err(format!("Failed to start execution: {}", e)),
    }
}
```

### Using Result Extensions

```rust
use anyhow::Context;

async fn execute_with_context(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> anyhow::Result<String> {
    let execution_id = engine.execute(request).await
        .context("Failed to start command execution")?;

    wait_for_completion(&engine, execution_id).await
        .context("Command execution failed")?;

    let result = engine.get_result(execution_id).await
        .context("Failed to get execution result")?;

    if result.exit_code != 0 {
        anyhow::bail!(
            "Command failed with exit code {}: {}",
            result.exit_code,
            result.stderr
        );
    }

    Ok(result.stdout)
}
```

---

## Retry Strategies

### Simple Retry

```rust
async fn execute_with_retry(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
    max_attempts: u32,
) -> Result<ExecutionResult, ExecutionError> {
    let mut attempt = 0;

    loop {
        attempt += 1;

        match engine.execute(request.clone()).await {
            Ok(execution_id) => {
                match wait_for_completion(&engine, execution_id).await {
                    Ok(_) => {
                        let result = engine.get_result(execution_id).await?;
                        if result.exit_code == 0 {
                            return Ok(result);
                        }

                        if attempt >= max_attempts {
                            return Ok(result); // Return failed result
                        }
                    }
                    Err(e) if attempt < max_attempts => {
                        // Retry on error
                        eprintln!("Attempt {} failed: {}", attempt, e);
                    }
                    Err(e) => return Err(e),
                }
            }
            Err(e) if attempt < max_attempts => {
                eprintln!("Attempt {} failed: {}", attempt, e);
            }
            Err(e) => return Err(e),
        }

        // Exponential backoff
        let delay = Duration::from_millis(100 * 2_u64.pow(attempt - 1));
        tokio::time::sleep(delay).await;
    }
}
```

### Retry with Specific Error Conditions

```rust
async fn execute_with_conditional_retry(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> Result<ExecutionResult, ExecutionError> {
    let max_attempts = 3;
    let mut attempt = 0;

    loop {
        attempt += 1;

        let execution_id = engine.execute(request.clone()).await?;

        match wait_for_completion(&engine, execution_id).await {
            Ok(_) => return engine.get_result(execution_id).await,

            // Retry on timeout
            Err(ExecutionError::Timeout(_)) if attempt < max_attempts => {
                eprintln!("Timeout, retrying... (attempt {})", attempt);
                tokio::time::sleep(Duration::from_secs(2)).await;
                continue;
            }

            // Don't retry on cancellation
            Err(ExecutionError::Cancelled) => {
                return Err(ExecutionError::Cancelled);
            }

            // Retry on other errors
            Err(e) if attempt < max_attempts => {
                eprintln!("Error: {}, retrying... (attempt {})", e, attempt);
                tokio::time::sleep(Duration::from_secs(2)).await;
                continue;
            }

            Err(e) => return Err(e),
        }
    }
}
```

---

## Validation Errors

### Handle Validation Errors

```rust
async fn execute_with_validation(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
    config: &ExecutionConfig,
) -> Result<ExecutionResult, String> {
    // Validate before executing
    if let Err(e) = request.validate(config) {
        match e {
            ValidationError::ScriptNotFound(path) => {
                return Err(format!("Script not found: {:?}", path));
            }
            ValidationError::TimeoutTooLarge(requested, max) => {
                return Err(format!(
                    "Timeout {}ms exceeds maximum {}ms",
                    requested, max
                ));
            }
            ValidationError::InvalidWorkingDir(dir) => {
                return Err(format!("Invalid working directory: {:?}", dir));
            }
            e => return Err(format!("Validation error: {}", e)),
        }
    }

    // Execute
    let execution_id = engine.execute(request).await
        .map_err(|e| e.to_string())?;

    wait_for_completion(&engine, execution_id).await
        .map_err(|e| e.to_string())?;

    engine.get_result(execution_id).await
        .map_err(|e| e.to_string())
}
```

---

## Timeout Handling

### Handle Timeouts Gracefully

```rust
async fn execute_with_timeout_handling(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> Result<ExecutionResult, String> {
    let execution_id = engine.execute(request).await
        .map_err(|e| e.to_string())?;

    match wait_for_completion(&engine, execution_id).await {
        Ok(_) => {
            engine.get_result(execution_id).await
                .map_err(|e| e.to_string())
        }
        Err(ExecutionError::Timeout(ms)) => {
            eprintln!("Execution timed out after {}ms", ms);

            // Try to get partial result
            if let Ok(result) = engine.get_result(execution_id).await {
                eprintln!("Partial output: {}", result.stdout);
            }

            Err(format!("Execution timed out after {}ms", ms))
        }
        Err(e) => Err(e.to_string()),
    }
}
```

### Increase Timeout for Long Operations

```rust
async fn execute_long_running_command(
    engine: &ExecutionEngine,
) -> Result<ExecutionResult, ExecutionError> {
    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::Script {
            path: PathBuf::from("/opt/scripts/long-task.sh"),
            interpreter: None,
        },
        env: HashMap::new(),
        working_dir: None,
        timeout_ms: Some(1_800_000), // 30 minutes
        metadata: Default::default(),
    };

    let execution_id = engine.execute(request).await?;
    wait_for_completion(&engine, execution_id).await?;
    engine.get_result(execution_id).await
}
```

---

## Cancellation Handling

### Handle User Cancellation

```rust
async fn execute_with_cancel_support(
    engine: Arc<ExecutionEngine>,
    request: ExecutionRequest,
    mut cancel_rx: tokio::sync::oneshot::Receiver<()>,
) -> Result<ExecutionResult, String> {
    let execution_id = engine.execute(request).await
        .map_err(|e| e.to_string())?;

    tokio::select! {
        // Wait for completion
        result = wait_for_completion(&engine, execution_id) => {
            result.map_err(|e| e.to_string())?;
            engine.get_result(execution_id).await
                .map_err(|e| e.to_string())
        }

        // Handle cancellation
        _ = &mut cancel_rx => {
            eprintln!("Cancelling execution...");

            engine.cancel(execution_id).await
                .map_err(|e| e.to_string())?;

            Err("Execution cancelled by user".to_string())
        }
    }
}
```

---

## Error Recovery

### Fallback to Alternative Command

```rust
async fn execute_with_fallback(
    engine: &ExecutionEngine,
    primary_request: ExecutionRequest,
    fallback_request: ExecutionRequest,
) -> Result<ExecutionResult, ExecutionError> {
    match engine.execute(primary_request).await {
        Ok(execution_id) => {
            match wait_for_completion(&engine, execution_id).await {
                Ok(_) => engine.get_result(execution_id).await,
                Err(_) => {
                    eprintln!("Primary command failed, trying fallback...");
                    let fallback_id = engine.execute(fallback_request).await?;
                    wait_for_completion(&engine, fallback_id).await?;
                    engine.get_result(fallback_id).await
                }
            }
        }
        Err(_) => {
            eprintln!("Primary command failed to start, trying fallback...");
            let fallback_id = engine.execute(fallback_request).await?;
            wait_for_completion(&engine, fallback_id).await?;
            engine.get_result(fallback_id).await
        }
    }
}
```

---

## Logging Errors

### Log Errors for Debugging

```rust
use tracing::{error, warn, info};

async fn execute_with_logging(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> Result<ExecutionResult, ExecutionError> {
    info!("Starting execution for command: {:?}", request.command);

    let execution_id = match engine.execute(request).await {
        Ok(id) => {
            info!("Execution started with ID: {}", id);
            id
        }
        Err(e) => {
            error!("Failed to start execution: {}", e);
            return Err(e);
        }
    };

    match wait_for_completion(&engine, execution_id).await {
        Ok(_) => {
            info!("Execution {} completed", execution_id);
        }
        Err(ExecutionError::Timeout(ms)) => {
            warn!("Execution {} timed out after {}ms", execution_id, ms);
            return Err(ExecutionError::Timeout(ms));
        }
        Err(e) => {
            error!("Execution {} failed: {}", execution_id, e);
            return Err(e);
        }
    }

    let result = engine.get_result(execution_id).await?;

    if result.exit_code != 0 {
        error!(
            "Command failed with exit code {}: {}",
            result.exit_code, result.stderr
        );
    } else {
        info!("Command completed successfully");
    }

    Ok(result)
}
```

---

## Error Conversion

### Convert to Custom Error Types

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Execution failed: {0}")]
    Execution(#[from] ExecutionError),

    #[error("Validation failed: {0}")]
    Validation(String),

    #[error("Timeout: {0}")]
    Timeout(String),

    #[error("Command failed with exit code {0}")]
    CommandFailed(i32),
}

impl From<ValidationError> for AppError {
    fn from(e: ValidationError) -> Self {
        AppError::Validation(e.to_string())
    }
}

async fn execute_app_command(
    engine: &ExecutionEngine,
    request: ExecutionRequest,
) -> Result<String, AppError> {
    let execution_id = engine.execute(request).await?;

    wait_for_completion(&engine, execution_id).await?;

    let result = engine.get_result(execution_id).await?;

    if result.exit_code != 0 {
        return Err(AppError::CommandFailed(result.exit_code));
    }

    Ok(result.stdout)
}
```

---

## Best Practices

### 1. Always Check Exit Codes

```rust
let result = engine.get_result(execution_id).await?;

if result.exit_code != 0 {
    eprintln!("Command failed: {}", result.stderr);
    // Handle failure
}
```

### 2. Provide Context

```rust
engine.execute(request).await
    .context("Failed to execute deployment script")?;
```

### 3. Log Errors

```rust
if let Err(e) = engine.execute(request).await {
    error!("Execution failed: {}", e);
    // Handle error
}
```

### 4. Handle Timeouts

```rust
let request = ExecutionRequest {
    timeout_ms: Some(300_000), // Set reasonable timeout
    // ...
};
```

### 5. Clean Up on Error

```rust
match engine.execute(request).await {
    Ok(execution_id) => {
        // Use execution
    }
    Err(e) => {
        // Clean up resources
        cleanup().await;
        return Err(e);
    }
}
```

---

## Related Documents

- [API Reference](api.md)
- [Usage Examples](usage.md)
- [Configuration](configuration.md)