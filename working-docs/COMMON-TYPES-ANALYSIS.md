# Common Types Analysis

**Date**: 2025-10-09
**Purpose**: Identify common types across modules for shared `cloudops-common` crate

---

## Overview

After completing 3 modules (Storage Service, Execution Engine, Estate Scanner), we've identified types that appear across multiple modules and should be extracted into a common crate.

---

## Common Types by Category

### 1. AWS Resource Types (Used in Storage + Estate Scanner)

**Shared by**: Storage Service, Estate Scanner

```rust
pub struct AWSResource {
    pub resource_type: String,
    pub identifier: String,
    pub account_id: String,
    pub account_name: String,
    pub region: String,
    pub service: String,
    pub arn: String,
    pub name: String,
    pub state: String,
    pub tags: HashMap<String, String>,
    pub iam: IAMPermissions,
    pub constraints: ResourceConstraints,
    pub metadata: serde_json::Value,
    pub last_synced: DateTime<Utc>,
}
```

**Location**:
- ✅ Defined in Storage Service (api.md)
- ✅ Referenced in Estate Scanner (types.md)

**Decision**: Move to `cloudops-common` crate

---

### 2. IAM Types (Used in Storage + Estate Scanner)

**Shared by**: Storage Service, Estate Scanner

```rust
pub struct IAMPermissions {
    pub allowed_actions: Vec<String>,
    pub denied_actions: Vec<String>,
    pub user_context: UserContext,
}

pub struct UserContext {
    pub username: String,
    pub role_arn: Option<String>,
    pub session_expires: Option<DateTime<Utc>>,
}
```

**Location**:
- ✅ Defined in Storage Service (api.md)
- ✅ Duplicated in Estate Scanner (types.md)

**Decision**: Move to `cloudops-common` crate

---

### 3. Resource Constraints (Used in Storage + Estate Scanner)

**Shared by**: Storage Service, Estate Scanner

```rust
pub struct ResourceConstraints {
    pub can_stop: bool,
    pub can_start: bool,
    pub can_delete: bool,
    pub has_backups: bool,
    pub has_dependencies: bool,
    pub custom_constraints: HashMap<String, serde_json::Value>,
}
```

**Location**:
- ✅ Defined in Storage Service (api.md)
- ✅ Duplicated in Estate Scanner (types.md)

**Decision**: Move to `cloudops-common` crate

---

### 4. Account Types (Used in Estate Scanner, likely in Request Builder)

**Shared by**: Estate Scanner, (Future: Request Builder)

```rust
pub struct Account {
    pub id: String,
    pub name: String,
    pub profile: String,
    pub role_arn: Option<String>,
}
```

**Location**:
- ✅ Defined in Estate Scanner (types.md)
- ⚠️ Will be needed by Request Builder

**Decision**: Move to `cloudops-common` crate

---

### 5. Configuration Types (Pattern across all modules)

**Pattern**: Every module has a `Config` struct

**Storage Service**:
```rust
pub struct StorageConfig {
    pub data_dir: PathBuf,
    pub qdrant_url: String,
    pub encryption_key: Option<String>,
    // ...
}
```

**Execution Engine**:
```rust
pub struct ExecutionConfig {
    pub log_directory: PathBuf,
    pub default_timeout_ms: u64,
    pub max_log_size_bytes: usize,
}
```

**Estate Scanner**:
```rust
pub struct ScanConfig {
    pub max_concurrent_services: usize,
    pub max_concurrent_transforms: usize,
    pub embedding_model: EmbeddingModelConfig,
    pub retry_config: RetryConfig,
}
```

**Common Pattern**: `RetryConfig`, `EmbeddingModelConfig`

```rust
pub struct RetryConfig {
    pub max_retries: usize,
    pub initial_delay_ms: u64,
    pub max_delay_ms: u64,
    pub exponential_backoff: bool,
}

pub struct EmbeddingModelConfig {
    pub model_name: String,
    pub dimension: usize,
    pub batch_size: usize,
}
```

**Decision**:
- ✅ Extract `RetryConfig` to `cloudops-common`
- ✅ Extract `EmbeddingModelConfig` to `cloudops-common`
- ❌ Keep module-specific configs in their respective modules

---

### 6. Error Types (Pattern but module-specific)

**Pattern**: Every module has custom error types

**Storage Service**:
```rust
pub enum StorageError {
    Qdrant(qdrant_client::QdrantError),
    Encryption(EncryptionError),
    // ...
}
```

**Execution Engine**:
```rust
pub enum ExecutionError {
    CommandFailed { exit_code: i32 },
    Timeout { duration: Duration },
    // ...
}
```

**Estate Scanner**:
```rust
pub enum ScanError {
    ExecutionFailed { service: String, error: String },
    AwsCliError { service: String, exit_code: i32 },
    // ...
}
```

**Common Pattern**: Error categories

```rust
pub enum ErrorCategory {
    Execution,
    Parsing,
    Storage,
    Network,
    Configuration,
}
```

**Decision**:
- ✅ Extract `ErrorCategory` to `cloudops-common`
- ❌ Keep module-specific error types in their modules
- ✅ Create common error utilities (e.g., `format_error`, `log_error`)

---

### 7. Result Type Alias (Pattern across all modules)

**Pattern**: Every module has `pub type Result<T> = std::result::Result<T, ModuleError>`

**Decision**:
- ❌ Keep in each module (module-specific error type)
- ✅ Document pattern in common types guide

---

### 8. Metadata Types (Potentially common)

**Execution Engine**:
```rust
pub struct ExecutionMetadata {
    pub source: String,
    pub conversation_id: Option<String>,
    pub tags: HashMap<String, String>,
}
```

**Decision**:
- ⚠️ Consider for common if Request Builder needs similar structure
- 📝 Defer until Request Builder design

---

## Proposed Common Crate Structure

```
cloudops-common/
├── src/
│   ├── lib.rs
│   ├── aws.rs               // AWS-related types
│   │   ├── AWSResource
│   │   ├── IAMPermissions
│   │   ├── UserContext
│   │   ├── ResourceConstraints
│   │   └── Account
│   ├── config.rs            // Common config types
│   │   ├── RetryConfig
│   │   └── EmbeddingModelConfig
│   ├── error.rs             // Common error utilities
│   │   ├── ErrorCategory
│   │   └── Error formatting utilities
│   └── utils.rs             // Common utilities
│       └── (To be determined)
└── Cargo.toml
```

---

## Summary of Common Types

| Type | Modules Using | Move to Common? | Priority |
|------|---------------|-----------------|----------|
| **AWSResource** | Storage, Estate Scanner | ✅ Yes | High |
| **IAMPermissions** | Storage, Estate Scanner | ✅ Yes | High |
| **UserContext** | Storage, Estate Scanner | ✅ Yes | High |
| **ResourceConstraints** | Storage, Estate Scanner | ✅ Yes | High |
| **Account** | Estate Scanner, (Future: Request Builder) | ✅ Yes | High |
| **RetryConfig** | Estate Scanner, (Pattern in all) | ✅ Yes | Medium |
| **EmbeddingModelConfig** | Estate Scanner, Storage | ✅ Yes | Medium |
| **ErrorCategory** | All modules | ✅ Yes | Low |
| Module-specific Configs | Each module | ❌ No | - |
| Module-specific Errors | Each module | ❌ No | - |

---

## Total Common Types

**Core AWS Types**: 5 structs
- `AWSResource`
- `IAMPermissions`
- `UserContext`
- `ResourceConstraints`
- `Account`

**Configuration Types**: 2 structs
- `RetryConfig`
- `EmbeddingModelConfig`

**Error Types**: 1 enum
- `ErrorCategory`

**Total**: 8 types to extract

---

## Dependencies for Common Crate

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
```

**Minimal dependencies** - no Tokio, no Tauri, no framework-specific deps.

---

## Migration Strategy

### Phase 1: Create Common Crate
1. Create `cloudops-common` crate
2. Define all 8 common types
3. Add comprehensive documentation
4. Add JSON schema examples

### Phase 2: Update Module Documentation
1. Storage Service: Reference `cloudops-common::aws::*`
2. Execution Engine: No changes (no common types used)
3. Estate Scanner: Reference `cloudops-common::aws::*` and `cloudops-common::config::*`

### Phase 3: Update Module Dependencies
All modules add:
```toml
[dependencies]
cloudops-common = { path = "../cloudops-common" }
```

---

## Benefits

1. **Single Source of Truth**: `AWSResource` defined once, used everywhere
2. **Type Safety**: Can't accidentally create incompatible types
3. **Easier Refactoring**: Change once, affects all modules
4. **Better Documentation**: Common types documented in one place
5. **Reduced Duplication**: No copy-paste of type definitions
6. **Dependency Management**: Common crate can be versioned independently

---

## Next Steps

1. ✅ Complete this analysis
2. 🔄 Create `cloudops-common` crate design document
3. 🔄 Update all module documentation to reference common types
4. 📝 Design Request Builder (which will also use common types)

---

## Notes

- Common crate should have **zero framework dependencies**
- Should be **pure data structures** (no business logic)
- Should be **backward compatible** (avoid breaking changes)
- Should have **comprehensive tests** (type serialization, validation)