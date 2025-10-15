# Event Handlers Guide

**Crate**: `cloudops-execution-engine`

Complete guide for implementing custom event handlers.

---

## Table of Contents

- [Overview](#overview)
- [EventHandler Trait](#eventhandler-trait)
- [Built-in Handlers](#built-in-handlers)
- [Custom Implementations](#custom-implementations)
- [Integration Examples](#integration-examples)
- [Best Practices](#best-practices)

---

## Overview

The Execution Engine uses the **Observer pattern** to emit events during execution. Custom event handlers allow you to:

- Stream output to UI in real-time
- Log executions to files or databases
- Send notifications (email, Slack, etc.)
- Update progress bars
- Integrate with external systems

---

## EventHandler Trait

### Definition

```rust
#[async_trait::async_trait]
pub trait EventHandler: Send + Sync {
    async fn on_event(&self, event: ExecutionEvent);
}
```

### Requirements

- **`Send + Sync`**: Handler must be thread-safe
- **`async`**: Handler method is async
- **Non-blocking**: Handlers should not block execution

---

## ExecutionEvent Types

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

---

## Built-in Handlers

### Console Handler

Simple handler that prints to stdout/stderr.

```rust
use cloudops_execution_engine::{EventHandler, ExecutionEvent};
use async_trait::async_trait;

pub struct ConsoleHandler;

#[async_trait]
impl EventHandler for ConsoleHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match event {
            ExecutionEvent::Started { execution_id, command, .. } => {
                println!("[START] {} - {}", execution_id, command);
            }
            ExecutionEvent::Stdout { line, .. } => {
                println!("{}", line);
            }
            ExecutionEvent::Stderr { line, .. } => {
                eprintln!("{}", line);
            }
            ExecutionEvent::Completed { execution_id, result, .. } => {
                println!("[DONE] {} - Exit code: {}", execution_id, result.exit_code);
            }
            ExecutionEvent::Failed { execution_id, error, .. } => {
                eprintln!("[FAIL] {} - {}", execution_id, error);
            }
            ExecutionEvent::Cancelled { execution_id, .. } => {
                println!("[CANCEL] {}", execution_id);
            }
            ExecutionEvent::Progress { completed, total, .. } => {
                println!("Progress: {}/{}", completed, total);
            }
        }
    }
}

// Usage
let engine = ExecutionEngine::new(config)
    .with_event_handler(Arc::new(ConsoleHandler));
```

### No-op Handler

Handler that does nothing (for testing or when events not needed).

```rust
pub struct NoopHandler;

#[async_trait]
impl EventHandler for NoopHandler {
    async fn on_event(&self, _event: ExecutionEvent) {
        // Do nothing
    }
}
```

---

## Custom Implementations

### File Logger Handler

Log all events to a file.

```rust
use tokio::sync::Mutex;
use tokio::fs::{File, OpenOptions};
use tokio::io::AsyncWriteExt;

pub struct FileLoggerHandler {
    file: Arc<Mutex<File>>,
}

impl FileLoggerHandler {
    pub async fn new(path: &str) -> Result<Self, std::io::Error> {
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(path)
            .await?;

        Ok(Self {
            file: Arc::new(Mutex::new(file)),
        })
    }
}

#[async_trait]
impl EventHandler for FileLoggerHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        let log_entry = format!("{}\n", serde_json::to_string(&event).unwrap());

        let mut file = self.file.lock().await;
        file.write_all(log_entry.as_bytes()).await.ok();
        file.flush().await.ok();
    }
}

// Usage
let handler = FileLoggerHandler::new("/var/log/executions.jsonl").await?;
let engine = ExecutionEngine::new(config)
    .with_event_handler(Arc::new(handler));
```

### WebSocket Handler

Stream events to WebSocket clients.

```rust
use tokio::sync::broadcast;

pub struct WebSocketHandler {
    tx: broadcast::Sender<String>,
}

impl WebSocketHandler {
    pub fn new() -> (Self, broadcast::Receiver<String>) {
        let (tx, rx) = broadcast::channel(100);
        (Self { tx }, rx)
    }
}

#[async_trait]
impl EventHandler for WebSocketHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        let json = serde_json::to_string(&event).unwrap();
        self.tx.send(json).ok();
    }
}

// Usage with Axum WebSocket
use axum::{
    extract::ws::{WebSocket, WebSocketUpgrade},
    response::IntoResponse,
};

async fn websocket_handler(
    ws: WebSocketUpgrade,
    rx: broadcast::Receiver<String>,
) -> impl IntoResponse {
    ws.on_upgrade(|socket| handle_socket(socket, rx))
}

async fn handle_socket(mut socket: WebSocket, mut rx: broadcast::Receiver<String>) {
    while let Ok(msg) = rx.recv().await {
        if socket.send(axum::extract::ws::Message::Text(msg)).await.is_err() {
            break;
        }
    }
}
```

### Database Logger Handler

Store events in a database.

```rust
use sqlx::{PgPool, Postgres};

pub struct DatabaseHandler {
    pool: PgPool,
}

impl DatabaseHandler {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl EventHandler for DatabaseHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match event {
            ExecutionEvent::Completed { execution_id, result, timestamp } => {
                sqlx::query!(
                    r#"
                    INSERT INTO execution_logs
                    (id, exit_code, stdout, stderr, started_at, completed_at)
                    VALUES ($1, $2, $3, $4, $5, $6)
                    "#,
                    execution_id,
                    result.exit_code,
                    result.stdout,
                    result.stderr,
                    result.started_at,
                    timestamp,
                )
                .execute(&self.pool)
                .await
                .ok();
            }
            _ => {}
        }
    }
}
```

### Multi-Handler (Fan-out)

Broadcast events to multiple handlers.

```rust
pub struct MultiHandler {
    handlers: Vec<Arc<dyn EventHandler>>,
}

impl MultiHandler {
    pub fn new(handlers: Vec<Arc<dyn EventHandler>>) -> Self {
        Self { handlers }
    }
}

#[async_trait]
impl EventHandler for MultiHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        // Send to all handlers concurrently
        let futures: Vec<_> = self
            .handlers
            .iter()
            .map(|h| h.on_event(event.clone()))
            .collect();

        futures::future::join_all(futures).await;
    }
}

// Usage
let console = Arc::new(ConsoleHandler);
let file_logger = Arc::new(FileLoggerHandler::new("/var/log/exec.log").await?);
let ws_handler = Arc::new(WebSocketHandler::new().0);

let multi_handler = MultiHandler::new(vec![console, file_logger, ws_handler]);

let engine = ExecutionEngine::new(config)
    .with_event_handler(Arc::new(multi_handler));
```

---

## Integration Examples

### Tauri Event Handler

Emit events to Tauri frontend.

```rust
use tauri::{AppHandle, Manager};

pub struct TauriEventHandler {
    app_handle: AppHandle,
}

impl TauriEventHandler {
    pub fn new(app_handle: AppHandle) -> Self {
        Self { app_handle }
    }
}

#[async_trait]
impl EventHandler for TauriEventHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match &event {
            ExecutionEvent::Stdout { execution_id, line, .. } => {
                self.app_handle
                    .emit_all("execution:stdout", json!({
                        "execution_id": execution_id,
                        "line": line,
                    }))
                    .ok();
            }
            ExecutionEvent::Stderr { execution_id, line, .. } => {
                self.app_handle
                    .emit_all("execution:stderr", json!({
                        "execution_id": execution_id,
                        "line": line,
                    }))
                    .ok();
            }
            ExecutionEvent::Completed { execution_id, result, .. } => {
                self.app_handle
                    .emit_all("execution:completed", json!({
                        "execution_id": execution_id,
                        "exit_code": result.exit_code,
                    }))
                    .ok();
            }
            ExecutionEvent::Progress { plan_id, completed, total, .. } => {
                self.app_handle
                    .emit_all("execution:progress", json!({
                        "plan_id": plan_id,
                        "completed": completed,
                        "total": total,
                    }))
                    .ok();
            }
            _ => {}
        }
    }
}

// In Tauri app setup
fn main() {
    tauri::Builder::default()
        .setup(|app| {
            let app_handle = app.handle();
            let handler = TauriEventHandler::new(app_handle);

            let engine = ExecutionEngine::new(ExecutionConfig::default())
                .with_event_handler(Arc::new(handler));

            app.manage(engine);
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error running tauri app");
}
```

**Frontend (TypeScript):**
```typescript
import { listen } from '@tauri-apps/api/event';

// Listen for stdout
await listen('execution:stdout', (event) => {
  const { execution_id, line } = event.payload;
  console.log(`[${execution_id}] ${line}`);
});

// Listen for completion
await listen('execution:completed', (event) => {
  const { execution_id, exit_code } = event.payload;
  console.log(`Execution ${execution_id} completed with code ${exit_code}`);
});

// Listen for progress
await listen('execution:progress', (event) => {
  const { plan_id, completed, total } = event.payload;
  console.log(`Plan ${plan_id}: ${completed}/${total}`);
});
```

### Slack Notification Handler

Send notifications to Slack.

```rust
use reqwest::Client;

pub struct SlackHandler {
    webhook_url: String,
    client: Client,
}

impl SlackHandler {
    pub fn new(webhook_url: String) -> Self {
        Self {
            webhook_url,
            client: Client::new(),
        }
    }

    async fn send_message(&self, text: String) {
        self.client
            .post(&self.webhook_url)
            .json(&serde_json::json!({ "text": text }))
            .send()
            .await
            .ok();
    }
}

#[async_trait]
impl EventHandler for SlackHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match event {
            ExecutionEvent::Failed { execution_id, error, .. } => {
                let message = format!(
                    ":x: Execution {} failed: {}",
                    execution_id, error
                );
                self.send_message(message).await;
            }
            ExecutionEvent::Completed { execution_id, result, .. }
                if result.exit_code != 0 =>
            {
                let message = format!(
                    ":warning: Execution {} completed with exit code {}",
                    execution_id, result.exit_code
                );
                self.send_message(message).await;
            }
            _ => {}
        }
    }
}
```

### Metrics Handler

Track execution metrics.

```rust
use std::sync::atomic::{AtomicU64, Ordering};

pub struct MetricsHandler {
    total_executions: Arc<AtomicU64>,
    completed_executions: Arc<AtomicU64>,
    failed_executions: Arc<AtomicU64>,
    total_duration_ms: Arc<AtomicU64>,
}

impl MetricsHandler {
    pub fn new() -> Self {
        Self {
            total_executions: Arc::new(AtomicU64::new(0)),
            completed_executions: Arc::new(AtomicU64::new(0)),
            failed_executions: Arc::new(AtomicU64::new(0)),
            total_duration_ms: Arc::new(AtomicU64::new(0)),
        }
    }

    pub fn stats(&self) -> MetricsStats {
        MetricsStats {
            total: self.total_executions.load(Ordering::Relaxed),
            completed: self.completed_executions.load(Ordering::Relaxed),
            failed: self.failed_executions.load(Ordering::Relaxed),
            avg_duration_ms: {
                let total_ms = self.total_duration_ms.load(Ordering::Relaxed);
                let total = self.total_executions.load(Ordering::Relaxed);
                if total > 0 {
                    total_ms / total
                } else {
                    0
                }
            },
        }
    }
}

#[async_trait]
impl EventHandler for MetricsHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        match event {
            ExecutionEvent::Started { .. } => {
                self.total_executions.fetch_add(1, Ordering::Relaxed);
            }
            ExecutionEvent::Completed { result, .. } => {
                self.completed_executions.fetch_add(1, Ordering::Relaxed);
                self.total_duration_ms.fetch_add(
                    result.duration.as_millis() as u64,
                    Ordering::Relaxed,
                );
            }
            ExecutionEvent::Failed { .. } | ExecutionEvent::Cancelled { .. } => {
                self.failed_executions.fetch_add(1, Ordering::Relaxed);
            }
            _ => {}
        }
    }
}

#[derive(Debug)]
pub struct MetricsStats {
    pub total: u64,
    pub completed: u64,
    pub failed: u64,
    pub avg_duration_ms: u64,
}
```

---

## Best Practices

### 1. Keep Handlers Fast

Event handlers should not block execution. Offload heavy work to background tasks.

```rust
// ❌ Bad: Blocking I/O in handler
#[async_trait]
impl EventHandler for BadHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        // This blocks the execution engine!
        std::thread::sleep(Duration::from_secs(1));
    }
}

// ✅ Good: Async I/O
#[async_trait]
impl EventHandler for GoodHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        tokio::spawn(async move {
            // Heavy work in background
            tokio::time::sleep(Duration::from_secs(1)).await;
        });
    }
}
```

### 2. Handle Errors Gracefully

Don't panic in event handlers.

```rust
#[async_trait]
impl EventHandler for ResilientHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        if let Err(e) = self.process_event(&event).await {
            eprintln!("Error processing event: {}", e);
            // Log error but don't panic
        }
    }
}
```

### 3. Use Channels for Buffering

Buffer events to prevent backpressure.

```rust
use tokio::sync::mpsc;

pub struct BufferedHandler {
    tx: mpsc::Sender<ExecutionEvent>,
}

impl BufferedHandler {
    pub fn new(buffer_size: usize) -> Self {
        let (tx, mut rx) = mpsc::channel(buffer_size);

        // Spawn worker to process events
        tokio::spawn(async move {
            while let Some(event) = rx.recv().await {
                // Process event
                process_event(event).await;
            }
        });

        Self { tx }
    }
}

#[async_trait]
impl EventHandler for BufferedHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        self.tx.send(event).await.ok();
    }
}
```

### 4. Clone Data When Needed

Events are cloned when sent to handlers. Keep payload small.

```rust
// ✅ Good: Events contain only necessary data
ExecutionEvent::Stdout {
    execution_id,
    line,  // Just the line, not entire output
    timestamp,
}
```

### 5. Use Filters

Only handle relevant events.

```rust
#[async_trait]
impl EventHandler for FilteredHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        // Only handle completed events
        if let ExecutionEvent::Completed { .. } = event {
            self.process_completion(event).await;
        }
    }
}
```

---

## Testing Event Handlers

### Mock Event Handler

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

pub struct MockHandler {
    events: Arc<Mutex<Vec<ExecutionEvent>>>,
}

impl MockHandler {
    pub fn new() -> Self {
        Self {
            events: Arc::new(Mutex::new(Vec::new())),
        }
    }

    pub async fn get_events(&self) -> Vec<ExecutionEvent> {
        self.events.lock().await.clone()
    }
}

#[async_trait]
impl EventHandler for MockHandler {
    async fn on_event(&self, event: ExecutionEvent) {
        self.events.lock().await.push(event);
    }
}

// Test
#[tokio::test]
async fn test_event_emission() {
    let handler = Arc::new(MockHandler::new());
    let engine = ExecutionEngine::new(config)
        .with_event_handler(handler.clone());

    let request = /* ... */;
    engine.execute(request).await.unwrap();

    // Wait and check events
    tokio::time::sleep(Duration::from_secs(1)).await;

    let events = handler.get_events().await;
    assert!(events.iter().any(|e| matches!(e, ExecutionEvent::Started { .. })));
    assert!(events.iter().any(|e| matches!(e, ExecutionEvent::Completed { .. })));
}
```

---

## Related Documents

- [API Reference](api.md)
- [Usage Examples](usage.md)
- [Configuration](configuration.md)