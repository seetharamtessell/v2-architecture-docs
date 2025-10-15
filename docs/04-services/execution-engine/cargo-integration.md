# Cargo Integration Guide

**Crate**: `cloudops-execution-engine`

Complete guide for using Cargo to manage the Execution Engine as a dependency.

---

## Table of Contents

- [What is Cargo?](#what-is-cargo)
- [Adding the Crate](#adding-the-crate)
- [Version Management](#version-management)
- [Feature Flags](#feature-flags)
- [Workspace Setup](#workspace-setup)
- [Local Development](#local-development)
- [Publishing](#publishing)
- [Dependency Updates](#dependency-updates)
- [Common Patterns](#common-patterns)

---

## What is Cargo?

**Cargo** is Rust's package manager and build system (equivalent to npm for JavaScript).

### Key Concepts

| Concept | Description | npm Equivalent |
|---------|-------------|----------------|
| **Crate** | A Rust package | npm package |
| **Cargo.toml** | Package manifest | package.json |
| **Cargo.lock** | Dependency lockfile | package-lock.json |
| **crates.io** | Package registry | npmjs.com |
| **cargo add** | Add dependency | npm install |
| **cargo build** | Build project | npm run build |
| **cargo test** | Run tests | npm test |
| **cargo publish** | Publish crate | npm publish |

---

## Adding the Crate

### Method 1: Using `cargo add` (Recommended)

```bash
# Add the execution engine
cargo add cloudops-execution-engine

# Add with specific version
cargo add cloudops-execution-engine@0.1.0

# Add required dependencies
cargo add tokio --features full
cargo add uuid --features v4,serde
cargo add serde --features derive
cargo add serde_json
```

**Equivalent to:**
```bash
npm install cloudops-execution-engine
```

### Method 2: Manual `Cargo.toml` Edit

Add to your `Cargo.toml`:

```toml
[dependencies]
cloudops-execution-engine = "0.1.0"
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = { version = "0.4", features = ["serde"] }
```

Then run:
```bash
cargo build
```

---

## Version Management

### Semantic Versioning

Cargo uses [SemVer](https://semver.org/) just like npm:

```toml
[dependencies]
# Exact version
cloudops-execution-engine = "=0.1.0"

# Compatible with 0.1.x (default)
cloudops-execution-engine = "0.1"

# Compatible with 0.x.x
cloudops-execution-engine = "0"

# Range
cloudops-execution-engine = ">=0.1.0, <0.2.0"

# Tilde requirement (patch updates)
cloudops-execution-engine = "~0.1.0"  # 0.1.0 <= version < 0.2.0

# Caret requirement (minor updates)
cloudops-execution-engine = "^0.1.0"  # 0.1.0 <= version < 1.0.0
```

### Version Comparison

| Cargo | npm Equivalent | Meaning |
|-------|----------------|---------|
| `"0.1.0"` | `"^0.1.0"` | Compatible updates (default) |
| `"=0.1.0"` | `"0.1.0"` | Exact version |
| `"~0.1.0"` | `"~0.1.0"` | Patch updates only |
| `"^0.1.0"` | `"^0.1.0"` | Minor updates allowed |

### Updating Dependencies

```bash
# Update to latest compatible version
cargo update

# Update specific crate
cargo update cloudops-execution-engine

# Update to latest (may break compatibility)
cargo update --breaking
```

**Equivalent to:**
```bash
npm update cloudops-execution-engine
```

---

## Feature Flags

Features allow conditional compilation of code (similar to optional dependencies in npm).

### Available Features

```toml
[dependencies]
cloudops-execution-engine = { version = "0.1", features = ["tauri-events"] }
```

#### Feature: `tauri-events`

Enable Tauri event system integration.

```toml
[dependencies]
cloudops-execution-engine = { version = "0.1", features = ["tauri-events"] }
tauri = "2"
```

**Usage:**
```rust
use cloudops_execution_engine::tauri::TauriEventHandler;

let handler = TauriEventHandler::new(app_handle);
let engine = ExecutionEngine::new(config)
    .with_event_handler(Arc::new(handler));
```

### Default Features

```toml
# With default features
cloudops-execution-engine = "0.1"

# Without default features
cloudops-execution-engine = { version = "0.1", default-features = false }

# Custom features only
cloudops-execution-engine = {
    version = "0.1",
    default-features = false,
    features = ["tauri-events"]
}
```

---

## Workspace Setup

For multi-crate projects (monorepo), use Cargo workspaces.

### Workspace Structure

```
my-project/
├── Cargo.toml          # Workspace root
├── client/
│   ├── Cargo.toml      # Client crate
│   └── src/
├── server/
│   ├── Cargo.toml      # Server crate
│   └── src/
└── shared/
    ├── Cargo.toml      # Shared crate
    └── src/
```

### Root `Cargo.toml`

```toml
[workspace]
members = [
    "client",
    "server",
    "shared",
]

# Shared dependencies (Rust 1.70+)
[workspace.dependencies]
cloudops-execution-engine = "0.1.0"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

### Member `Cargo.toml`

```toml
[package]
name = "my-client"
version = "0.1.0"
edition = "2021"

[dependencies]
# Use workspace dependency
cloudops-execution-engine.workspace = true
tokio.workspace = true
serde.workspace = true
```

### Workspace Commands

```bash
# Build all workspace members
cargo build

# Build specific member
cargo build -p my-client

# Run tests for all members
cargo test

# Run tests for specific member
cargo test -p my-server
```

---

## Local Development

### Using Path Dependencies

For local development, reference the crate by path:

```toml
[dependencies]
cloudops-execution-engine = { path = "../cloudops-execution-engine" }
```

**Equivalent to:**
```bash
npm link cloudops-execution-engine
```

### Example Setup

```
my-project/
├── Cargo.toml
├── src/
└── ../cloudops-execution-engine/
    ├── Cargo.toml
    └── src/
```

**my-project/Cargo.toml:**
```toml
[dependencies]
cloudops-execution-engine = { path = "../cloudops-execution-engine" }
```

### Git Dependencies

Use a crate directly from Git:

```toml
[dependencies]
cloudops-execution-engine = {
    git = "https://github.com/yourorg/cloudops-execution-engine",
    branch = "main"
}

# Or specific commit
cloudops-execution-engine = {
    git = "https://github.com/yourorg/cloudops-execution-engine",
    rev = "abc123"
}

# Or specific tag
cloudops-execution-engine = {
    git = "https://github.com/yourorg/cloudops-execution-engine",
    tag = "v0.1.0"
}
```

**Equivalent to:**
```bash
npm install github:yourorg/cloudops-execution-engine
```

---

## Publishing

### Prerequisites

1. **Create crates.io account**: https://crates.io/
2. **Get API token**: https://crates.io/me
3. **Login to cargo**:
   ```bash
   cargo login <your-api-token>
   ```

### Prepare for Publishing

#### 1. Update `Cargo.toml`

```toml
[package]
name = "cloudops-execution-engine"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <you@example.com>"]
license = "MIT OR Apache-2.0"
description = "Pure Rust execution engine for running commands with streaming support"
homepage = "https://github.com/yourorg/cloudops-execution-engine"
repository = "https://github.com/yourorg/cloudops-execution-engine"
documentation = "https://docs.rs/cloudops-execution-engine"
readme = "README.md"
keywords = ["execution", "command", "async", "tokio", "aws"]
categories = ["asynchronous", "command-line-utilities"]

[dependencies]
# ... your dependencies
```

#### 2. Add Required Files

```
cloudops-execution-engine/
├── Cargo.toml
├── README.md          # Required
├── LICENSE-MIT        # Recommended
├── LICENSE-APACHE     # Recommended
└── src/
    └── lib.rs
```

#### 3. Test Package

```bash
# Dry-run to check what would be published
cargo publish --dry-run

# Build documentation locally
cargo doc --open
```

### Publish to crates.io

```bash
cargo publish
```

**Equivalent to:**
```bash
npm publish
```

### Version Updates

```bash
# Update version in Cargo.toml
# Then publish new version
cargo publish
```

**Note:** You cannot overwrite a published version. Always increment version number.

---

## Dependency Updates

### Check for Updates

```bash
# Install cargo-outdated
cargo install cargo-outdated

# Check outdated dependencies
cargo outdated
```

**Equivalent to:**
```bash
npm outdated
```

### Update Dependencies

```bash
# Update to latest compatible versions
cargo update

# Update and edit Cargo.toml
cargo upgrade  # Requires: cargo install cargo-edit
```

### Audit Dependencies

```bash
# Install cargo-audit
cargo install cargo-audit

# Check for security vulnerabilities
cargo audit
```

**Equivalent to:**
```bash
npm audit
```

---

## Common Patterns

### Pattern 1: Basic Application

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2021"

[dependencies]
cloudops-execution-engine = "0.1"
tokio = { version = "1", features = ["full"] }
anyhow = "1"
```

```rust
use cloudops_execution_engine::{ExecutionEngine, ExecutionConfig};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let engine = ExecutionEngine::new(ExecutionConfig::default());
    // ... use engine
    Ok(())
}
```

### Pattern 2: Tauri Application

```toml
[package]
name = "my-tauri-app"
version = "0.1.0"
edition = "2021"

[dependencies]
cloudops-execution-engine = { version = "0.1", features = ["tauri-events"] }
tauri = "2"
tokio = { version = "1", features = ["full"] }
```

```rust
use tauri::Manager;
use cloudops_execution_engine::{ExecutionEngine, ExecutionConfig};

#[tauri::command]
async fn execute_command(
    engine: tauri::State<'_, ExecutionEngine>,
    request: ExecutionRequest,
) -> Result<Uuid, String> {
    engine.execute(request)
        .await
        .map_err(|e| e.to_string())
}

fn main() {
    let engine = ExecutionEngine::new(ExecutionConfig::default());

    tauri::Builder::default()
        .manage(engine)
        .invoke_handler(tauri::generate_handler![execute_command])
        .run(tauri::generate_context!())
        .expect("error running tauri app");
}
```

### Pattern 3: Server Application

```toml
[package]
name = "my-server"
version = "0.1.0"
edition = "2021"

[dependencies]
cloudops-execution-engine = "0.1"
axum = "0.7"
tokio = { version = "1", features = ["full"] }
```

```rust
use axum::{Router, routing::post, Json};
use cloudops_execution_engine::{ExecutionEngine, ExecutionRequest};
use std::sync::Arc;

async fn execute(
    engine: Arc<ExecutionEngine>,
    Json(request): Json<ExecutionRequest>,
) -> Json<Uuid> {
    let execution_id = engine.execute(request).await.unwrap();
    Json(execution_id)
}

#[tokio::main]
async fn main() {
    let engine = Arc::new(ExecutionEngine::new(Default::default()));

    let app = Router::new()
        .route("/execute", post(execute))
        .with_state(engine);

    axum::Server::bind(&"0.0.0.0:3000".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

### Pattern 4: CLI Tool

```toml
[package]
name = "my-cli"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "mycli"
path = "src/main.rs"

[dependencies]
cloudops-execution-engine = "0.1"
clap = { version = "4", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

```rust
use clap::Parser;
use cloudops_execution_engine::{ExecutionEngine, ExecutionRequest, Command};

#[derive(Parser)]
struct Cli {
    #[arg(short, long)]
    service: String,

    #[arg(short, long)]
    operation: String,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    let engine = ExecutionEngine::new(Default::default());

    let request = ExecutionRequest {
        id: Uuid::new_v4(),
        command: Command::AwsCli {
            service: cli.service,
            operation: cli.operation,
            args: vec![],
            profile: None,
            region: None,
        },
        env: HashMap::new(),
        working_dir: None,
        timeout_ms: None,
        metadata: Default::default(),
    };

    let execution_id = engine.execute(request).await?;
    let result = engine.get_result(execution_id).await?;

    println!("{}", result.stdout);
    Ok(())
}
```

---

## Cargo Commands Reference

### Essential Commands

```bash
# Create new project
cargo new my-project
cargo new my-library --lib

# Add dependency
cargo add cloudops-execution-engine

# Build project
cargo build
cargo build --release

# Run project
cargo run
cargo run --release

# Run tests
cargo test

# Generate documentation
cargo doc --open

# Check code (faster than build)
cargo check

# Format code
cargo fmt

# Lint code
cargo clippy

# Clean build artifacts
cargo clean
```

### Dependency Management

```bash
# Add dependency
cargo add <crate-name>

# Remove dependency
cargo remove <crate-name>

# Update dependencies
cargo update

# Show dependency tree
cargo tree

# Check for outdated deps
cargo outdated
```

### Publishing

```bash
# Login to crates.io
cargo login <token>

# Package crate
cargo package

# Dry-run publish
cargo publish --dry-run

# Publish to crates.io
cargo publish

# Yank a version
cargo yank --vers 0.1.0
```

---

## npm to Cargo Command Mapping

| npm | Cargo | Description |
|-----|-------|-------------|
| `npm init` | `cargo new` | Create new project |
| `npm install` | `cargo build` | Install dependencies |
| `npm install <pkg>` | `cargo add <crate>` | Add dependency |
| `npm uninstall <pkg>` | `cargo remove <crate>` | Remove dependency |
| `npm update` | `cargo update` | Update dependencies |
| `npm outdated` | `cargo outdated` | Check outdated deps |
| `npm run build` | `cargo build` | Build project |
| `npm test` | `cargo test` | Run tests |
| `npm publish` | `cargo publish` | Publish package |
| `npm audit` | `cargo audit` | Security audit |
| `npm run <script>` | `cargo run --bin <name>` | Run binary |
| `npm link` | path dependency | Local development |

---

## Troubleshooting

### Issue: Dependency Conflict

```
error: failed to select a version for `tokio`
```

**Solution:** Specify compatible versions or use workspace dependencies.

### Issue: Feature Not Found

```
error: feature `tauri-events` is not available
```

**Solution:** Check feature name spelling and crate version.

### Issue: Can't Find Crate

```
error: no matching package named `cloudops-execution-engine` found
```

**Solution:**
- Check crate name spelling
- Ensure crate is published to crates.io
- Use path or git dependency for unpublished crates

### Issue: Build Fails

```bash
# Clear cache and rebuild
cargo clean
cargo build
```

---

## Best Practices

1. **Use `Cargo.lock`** - Commit to version control for applications
2. **Semantic versioning** - Follow SemVer for version updates
3. **Feature flags** - Keep optional dependencies behind features
4. **Documentation** - Document public API with `///` comments
5. **Testing** - Write unit and integration tests
6. **CI/CD** - Automate testing and publishing

---

## Related Documents

- [API Reference](api.md)
- [Architecture](architecture.md)
- [Usage Examples](usage.md)
- [Official Cargo Book](https://doc.rust-lang.org/cargo/)