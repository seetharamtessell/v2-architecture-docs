# Estate Scanner Types

Complete type definitions for the Estate Scanner module.

> **📦 Common Types**: This module uses shared types from [`cloudops-common`](../common/). The following types are imported from the common crate:
> - **AWS Types**: `AWSResource`, `IAMPermissions`, `UserContext`, `ResourceConstraints`, `Account` - See [Common AWS Types](../common/README.md#aws-resource-types-awsrs)
> - **Config Types**: `RetryConfig`, `EmbeddingModelConfig` - See [Common Config Types](../common/README.md#configuration-types-configrs)
> - **Error Types**: `ErrorCategory` - See [Common Error Utilities](../common/README.md#error-utilities-errorrs)

---

## Table of Contents

1. [Core Types](#core-types)
2. [Configuration Types](#configuration-types)
3. [Request/Response Types](#requestresponse-types)
4. [Error Types](#error-types)
5. [Common Type Imports](#common-type-imports)

---

## Core Types

### ScanRequest

```rust
/// Request to scan AWS resources
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ScanRequest {
    /// AWS accounts to scan
    pub accounts: Vec<Account>,

    /// AWS regions to scan
    pub regions: Vec<String>,

    /// AWS services to scan (e.g., "ec2", "rds", "s3")
    pub services: Vec<String>,

    /// Whether to cleanup stale resources after scan
    #[serde(default = "default_cleanup_stale")]
    pub cleanup_stale: bool,
}

fn default_cleanup_stale() -> bool {
    true
}
```

**JSON Schema**:
```json
{
  "accounts": [
    {
      "id": "123456789012",
      "name": "production-account",
      "profile": "production"
    }
  ],
  "regions": ["us-east-1", "us-west-2"],
  "services": ["ec2", "rds", "s3"],
  "cleanup_stale": true
}
```

### Account

> **📦 Imported from [`cloudops-common`](../common/README.md#account)**

```rust
use cloudops_common::aws::Account;

// Account is defined in cloudops-common/src/aws.rs
// See: docs/02-client/modules/common/README.md#account
```

### ScanResult

```rust
/// Result of a scan operation
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ScanResult {
    /// Total number of resources discovered
    pub resources_discovered: usize,

    /// Number of resources successfully upserted to storage
    pub resources_upserted: usize,

    /// Number of stale resources deleted
    pub resources_deleted: usize,

    /// Non-fatal errors that occurred during scan
    pub errors: Vec<ScanError>,

    /// Total duration of scan
    #[serde(serialize_with = "serialize_duration")]
    pub duration: Duration,
}

fn serialize_duration<S>(duration: &Duration, serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    serializer.serialize_str(&format!("{:.2}s", duration.as_secs_f64()))
}
```

**JSON Schema**:
```json
{
  "resources_discovered": 150,
  "resources_upserted": 148,
  "resources_deleted": 5,
  "errors": [
    {
      "type": "AwsCliError",
      "service": "lambda",
      "exit_code": 1,
      "stderr": "AccessDenied: User is not authorized to perform: lambda:ListFunctions"
    }
  ],
  "duration": "45.32s"
}
```

### ScanContext

```rust
/// Context for a scan operation
#[derive(Debug, Clone)]
pub struct ScanContext {
    /// AWS account ID
    pub account_id: String,

    /// AWS account name
    pub account_name: String,

    /// AWS region
    pub region: String,

    /// AWS CLI profile (optional)
    pub profile: Option<String>,

    /// Role ARN to assume (optional)
    pub role_arn: Option<String>,

    /// Current user context
    pub user_context: UserContext,
}
```

### RawResource

```rust
/// Raw resource data from AWS CLI
#[derive(Debug, Clone)]
pub struct RawResource {
    /// Service name (e.g., "ec2", "rds", "s3")
    pub service: String,

    /// Resource type (e.g., "instance", "db_instance", "bucket")
    pub resource_type: String,

    /// Raw JSON data from AWS CLI
    pub data: serde_json::Value,
}
```

### EnrichedResource

```rust
/// Resource with IAM permissions and embedding
#[derive(Debug, Clone)]
pub struct EnrichedResource {
    /// Transformed AWS resource
    pub resource: AWSResource,

    /// 384D embedding vector
    pub embedding: Vec<f32>,
}
```

---

## Configuration Types

### ScanConfig

```rust
/// Configuration for Estate Scanner
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ScanConfig {
    /// Maximum concurrent service scans per region
    #[serde(default = "default_max_concurrent_services")]
    pub max_concurrent_services: usize,

    /// Maximum concurrent resource transformations
    #[serde(default = "default_max_concurrent_transforms")]
    pub max_concurrent_transforms: usize,

    /// Maximum concurrent IAM permission checks
    #[serde(default = "default_max_concurrent_iam_checks")]
    pub max_concurrent_iam_checks: usize,

    /// Embedding model configuration
    #[serde(default)]
    pub embedding_model: EmbeddingModelConfig,

    /// Retry configuration
    #[serde(default)]
    pub retry_config: RetryConfig,

    /// Timeout for AWS CLI operations (milliseconds)
    #[serde(default = "default_aws_cli_timeout")]
    pub aws_cli_timeout_ms: u64,

    /// Enable IAM permission discovery
    #[serde(default = "default_true")]
    pub enable_iam_discovery: bool,

    /// Enable embedding generation
    #[serde(default = "default_true")]
    pub enable_embeddings: bool,
}

fn default_max_concurrent_services() -> usize { 10 }
fn default_max_concurrent_transforms() -> usize { 100 }
fn default_max_concurrent_iam_checks() -> usize { 50 }
fn default_aws_cli_timeout() -> u64 { 120000 }
fn default_true() -> bool { true }

impl Default for ScanConfig {
    fn default() -> Self {
        Self {
            max_concurrent_services: 10,
            max_concurrent_transforms: 100,
            max_concurrent_iam_checks: 50,
            embedding_model: EmbeddingModelConfig::default(),
            retry_config: RetryConfig::default(),
            aws_cli_timeout_ms: 120000,
            enable_iam_discovery: true,
            enable_embeddings: true,
        }
    }
}
```

### EmbeddingModelConfig

> **📦 Imported from [`cloudops-common`](../common/README.md#embeddingmodelconfig)**

```rust
use cloudops_common::config::EmbeddingModelConfig;

// EmbeddingModelConfig is defined in cloudops-common/src/config.rs
// See: docs/02-client/modules/common/README.md#embeddingmodelconfig
```

### RetryConfig

> **📦 Imported from [`cloudops-common`](../common/README.md#retryconfig)**

```rust
use cloudops_common::config::RetryConfig;

// RetryConfig is defined in cloudops-common/src/config.rs
// See: docs/02-client/modules/common/README.md#retryconfig
```

---

## Request/Response Types

### ServiceScanRequest

```rust
/// Request to scan a specific service
#[derive(Debug, Clone)]
pub struct ServiceScanRequest {
    /// Scan context
    pub context: ScanContext,

    /// Service to scan
    pub service: String,
}
```

### ServiceScanResult

```rust
/// Result of scanning a single service
#[derive(Debug, Clone)]
pub struct ServiceScanResult {
    /// Service name
    pub service: String,

    /// Number of resources discovered
    pub resources_discovered: usize,

    /// Number of resources upserted
    pub resources_upserted: usize,
}
```

### AccountScanResult

```rust
/// Result of scanning a single account
#[derive(Debug, Clone)]
pub struct AccountScanResult {
    /// Number of resources discovered
    pub resources_discovered: usize,

    /// Number of resources upserted
    pub resources_upserted: usize,

    /// Number of resources deleted
    pub resources_deleted: usize,
}

impl Default for AccountScanResult {
    fn default() -> Self {
        Self {
            resources_discovered: 0,
            resources_upserted: 0,
            resources_deleted: 0,
        }
    }
}

impl AccountScanResult {
    pub fn merge(&mut self, other: RegionScanResult) {
        self.resources_discovered += other.resources_discovered;
        self.resources_upserted += other.resources_upserted;
    }
}
```

### RegionScanResult

```rust
/// Result of scanning a single region
#[derive(Debug, Clone)]
pub struct RegionScanResult {
    /// Number of resources discovered
    pub resources_discovered: usize,

    /// Number of resources upserted
    pub resources_upserted: usize,
}

impl Default for RegionScanResult {
    fn default() -> Self {
        Self {
            resources_discovered: 0,
            resources_upserted: 0,
        }
    }
}

impl RegionScanResult {
    pub fn merge(&mut self, other: ServiceScanResult) {
        self.resources_discovered += other.resources_discovered;
        self.resources_upserted += other.resources_upserted;
    }
}
```

---

## Error Types

### ScanError

```rust
/// Errors that can occur during scanning
#[derive(Debug, Clone, Serialize, Deserialize, thiserror::Error)]
#[serde(tag = "type")]
pub enum ScanError {
    /// Execution Engine failed to execute AWS CLI
    #[error("Execution failed for {service} in {region}: {error}")]
    ExecutionFailed {
        service: String,
        region: String,
        error: String,
    },

    /// AWS CLI returned non-zero exit code
    #[error("AWS CLI error for {service}: exit_code={exit_code}, stderr={stderr}")]
    AwsCliError {
        service: String,
        exit_code: i32,
        stderr: String,
    },

    /// Failed to parse AWS CLI JSON output
    #[error("Parse error for {service}: {error}")]
    ParseError {
        service: String,
        raw_output: String,
        error: String,
    },

    /// Missing required field in AWS CLI output
    #[error("Missing field {field} in {service} output")]
    MissingField {
        service: String,
        field: String,
    },

    /// IAM permission discovery failed
    #[error("IAM discovery failed for {resource_arn}: {error}")]
    IAMDiscoveryFailed {
        resource_arn: String,
        error: String,
    },

    /// Embedding generation failed
    #[error("Embedding generation failed for {resource_id}: {error}")]
    EmbeddingError {
        resource_id: String,
        error: String,
    },

    /// Storage Service upsert failed
    #[error("Storage upsert failed for {resource_id}: {error}")]
    StorageFailed {
        resource_id: String,
        error: String,
    },

    /// Service not supported
    #[error("Service {service} is not supported")]
    UnsupportedService {
        service: String,
    },

    /// Invalid ARN format
    #[error("Invalid ARN: {arn}")]
    InvalidArn {
        arn: String,
    },

    /// Timeout waiting for operation
    #[error("Timeout after {timeout_ms}ms: {operation}")]
    Timeout {
        operation: String,
        timeout_ms: u64,
    },
}

impl ScanError {
    /// Get error category
    pub fn category(&self) -> ErrorCategory {
        match self {
            Self::ExecutionFailed { .. } => ErrorCategory::Execution,
            Self::AwsCliError { .. } => ErrorCategory::AwsCli,
            Self::ParseError { .. } | Self::MissingField { .. } => ErrorCategory::Parsing,
            Self::IAMDiscoveryFailed { .. } => ErrorCategory::IAM,
            Self::EmbeddingError { .. } => ErrorCategory::Embedding,
            Self::StorageFailed { .. } => ErrorCategory::Storage,
            Self::UnsupportedService { .. } => ErrorCategory::Parsing,
            Self::InvalidArn { .. } => ErrorCategory::Parsing,
            Self::Timeout { .. } => ErrorCategory::Execution,
        }
    }
}
```

### ErrorCategory

> **📦 Imported from [`cloudops-common`](../common/README.md#errorcategory)**

```rust
use cloudops_common::error::ErrorCategory;

// ErrorCategory is defined in cloudops-common/src/error.rs
// See: docs/02-client/modules/common/README.md#errorcategory
```

### Result Type Alias

```rust
/// Result type for Estate Scanner operations
pub type Result<T> = std::result::Result<T, ScanError>;
```

---

## Common Type Imports

The following types are imported from [`cloudops-common`](../common/) and used throughout the Estate Scanner module:

### AWS Types

```rust
use cloudops_common::aws::{
    AWSResource,
    IAMPermissions,
    UserContext,
    ResourceConstraints,
    Account,
};
```

- **AWSResource** - Complete AWS resource representation with IAM permissions, constraints, and metadata. See [AWSResource documentation](../common/README.md#awsresource)
- **IAMPermissions** - Per-resource IAM permissions (allowed/denied actions + user context). See [IAMPermissions documentation](../common/README.md#iampermissions)
- **UserContext** - AWS user/role context with session info. See [UserContext documentation](../common/README.md#usercontext)
- **ResourceConstraints** - Operational constraints (can_stop, can_start, etc.). See [ResourceConstraints documentation](../common/README.md#resourceconstraints)
- **Account** - AWS account configuration with profile and role info. See [Account documentation](../common/README.md#account)

### Configuration Types

```rust
use cloudops_common::config::{
    RetryConfig,
    EmbeddingModelConfig,
};
```

- **RetryConfig** - Retry configuration (max_retries, backoff strategy). See [RetryConfig documentation](../common/README.md#retryconfig)
- **EmbeddingModelConfig** - Embedding model configuration (model name, dimension, batch size). See [EmbeddingModelConfig documentation](../common/README.md#embeddingmodelconfig)

### Error Types

```rust
use cloudops_common::error::ErrorCategory;
```

- **ErrorCategory** - Common error categories (Execution, AwsCli, Parsing, IAM, Embedding, Storage). See [ErrorCategory documentation](../common/README.md#errorcategory)

---

## Type Conversions

### From ExecutionError to ScanError

```rust
impl From<ExecutionError> for ScanError {
    fn from(error: ExecutionError) -> Self {
        ScanError::ExecutionFailed {
            service: "unknown".to_string(),
            region: "unknown".to_string(),
            error: error.to_string(),
        }
    }
}
```

### From serde_json::Error to ScanError

```rust
impl From<serde_json::Error> for ScanError {
    fn from(error: serde_json::Error) -> Self {
        ScanError::ParseError {
            service: "unknown".to_string(),
            raw_output: String::new(),
            error: error.to_string(),
        }
    }
}
```

---

## JSON Schema Examples

### Complete ScanRequest

```json
{
  "accounts": [
    {
      "id": "123456789012",
      "name": "production-account",
      "profile": "production",
      "role_arn": "arn:aws:iam::123456789012:role/ScannerRole"
    },
    {
      "id": "987654321098",
      "name": "staging-account",
      "profile": "staging"
    }
  ],
  "regions": [
    "us-east-1",
    "us-west-2",
    "eu-west-1"
  ],
  "services": [
    "ec2",
    "rds",
    "s3",
    "lambda",
    "vpc"
  ],
  "cleanup_stale": true
}
```

### Complete ScanConfig

```json
{
  "max_concurrent_services": 10,
  "max_concurrent_transforms": 100,
  "max_concurrent_iam_checks": 50,
  "embedding_model": {
    "model_name": "all-MiniLM-L6-v2",
    "dimension": 384,
    "batch_size": 100
  },
  "retry_config": {
    "max_retries": 3,
    "initial_delay_ms": 1000,
    "max_delay_ms": 30000,
    "exponential_backoff": true
  },
  "aws_cli_timeout_ms": 120000,
  "enable_iam_discovery": true,
  "enable_embeddings": true
}
```

### Complete ScanResult

```json
{
  "resources_discovered": 1523,
  "resources_upserted": 1518,
  "resources_deleted": 12,
  "errors": [
    {
      "type": "AwsCliError",
      "service": "lambda",
      "exit_code": 255,
      "stderr": "AccessDenied: User: arn:aws:iam::123456789012:user/scanner is not authorized to perform: lambda:ListFunctions"
    },
    {
      "type": "IAMDiscoveryFailed",
      "resource_arn": "arn:aws:ec2:us-west-2:123456789012:instance/i-0123456789abcdef0",
      "error": "RateLimitExceeded: Rate exceeded for SimulatePrincipalPolicy"
    }
  ],
  "duration": "127.45s"
}
```

---

## See Also

- [Architecture](architecture.md) - Architecture overview
- [API Reference](api.md) - Complete API documentation
- [Service Scanners](service-scanners.md) - Service scanner implementations