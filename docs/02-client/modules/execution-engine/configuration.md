# Configuration Guide

**Crate**: `cloudops-execution-engine`

Complete configuration reference for the Execution Engine.

---

## ExecutionConfig

Main configuration structure for the execution engine.

```rust
pub struct ExecutionConfig {
    /// Default timeout in milliseconds
    pub default_timeout_ms: u64,

    /// Maximum allowed timeout in milliseconds
    pub max_timeout_ms: u64,

    /// Enable real-time output streaming
    pub stream_output: bool,

    /// Custom log directory (None = system temp dir)
    pub log_dir: Option<PathBuf>,

    /// Maximum concurrent executions
    pub max_concurrent_executions: usize,
}
```

---

## Default Configuration

```rust
impl Default for ExecutionConfig {
    fn default() -> Self {
        Self {
            default_timeout_ms: 120_000,        // 2 minutes
            max_timeout_ms: 600_000,            // 10 minutes
            stream_output: true,
            log_dir: None,                      // System temp dir
            max_concurrent_executions: 10,
        }
    }
}
```

**Usage:**
```rust
let config = ExecutionConfig::default();
let engine = ExecutionEngine::new(config);
```

---

## Configuration Options

### Timeout Settings

#### `default_timeout_ms`

Default timeout for commands that don't specify a timeout.

```rust
let config = ExecutionConfig {
    default_timeout_ms: 300_000, // 5 minutes
    ..Default::default()
};
```

#### `max_timeout_ms`

Maximum allowed timeout. Commands exceeding this will be rejected.

```rust
let config = ExecutionConfig {
    max_timeout_ms: 1_800_000, // 30 minutes
    ..Default::default()
};
```

### Output Streaming

#### `stream_output`

Enable/disable real-time output streaming via events.

```rust
let config = ExecutionConfig {
    stream_output: true,  // Enable streaming
    ..Default::default()
};
```

**When to disable:**
- Not using event handlers
- Performance-sensitive scenarios
- Only care about final result

### Log Directory

#### `log_dir`

Custom directory for execution logs.

```rust
let config = ExecutionConfig {
    log_dir: Some(PathBuf::from("/var/log/cloudops")),
    ..Default::default()
};
```

**Default behavior (None):**
- Uses system temp directory
- Path: `/tmp/cloudops-executions/` on Unix
- Path: `C:\Users\{user}\AppData\Local\Temp\cloudops-executions\` on Windows

### Concurrency

#### `max_concurrent_executions`

Maximum number of parallel executions.

```rust
let config = ExecutionConfig {
    max_concurrent_executions: 20,
    ..Default::default()
};
```

---

## Configuration Examples

### Production Configuration

```rust
let config = ExecutionConfig {
    default_timeout_ms: 300_000,        // 5 minutes
    max_timeout_ms: 1_800_000,          // 30 minutes
    stream_output: true,
    log_dir: Some(PathBuf::from("/var/log/cloudops")),
    max_concurrent_executions: 50,
};
```

### Development Configuration

```rust
let config = ExecutionConfig {
    default_timeout_ms: 60_000,         // 1 minute
    max_timeout_ms: 300_000,            // 5 minutes
    stream_output: true,
    log_dir: None,                      // Use temp dir
    max_concurrent_executions: 5,
};
```

### Testing Configuration

```rust
let config = ExecutionConfig {
    default_timeout_ms: 10_000,         // 10 seconds
    max_timeout_ms: 30_000,             // 30 seconds
    stream_output: false,               // Disable streaming
    log_dir: Some(PathBuf::from("/tmp/test-logs")),
    max_concurrent_executions: 1,       // Serial only
};
```

---

## Environment-based Configuration

```rust
use std::env;

fn load_config_from_env() -> ExecutionConfig {
    ExecutionConfig {
        default_timeout_ms: env::var("EXEC_DEFAULT_TIMEOUT_MS")
            .ok()
            .and_then(|s| s.parse().ok())
            .unwrap_or(120_000),

        max_timeout_ms: env::var("EXEC_MAX_TIMEOUT_MS")
            .ok()
            .and_then(|s| s.parse().ok())
            .unwrap_or(600_000),

        stream_output: env::var("EXEC_STREAM_OUTPUT")
            .ok()
            .and_then(|s| s.parse().ok())
            .unwrap_or(true),

        log_dir: env::var("EXEC_LOG_DIR")
            .ok()
            .map(PathBuf::from),

        max_concurrent_executions: env::var("EXEC_MAX_CONCURRENT")
            .ok()
            .and_then(|s| s.parse().ok())
            .unwrap_or(10),
    }
}
```

---

## Configuration Files

### TOML Configuration

```toml
# config/execution.toml
[execution]
default_timeout_ms = 300000
max_timeout_ms = 1800000
stream_output = true
log_dir = "/var/log/cloudops"
max_concurrent_executions = 50
```

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct ConfigFile {
    execution: ExecutionConfig,
}

fn load_config_from_file(path: &str) -> Result<ExecutionConfig, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string(path)?;
    let config_file: ConfigFile = toml::from_str(&content)?;
    Ok(config_file.execution)
}
```

### JSON Configuration

```json
{
  "default_timeout_ms": 300000,
  "max_timeout_ms": 1800000,
  "stream_output": true,
  "log_dir": "/var/log/cloudops",
  "max_concurrent_executions": 50
}
```

```rust
fn load_config_from_json(path: &str) -> Result<ExecutionConfig, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string(path)?;
    let config: ExecutionConfig = serde_json::from_str(&content)?;
    Ok(config)
}
```

---

## Best Practices

### 1. Set Appropriate Timeouts

```rust
// Too short - may timeout frequently
default_timeout_ms: 10_000  // 10 seconds

// Recommended for AWS operations
default_timeout_ms: 120_000  // 2 minutes

// For long-running operations
default_timeout_ms: 600_000  // 10 minutes
```

### 2. Tune Concurrency

```rust
// Low concurrency for resource-constrained systems
max_concurrent_executions: 5

// Medium concurrency for typical workloads
max_concurrent_executions: 10-20

// High concurrency for powerful systems
max_concurrent_executions: 50-100
```

### 3. Use Structured Logs

```rust
let config = ExecutionConfig {
    log_dir: Some(PathBuf::from("/var/log/cloudops")),
    ..Default::default()
};

// Logs will be written to:
// /var/log/cloudops/{execution_id}.log
```

### 4. Disable Streaming When Not Needed

```rust
// If not using event handlers, disable streaming
let config = ExecutionConfig {
    stream_output: false,
    ..Default::default()
};
```

---

## Validation

Configuration is validated on engine creation:

```rust
let config = ExecutionConfig {
    default_timeout_ms: 700_000,  // Exceeds max
    max_timeout_ms: 600_000,
    ..Default::default()
};

// This will panic or return error
let engine = ExecutionEngine::new(config);
```

---

## Related Documents

- [API Reference](api.md)
- [Usage Examples](usage.md)
- [Error Handling](error-handling.md)