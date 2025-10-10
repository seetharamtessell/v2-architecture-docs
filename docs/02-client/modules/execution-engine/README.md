# Execution Engine Module

**Crate Name**: `cloudops-execution-engine`
**Status**: Design Phase 🔄
**Type**: Pure Rust Crate (Reusable Library)

## Overview

The Execution Engine is a **standalone, reusable Rust crate** for executing commands and scripts with real-time streaming, background execution, and cancellation support. Built on Tokio for async, non-blocking execution.

**Like an npm package** - this crate can be used anywhere: Tauri clients, server microservices, CLI tools, or any Rust application.

## Key Features

✅ **Async/Non-blocking** - Built on Tokio for concurrent execution
✅ **Real-time Streaming** - Line-by-line stdout/stderr streaming
✅ **Background Execution** - Returns immediately with execution ID
✅ **Multiple Execution Modes** - Serial, parallel, dependency-based
✅ **Cancellation Support** - Kill running processes
✅ **Timeout Handling** - Configurable timeouts per command
✅ **Event System** - Pluggable event handlers (Tauri, WebSocket, etc.)
✅ **Standardized I/O** - Type-safe request/response formats
✅ **Log Management** - Automatic logging to temp folder
✅ **Pure Rust** - No UI framework dependencies

## Quick Start

### Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
cloudops-execution-engine = "0.1.0"
tokio = { version = "1", features = ["full"] }
```

Or use cargo:

```bash
cargo add cloudops-execution-engine
cargo add tokio --features full
```

### Basic Usage

```rust
use cloudops_execution_engine::{
    ExecutionEngine, ExecutionRequest, Command, ExecutionConfig,
};
use std::collections::HashMap;
use uuid::Uuid;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create engine
    let engine = ExecutionEngine::new(ExecutionConfig::default());

    // Create execution request
    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::AwsCli {
            service: "ec2".to_string(),
            operation: "describe-instances".to_string(),
            args: vec!["--region".to_string(), "us-west-2".to_string()],
            profile: Some("production".to_string()),
            region: Some("us-west-2".to_string()),
        },
        env: HashMap::new(),
        working_dir: None,
        timeout_ms: None,
        metadata: Default::default(),
    };

    // Execute (returns immediately)
    let execution_id = engine.execute(request).await?;

    // Wait for completion
    loop {
        let status = engine.get_status(execution_id).await?;
        if status == ExecutionStatus::Completed {
            break;
        }
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    }

    // Get result
    let result = engine.get_result(execution_id).await?;
    println!("Exit code: {}", result.exit_code);
    println!("Output:\n{}", result.stdout);

    Ok(())
}
```

## Command Types

The engine supports multiple command types:

```rust
// 1. AWS CLI commands
Command::AwsCli {
    service: "ec2",
    operation: "describe-instances",
    args: vec!["--region", "us-west-2"],
    profile: Some("prod"),
    region: Some("us-west-2"),
}

// 2. Execute script file
Command::Script {
    path: PathBuf::from("/tmp/deploy.sh"),
    interpreter: None, // Auto-detect from shebang
}

// 3. Execute command with args
Command::Exec {
    program: "python3".to_string(),
    args: vec!["script.py".to_string(), "--input".to_string(), "data.json".to_string()],
}

// 4. Execute shell command string
Command::Shell {
    command: "aws ec2 describe-instances | jq '.Reservations[]'".to_string(),
    shell: "bash".to_string(),
}
```

## Execution Modes

### Single Command (Background)

```rust
let execution_id = engine.execute(request).await?;
let result = engine.get_result(execution_id).await?;
```

### Multiple Commands (Serial)

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

let execution_id = engine.execute_plan(plan).await?;
```

### Multiple Commands (Parallel)

```rust
let plan = ExecutionPlan {
    id: Uuid::new_v4(),
    description: "Scan all regions".to_string(),
    strategy: ExecutionStrategy::Parallel {
        max_concurrency: Some(5),
    },
    commands: vec![request1, request2, request3],
    metadata: Default::default(),
};

let execution_id = engine.execute_plan(plan).await?;
```

## Event Streaming

Implement custom event handlers for real-time updates:

```rust
use cloudops_execution_engine::{EventHandler, ExecutionEvent};
use async_trait::async_trait;

struct MyEventHandler;

#[async_trait]
impl EventHandler for MyEventHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match event {
            ExecutionEvent::Stdout { execution_id, line } => {
                println!("[{}] {}", execution_id, line);
            }
            ExecutionEvent::Completed { execution_id, result } => {
                println!("Execution {} completed with code {}",
                    execution_id, result.exit_code);
            }
            _ => {}
        }
    }
}

// Use with engine
let engine = ExecutionEngine::new(config)
    .with_event_handler(Arc::new(MyEventHandler));
```

## Cancellation

```rust
// Start execution
let execution_id = engine.execute(request).await?;

// Cancel it
engine.cancel(execution_id).await?;
```

## Use Cases

### 1. **Tauri Desktop App**
Execute AWS commands with real-time UI updates via Tauri events.

### 2. **Server Microservice**
Execute operations triggered by API requests, stream results via WebSocket.

### 3. **CLI Tool**
Build command-line tools that execute complex workflows.

### 4. **Background Job Processor**
Execute scheduled tasks with logging and error handling.

### 5. **CI/CD Pipeline**
Run deployment scripts with parallel execution and dependency management.

## Architecture

```
┌─────────────────────────────────────────────────┐
│         ExecutionEngine (API)                   │
│  • execute()                                    │
│  • execute_plan()                               │
│  • get_status() / get_result()                  │
│  • cancel()                                     │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│         State Manager                           │
│  • Track executions (HashMap)                   │
│  • Manage cancellation tokens                   │
│  • Store results                                │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│         Executor (Tokio)                        │
│  • tokio::process::Command                      │
│  • Stream stdout/stderr                         │
│  • Handle timeouts                              │
│  • Emit events                                  │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│         Process (bash/aws/sh)                   │
└─────────────────────────────────────────────────┘
```

## Documentation

- **[Architecture](architecture.md)** - Design patterns and components
- **[API Reference](api.md)** - Complete API documentation
- **[Types](types.md)** - Data structures and formats
- **[Cargo Integration](cargo-integration.md)** - Dependency management
- **[Event Handlers](event-handlers.md)** - Custom event handlers
- **[Configuration](configuration.md)** - Configuration options
- **[Error Handling](error-handling.md)** - Error types and strategies
- **[Testing](testing.md)** - Testing guide
- **[Examples](examples/)** - Usage examples

## Examples

See [examples/](examples/) directory for:
- Simple execution
- Tauri integration
- Server integration
- CLI tool
- Advanced patterns

## Requirements

- Rust 1.70+
- Tokio runtime
- Unix-like OS or Windows (cross-platform)

## License

MIT OR Apache-2.0

## Contributing

See [CONTRIBUTING.md](../../../../CONTRIBUTING.md)

---

**Next:** [Architecture Documentation](architecture.md)