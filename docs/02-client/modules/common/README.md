# Common Types Module

**Crate Name**: `cloudops-common`
**Version**: 1.0.0
**Purpose**: Shared types and utilities across all CloudOps client modules

---

## Overview

The **Common Types Module** provides shared data structures, configuration types, and utilities used by all CloudOps client modules. This ensures type consistency, reduces duplication, and provides a single source of truth for core types.

### Philosophy

> "Define once, use everywhere. Common types are pure data structures with zero framework dependencies, making them reusable across any context."

---

## Key Principles

1. **Pure Data Structures**: No business logic, only data definitions
2. **Zero Framework Dependencies**: No Tokio, Tauri, or framework-specific code
3. **Minimal Dependencies**: Only serde, chrono, and essential serialization libraries
4. **Backward Compatible**: Types designed to be stable and extensible
5. **Well Documented**: Every type has comprehensive documentation and JSON schemas

---

## Module Organization

```
cloudops-common/
├── src/
│   ├── lib.rs               # Module exports
│   ├── aws.rs               # AWS resource types
│   ├── config.rs            # Common configuration types
│   ├── error.rs             # Error utilities
│   └── utils.rs             # Utility functions
└── Cargo.toml
```

---

## Core Types

### 1. AWS Resource Types (`aws.rs`)

#### AWSResource

Complete representation of an AWS resource with metadata, IAM permissions, and constraints.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AWSResource {
    /// Resource type (e.g., "ec2_instance", "rds_instance", "s3_bucket")
    pub resource_type: String,

    /// Unique identifier (e.g., "i-0123456789abcdef0")
    pub identifier: String,

    /// AWS account ID
    pub account_id: String,

    /// Human-readable account name
    pub account_name: String,

    /// AWS region
    pub region: String,

    /// AWS service (e.g., "ec2", "rds", "s3")
    pub service: String,

    /// Full ARN
    pub arn: String,

    /// Resource name (from tags or identifier)
    pub name: String,

    /// Current state (e.g., "running", "available", "stopped")
    pub state: String,

    /// Resource tags
    pub tags: HashMap<String, String>,

    /// IAM permissions for this resource
    pub iam: IAMPermissions,

    /// Operational constraints
    pub constraints: ResourceConstraints,

    /// Service-specific metadata
    pub metadata: serde_json::Value,

    /// Last sync timestamp
    pub last_synced: DateTime<Utc>,
}
```

**Used By**:
- ✅ Storage Service (stores in Qdrant)
- ✅ Estate Scanner (creates and enriches)
- 🔄 Request Builder (queries and sends to server)

**JSON Schema**:
```json
{
  "resource_type": "ec2_instance",
  "identifier": "i-0123456789abcdef0",
  "account_id": "123456789012",
  "account_name": "production-account",
  "region": "us-west-2",
  "service": "ec2",
  "arn": "arn:aws:ec2:us-west-2:123456789012:instance/i-0123456789abcdef0",
  "name": "web-server-1",
  "state": "running",
  "tags": {
    "env": "production",
    "app": "web"
  },
  "iam": { /* IAMPermissions */ },
  "constraints": { /* ResourceConstraints */ },
  "metadata": { /* Service-specific data */ },
  "last_synced": "2025-10-09T10:30:00Z"
}
```

---

#### IAMPermissions

Per-resource IAM permissions discovered via AWS IAM SimulatePrincipalPolicy.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IAMPermissions {
    /// Actions the current user/role is allowed to perform
    pub allowed_actions: Vec<String>,

    /// Actions explicitly denied
    pub denied_actions: Vec<String>,

    /// User/role context
    pub user_context: UserContext,
}

impl Default for IAMPermissions {
    fn default() -> Self {
        Self {
            allowed_actions: Vec::new(),
            denied_actions: Vec::new(),
            user_context: UserContext::default(),
        }
    }
}
```

**Used By**:
- ✅ Storage Service (stores with resource)
- ✅ Estate Scanner (discovers and populates)
- 🔄 Request Builder (includes in context for server)

**Example**:
```json
{
  "allowed_actions": [
    "ec2:DescribeInstances",
    "ec2:StopInstances",
    "ec2:StartInstances"
  ],
  "denied_actions": [
    "ec2:TerminateInstances"
  ],
  "user_context": {
    "username": "john.doe",
    "role_arn": "arn:aws:iam::123456789012:role/DeveloperRole",
    "session_expires": "2025-10-09T18:00:00Z"
  }
}
```

---

#### UserContext

AWS user or role context.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserContext {
    /// AWS username or user ID
    pub username: String,

    /// Role ARN if using assumed role
    #[serde(skip_serializing_if = "Option::is_none")]
    pub role_arn: Option<String>,

    /// Session expiration time (for temporary credentials)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub session_expires: Option<DateTime<Utc>>,
}

impl Default for UserContext {
    fn default() -> Self {
        Self {
            username: String::new(),
            role_arn: None,
            session_expires: None,
        }
    }
}
```

**Used By**:
- ✅ Storage Service
- ✅ Estate Scanner
- 🔄 Request Builder

---

#### ResourceConstraints

Operational constraints for a resource (can stop, start, delete, etc.).

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceConstraints {
    /// Can stop the resource (state + IAM check)
    pub can_stop: bool,

    /// Can start the resource (state + IAM check)
    pub can_start: bool,

    /// Can delete the resource (dependencies + IAM check)
    pub can_delete: bool,

    /// Has backups (snapshots, etc.)
    pub has_backups: bool,

    /// Has dependencies (attached volumes, read replicas, etc.)
    pub has_dependencies: bool,

    /// Service-specific constraints
    #[serde(flatten)]
    pub custom_constraints: HashMap<String, serde_json::Value>,
}

impl Default for ResourceConstraints {
    fn default() -> Self {
        Self {
            can_stop: false,
            can_start: false,
            can_delete: false,
            has_backups: false,
            has_dependencies: false,
            custom_constraints: HashMap::new(),
        }
    }
}
```

**Used By**:
- ✅ Storage Service
- ✅ Estate Scanner (analyzes and populates)

**Example**:
```json
{
  "can_stop": true,
  "can_start": false,
  "can_delete": false,
  "has_backups": true,
  "has_dependencies": true,
  "multi_az": true
}
```

---

#### Account

AWS account configuration.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Account {
    /// AWS account ID (12 digits)
    pub id: String,

    /// Human-readable account name
    pub name: String,

    /// AWS CLI profile name from ~/.aws/credentials
    pub profile: String,

    /// Optional: Role ARN to assume
    #[serde(skip_serializing_if = "Option::is_none")]
    pub role_arn: Option<String>,
}
```

**Used By**:
- ✅ Estate Scanner (multi-account scanning)
- 🔄 Request Builder (account context)

**Example**:
```json
{
  "id": "123456789012",
  "name": "production-account",
  "profile": "production",
  "role_arn": "arn:aws:iam::123456789012:role/ScannerRole"
}
```

---

### 2. Configuration Types (`config.rs`)

#### RetryConfig

Common retry configuration for all modules.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RetryConfig {
    /// Maximum number of retries
    #[serde(default = "default_max_retries")]
    pub max_retries: usize,

    /// Initial delay between retries (milliseconds)
    #[serde(default = "default_initial_delay")]
    pub initial_delay_ms: u64,

    /// Maximum delay between retries (milliseconds)
    #[serde(default = "default_max_delay")]
    pub max_delay_ms: u64,

    /// Use exponential backoff
    #[serde(default = "default_true")]
    pub exponential_backoff: bool,
}

fn default_max_retries() -> usize { 3 }
fn default_initial_delay() -> u64 { 1000 }
fn default_max_delay() -> u64 { 30000 }
fn default_true() -> bool { true }

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_retries: 3,
            initial_delay_ms: 1000,
            max_delay_ms: 30000,
            exponential_backoff: true,
        }
    }
}
```

**Used By**:
- ✅ Estate Scanner (AWS CLI retries)
- 🔄 Request Builder (server request retries)

**Example**:
```json
{
  "max_retries": 3,
  "initial_delay_ms": 1000,
  "max_delay_ms": 30000,
  "exponential_backoff": true
}
```

---

#### EmbeddingModelConfig

Configuration for semantic embedding model.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EmbeddingModelConfig {
    /// Model name (e.g., "all-MiniLM-L6-v2")
    #[serde(default = "default_model_name")]
    pub model_name: String,

    /// Embedding dimension
    #[serde(default = "default_dimension")]
    pub dimension: usize,

    /// Batch size for embedding generation
    #[serde(default = "default_batch_size")]
    pub batch_size: usize,
}

fn default_model_name() -> String { "all-MiniLM-L6-v2".to_string() }
fn default_dimension() -> usize { 384 }
fn default_batch_size() -> usize { 100 }

impl Default for EmbeddingModelConfig {
    fn default() -> Self {
        Self {
            model_name: "all-MiniLM-L6-v2".to_string(),
            dimension: 384,
            batch_size: 100,
        }
    }
}
```

**Used By**:
- ✅ Estate Scanner (generates embeddings)
- ✅ Storage Service (validates dimension)

**Example**:
```json
{
  "model_name": "all-MiniLM-L6-v2",
  "dimension": 384,
  "batch_size": 100
}
```

---

### 3. Error Utilities (`error.rs`)

#### ErrorCategory

Common error categorization for all modules.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum ErrorCategory {
    /// Command execution errors
    Execution,

    /// AWS CLI errors
    AwsCli,

    /// JSON/data parsing errors
    Parsing,

    /// IAM/permission errors
    IAM,

    /// Embedding generation errors
    Embedding,

    /// Storage/database errors
    Storage,

    /// Network/HTTP errors
    Network,

    /// Configuration errors
    Configuration,
}
```

**Used By**:
- ✅ All modules (error categorization)

---

## Installation

Add to your module's `Cargo.toml`:

```toml
[dependencies]
cloudops-common = { path = "../cloudops-common" }
```

Or if published to crates.io:

```toml
[dependencies]
cloudops-common = "1.0"
```

---

## Usage Examples

### Import All AWS Types

```rust
use cloudops_common::aws::*;

let resource = AWSResource {
    resource_type: "ec2_instance".to_string(),
    identifier: "i-0123456789abcdef0".to_string(),
    // ...
};
```

### Import Specific Types

```rust
use cloudops_common::aws::{AWSResource, IAMPermissions, Account};
use cloudops_common::config::{RetryConfig, EmbeddingModelConfig};
```

### Use in Storage Service

```rust
use cloudops_common::aws::{AWSResource, IAMPermissions};

pub struct EstateStorage {
    // ...
}

impl EstateStorage {
    pub async fn upsert_resource(&self, resource: AWSResource) -> Result<()> {
        // resource.iam contains IAMPermissions
        // resource.constraints contains ResourceConstraints
        // ...
    }
}
```

### Use in Estate Scanner

```rust
use cloudops_common::aws::{AWSResource, Account};
use cloudops_common::config::RetryConfig;

pub struct EstateScanner {
    retry_config: RetryConfig,
}

impl EstateScanner {
    pub async fn scan(&self, accounts: Vec<Account>) -> Result<Vec<AWSResource>> {
        // Use Account struct
        // Create AWSResource instances
        // ...
    }
}
```

---

## Type Compatibility

All types in `cloudops-common` are:

- ✅ **Serializable**: Via `serde` to/from JSON
- ✅ **Clonable**: All types implement `Clone`
- ✅ **Debug**: All types implement `Debug`
- ✅ **Send + Sync**: Safe for concurrent use
- ✅ **Documented**: Comprehensive doc comments

---

## Dependencies

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
```

**Minimal dependencies** ensure the crate is:
- Lightweight
- Fast to compile
- Easy to integrate
- Framework-agnostic

---

## Module Status

| Component | Status |
|-----------|--------|
| AWS Types | ✅ Complete |
| Config Types | ✅ Complete |
| Error Utilities | ✅ Complete |
| Documentation | ✅ Complete |
| Tests | 📝 Pending |

---

## Migration Notes

### For Existing Modules

When migrating to use `cloudops-common`:

1. **Add Dependency**:
   ```toml
   cloudops-common = { path = "../cloudops-common" }
   ```

2. **Update Imports**:
   ```rust
   // Before
   use crate::types::{AWSResource, IAMPermissions};

   // After
   use cloudops_common::aws::{AWSResource, IAMPermissions};
   ```

3. **Remove Duplicate Definitions**: Delete local type definitions that now exist in common

4. **Update Documentation**: Reference `cloudops-common` in type documentation

---

## See Also

- [Storage Service](../storage-service/) - Uses AWS types
- [Estate Scanner](../estate-scanner/) - Creates and enriches AWS types
- [Request Builder](../request-builder/) - Queries and uses AWS types

---

## Design Philosophy

### Why a Common Crate?

**Problem**: Types like `AWSResource` and `IAMPermissions` are used by multiple modules, leading to:
- 🔴 Duplication across modules
- 🔴 Risk of type incompatibility
- 🔴 Difficult refactoring (change in multiple places)

**Solution**: Extract common types to shared crate:
- ✅ Single source of truth
- ✅ Type safety guaranteed
- ✅ Easy refactoring (change once)
- ✅ Better documentation

### What Belongs in Common?

**Include**:
- ✅ Types used by 2+ modules
- ✅ Pure data structures (no business logic)
- ✅ Core domain models (AWS resources, accounts, etc.)
- ✅ Common configuration patterns

**Exclude**:
- ❌ Module-specific types
- ❌ Business logic or algorithms
- ❌ Framework-specific code
- ❌ Types used by only one module

---

**Status**: ✅ Design Complete | Ready for implementation