# Estate Scanner Module

**Version**: 1.0.0
**Status**: Design Phase
**Dependencies**: Execution Engine, Storage Service, cloudops-common

> **📦 Common Types**: This module uses shared types from [`cloudops-common`](../common/). Key types include `AWSResource`, `IAMPermissions`, `UserContext`, `ResourceConstraints`, `Account`, `RetryConfig`, `EmbeddingModelConfig`, and `ErrorCategory`. See [Common Types Documentation](../common/README.md) for details.

---

## Overview

The **Estate Scanner** is a thin orchestration module that discovers AWS resources, enriches them with IAM permissions and metadata, and stores them for semantic search. It acts as the bridge between raw AWS CLI output and the structured storage layer.

### Philosophy

> "Estate Scanner is a composer, not a worker. It orchestrates the execution of AWS commands via the Execution Engine and transforms the results into rich, searchable resources via the Storage Service."

### Key Responsibilities

1. **Orchestration**: Coordinate AWS CLI executions via Execution Engine
2. **Transformation**: Parse AWS CLI JSON → `AWSResource` structures
3. **Enrichment**: Discover IAM permissions via AWS IAM APIs
4. **Embedding**: Generate semantic embeddings for vector search
5. **Storage**: Upsert enriched resources to Storage Service

### What Estate Scanner Is NOT

- ❌ Not a command executor (delegates to Execution Engine)
- ❌ Not a data store (delegates to Storage Service)
- ❌ Not a scheduler (scanning is triggered externally)
- ❌ Not AWS-SDK dependent for resource discovery (uses AWS CLI via Execution Engine)

---

## Architecture Principles

### 1. Thin Orchestrator Pattern

Estate Scanner follows the **thin orchestrator** pattern:

```
┌─────────────────────────────────────────────────────────────┐
│                     Estate Scanner                           │
│                   (Thin Orchestrator)                        │
│                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │
│  │   Execute    │→  │  Transform   │→  │    Store     │    │
│  │   AWS CLI    │   │  + Enrich    │   │   Resource   │    │
│  └──────────────┘   └──────────────┘   └──────────────┘    │
│         ↓                   ↓                   ↓            │
└─────────┼───────────────────┼───────────────────┼────────────┘
          ▼                   ▼                   ▼
  Execution Engine    Estate Scanner      Storage Service
   (Heavy Lifter)     (Coordinator)       (Persistence)
```

### 2. Pluggable Service Scanners

Estate Scanner uses a **trait-based architecture** for different AWS services:

```rust
#[async_trait]
pub trait ServiceScanner: Send + Sync {
    /// Service name (e.g., "ec2", "rds", "s3")
    fn service_name(&self) -> &str;

    /// Execute AWS CLI command to discover resources
    async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>>;

    /// Transform raw AWS CLI output to AWSResource
    async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource>;
}
```

**Built-in Scanners**:
- `EC2Scanner` - EC2 instances, volumes, snapshots
- `RDSScanner` - RDS instances, clusters, snapshots
- `S3Scanner` - S3 buckets, objects
- `LambdaScanner` - Lambda functions, layers
- `VPCScanner` - VPCs, subnets, security groups
- (Extensible: add custom scanners via trait)

### 3. IAM Context-Aware

Every resource includes **per-resource IAM permissions** discovered via AWS IAM SimulatePrincipalPolicy:

```rust
pub struct AWSResource {
    // ... resource metadata
    pub iam: IAMPermissions {
        allowed_actions: vec!["ec2:StopInstances", "ec2:StartInstances"],
        denied_actions: vec!["ec2:TerminateInstances"],
        user_context: UserContext {
            username: "john.doe",
            role_arn: "arn:aws:iam::123456789012:role/DeveloperRole",
            session_expires: Some(timestamp),
        },
    },
}
```

**Why This Matters**: When a user asks "stop pg-main1", the AI can immediately check if they have `rds:StopDBInstance` permission for that specific resource.

### 4. Semantic Search Ready

Estate Scanner generates **384-dimensional embeddings** for each resource:

```rust
// Embedding text combines multiple fields for rich search
let embedding_text = format!(
    "{} {} {} {}",
    resource.resource_type,      // "ec2_instance"
    resource.name,                // "web-server-1"
    resource.identifier,          // "i-0123456789abcdef0"
    tags.iter()                   // "env: production app: api"
        .map(|(k, v)| format!("{}: {}", k, v))
        .collect::<Vec<_>>()
        .join(" ")
);

// Generate 384D vector using all-MiniLM-L6-v2
let embedding = embedder.embed(&embedding_text).await?;
```

**Search Examples**:
- "production postgres database" → Finds RDS instances with `env: production` and `engine: postgres`
- "web servers in us-west-2" → Finds EC2 instances tagged with `app: web` in `us-west-2`

---

## Quick Start

### Installation

```toml
[dependencies]
cloudops-estate-scanner = "1.0"
cloudops-execution-engine = "1.0"
cloudops-storage-service = "1.0"
```

### Basic Usage

```rust
use cloudops_estate_scanner::{EstateScanner, ScanConfig, ScanRequest};
use cloudops_execution_engine::ExecutionEngine;
use cloudops_storage_service::StorageService;

#[tokio::main]
async fn main() -> Result<()> {
    // 1. Initialize dependencies
    let execution_engine = ExecutionEngine::new(Default::default());
    let storage_service = StorageService::new(Default::default()).await?;

    // 2. Create Estate Scanner
    let scanner = EstateScanner::new(
        ScanConfig::default(),
        execution_engine,
        storage_service,
    );

    // 3. Scan an AWS account
    let request = ScanRequest {
        accounts: vec![
            Account {
                id: "123456789012".to_string(),
                name: "production-account".to_string(),
                profile: "production".to_string(),
            },
        ],
        regions: vec!["us-east-1".to_string(), "us-west-2".to_string()],
        services: vec!["ec2".to_string(), "rds".to_string(), "s3".to_string()],
        cleanup_stale: true,  // Remove deleted resources
    };

    // 4. Execute scan
    let result = scanner.scan(request).await?;

    println!("Scanned {} resources", result.resources_discovered);
    println!("Upserted {} resources", result.resources_upserted);
    println!("Deleted {} stale resources", result.resources_deleted);

    Ok(())
}
```

---

## Core Concepts

### 1. Scan Context

Every scan operation has a **context** that provides account, region, and credential information:

```rust
pub struct ScanContext {
    pub account_id: String,
    pub account_name: String,
    pub region: String,
    pub profile: Option<String>,
    pub role_arn: Option<String>,
    pub user_context: UserContext,
}
```

### 2. Service Scanners

Each AWS service has a dedicated **scanner** that knows how to:
- Execute the right AWS CLI commands
- Parse service-specific JSON output
- Transform to standardized `AWSResource` format

```rust
// Example: EC2 Scanner
pub struct EC2Scanner {
    execution_engine: Arc<ExecutionEngine>,
}

impl EC2Scanner {
    async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
        // Execute AWS CLI
        let request = ExecutionRequest {
            command: Command::AwsCli {
                service: "ec2".to_string(),
                operation: "describe-instances".to_string(),
                args: vec![
                    "--region".to_string(),
                    context.region.clone(),
                ],
                profile: context.profile.clone(),
                region: Some(context.region.clone()),
            },
            // ...
        };

        let result = self.execution_engine.execute(request).await?;

        // Parse JSON output
        let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

        // Extract instances
        let raw_resources = json["Reservations"]
            .as_array()
            .unwrap_or(&vec![])
            .iter()
            .flat_map(|r| r["Instances"].as_array().unwrap_or(&vec![]))
            .map(|instance| RawResource {
                service: "ec2".to_string(),
                resource_type: "instance".to_string(),
                data: instance.clone(),
            })
            .collect();

        Ok(raw_resources)
    }
}
```

### 3. IAM Discovery

Estate Scanner discovers **per-resource IAM permissions** by calling AWS IAM SimulatePrincipalPolicy:

```rust
pub struct IAMDiscovery {
    iam_client: aws_sdk_iam::Client,
    sts_client: aws_sdk_sts::Client,
}

impl IAMDiscovery {
    /// Discover what actions the current principal can perform on a resource
    pub async fn discover_permissions(&self, resource_arn: &str) -> Result<IAMPermissions> {
        // 1. Get current principal (user or role)
        let identity = self.sts_client.get_caller_identity().send().await?;
        let principal_arn = identity.arn().unwrap();

        // 2. Get relevant actions for resource type
        let actions = self.get_relevant_actions(resource_arn)?;

        // 3. Simulate policy
        let result = self.iam_client
            .simulate_principal_policy()
            .policy_source_arn(principal_arn)
            .resource_arns(resource_arn)
            .set_action_names(Some(actions))
            .send()
            .await?;

        // 4. Parse allowed/denied actions
        let mut allowed = Vec::new();
        let mut denied = Vec::new();

        for eval in result.evaluation_results() {
            match eval.eval_decision() {
                Some(PolicyEvaluationDecisionType::Allowed) => {
                    allowed.push(eval.eval_action_name().unwrap().to_string());
                }
                _ => {
                    denied.push(eval.eval_action_name().unwrap().to_string());
                }
            }
        }

        Ok(IAMPermissions {
            allowed_actions: allowed,
            denied_actions: denied,
            user_context: UserContext {
                username: identity.user_id().unwrap().to_string(),
                role_arn: identity.arn().map(|s| s.to_string()),
                session_expires: None,  // TODO: Extract from session token
            },
        })
    }
}
```

### 4. Embedding Generation

Estate Scanner generates **semantic embeddings** for each resource:

```rust
pub struct EmbeddingGenerator {
    model: SentenceEmbedder,  // all-MiniLM-L6-v2
}

impl EmbeddingGenerator {
    pub async fn generate(&self, resource: &AWSResource) -> Result<Vec<f32>> {
        // Create rich text representation
        let text = format!(
            "{} {} {} {}",
            resource.resource_type,
            resource.name,
            resource.identifier,
            resource.tags.iter()
                .map(|(k, v)| format!("{}: {}", k, v))
                .collect::<Vec<_>>()
                .join(" ")
        );

        // Generate 384D vector
        let embedding = self.model.embed(&text).await?;

        if embedding.len() != 384 {
            return Err(Error::InvalidEmbeddingDimension {
                expected: 384,
                actual: embedding.len(),
            });
        }

        Ok(embedding)
    }
}
```

---

## Scan Flow

### Complete Scan Pipeline

```
1. Scan Request
   ↓
2. For Each Account:
   ├─ For Each Region:
   │  ├─ For Each Service:
   │  │  ├─ Execute AWS CLI (via Execution Engine)
   │  │  ├─ Parse JSON output
   │  │  ├─ Transform to AWSResource
   │  │  ├─ Discover IAM permissions
   │  │  ├─ Generate embedding
   │  │  └─ Upsert to Storage Service
   │  └─ (Parallel execution)
   └─ Cleanup stale resources (if enabled)
   ↓
3. Scan Result
```

### Detailed Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Estate Scanner: Receive Scan Request                         │
│    - accounts: ["123456789012"]                                  │
│    - regions: ["us-west-2"]                                      │
│    - services: ["ec2"]                                           │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Get Service Scanner (EC2Scanner)                             │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Execute AWS CLI via Execution Engine                         │
│    - Command: aws ec2 describe-instances --region us-west-2     │
│    - Result: ExecutionResult with JSON stdout                   │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Parse JSON Output                                            │
│    - Extract instances from JSON                                │
│    - Create RawResource for each instance                       │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Transform Each Instance (Parallel)                           │
│    For each raw instance:                                       │
│    ├─ Parse fields (id, state, type, tags, etc.)               │
│    ├─ Build ARN                                                 │
│    ├─ Extract tags                                              │
│    └─ Create base AWSResource                                   │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Enrich with IAM Permissions (Parallel)                       │
│    For each resource:                                           │
│    ├─ Get current principal ARN (STS)                           │
│    ├─ Simulate policy for resource (IAM)                        │
│    ├─ Parse allowed/denied actions                              │
│    └─ Attach IAMPermissions to resource                         │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Generate Embeddings (Batch)                                  │
│    - Create embedding text for all resources                    │
│    - Generate 384D vectors using all-MiniLM-L6-v2               │
│    - Validate dimensions                                        │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. Batch Upsert to Storage Service                              │
│    - Generate deterministic point IDs                           │
│    - Encrypt sensitive data                                     │
│    - Upsert to Qdrant (creates if new, updates if exists)      │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. Cleanup Stale Resources (if enabled)                         │
│    - Get existing point IDs from Qdrant                         │
│    - Compare with current resource IDs                          │
│    - Delete stale points                                        │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 10. Return Scan Result                                          │
│     - resources_discovered: 150                                 │
│     - resources_upserted: 150                                   │
│     - resources_deleted: 5                                      │
│     - duration: 45.2s                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Performance Characteristics

### Parallelization Strategy

Estate Scanner uses **structured parallelism** for optimal performance:

```rust
// Level 1: Accounts (Sequential - different credentials)
for account in accounts {
    // Level 2: Regions (Parallel - independent)
    let region_tasks: Vec<_> = regions.iter()
        .map(|region| scan_region(account, region))
        .collect();

    futures::future::try_join_all(region_tasks).await?;
}

async fn scan_region(account: &Account, region: &str) {
    // Level 3: Services (Parallel - independent)
    let service_tasks: Vec<_> = services.iter()
        .map(|service| scan_service(account, region, service))
        .collect();

    futures::future::try_join_all(service_tasks).await?;
}

async fn scan_service(account: &Account, region: &str, service: &str) {
    // Level 4: Resources (Parallel - transformation + enrichment)
    let raw_resources = discover_resources(service, region).await?;

    let resource_tasks: Vec<_> = raw_resources.iter()
        .map(|raw| transform_and_enrich(raw, account, region))
        .collect();

    let resources = futures::future::try_join_all(resource_tasks).await?;

    // Level 5: Batch upsert (Single call)
    storage.batch_upsert(resources).await?;
}
```

### Expected Performance

| Operation | Time (1000 resources) | Parallelism |
|-----------|----------------------|-------------|
| **AWS CLI Execution** | 5-10s per service | 10 concurrent services |
| **JSON Parsing** | <100ms total | Single-threaded |
| **Transformation** | 1-2s total | 100 concurrent transforms |
| **IAM Discovery** | 10-20s total | 50 concurrent API calls |
| **Embedding Generation** | 2-5s total | CPU-bound (rayon) |
| **Storage Upsert** | 1-3s total | Batch operation |
| **Total** | ~30-50s | Full parallelism |

### Optimization Tips

1. **Reduce IAM API Calls**: Cache SimulatePrincipalPolicy results per resource type
2. **Batch Embeddings**: Generate embeddings in batches of 100
3. **Concurrency Limits**: Tune based on AWS API rate limits
4. **Incremental Scans**: Only scan changed resources (compare `last_synced` timestamps)

---

## Configuration

```rust
pub struct ScanConfig {
    /// Maximum concurrent service scans per region
    pub max_concurrent_services: usize,  // Default: 10

    /// Maximum concurrent IAM permission discoveries
    pub max_concurrent_iam_checks: usize,  // Default: 50

    /// Embedding model configuration
    pub embedding_model: EmbeddingModelConfig,

    /// Retry configuration for failed operations
    pub retry_config: RetryConfig,

    /// Timeout for AWS CLI operations
    pub aws_cli_timeout_ms: u64,  // Default: 120000 (2 minutes)
}

pub struct EmbeddingModelConfig {
    pub model_name: String,  // Default: "all-MiniLM-L6-v2"
    pub dimension: usize,    // Default: 384
    pub batch_size: usize,   // Default: 100
}

pub struct RetryConfig {
    pub max_retries: usize,           // Default: 3
    pub initial_delay_ms: u64,        // Default: 1000
    pub max_delay_ms: u64,            // Default: 30000
    pub exponential_backoff: bool,    // Default: true
}
```

---

## Error Handling

Estate Scanner provides **graceful degradation** and detailed error reporting:

```rust
pub enum ScanError {
    /// Execution Engine failed to execute AWS CLI
    ExecutionFailed {
        service: String,
        region: String,
        error: ExecutionError,
    },

    /// AWS CLI returned non-zero exit code
    AwsCliError {
        service: String,
        exit_code: i32,
        stderr: String,
    },

    /// Failed to parse AWS CLI JSON output
    ParseError {
        service: String,
        raw_output: String,
        error: serde_json::Error,
    },

    /// IAM permission discovery failed
    IAMDiscoveryFailed {
        resource_arn: String,
        error: aws_sdk_iam::Error,
    },

    /// Embedding generation failed
    EmbeddingError {
        resource_id: String,
        error: EmbeddingError,
    },

    /// Storage Service upsert failed
    StorageFailed {
        resource_id: String,
        error: StorageError,
    },
}

/// Scan result with partial success
pub struct ScanResult {
    pub resources_discovered: usize,
    pub resources_upserted: usize,
    pub resources_deleted: usize,
    pub errors: Vec<ScanError>,  // Non-fatal errors
    pub duration: Duration,
}
```

**Partial Success Example**:
```rust
// Scan 3 services: ec2, rds, s3
let result = scanner.scan(request).await?;

// EC2: Success (50 resources)
// RDS: Failed (network timeout)
// S3: Success (20 resources)

println!("Discovered: {}", result.resources_discovered);  // 70
println!("Errors: {}", result.errors.len());              // 1 (RDS timeout)

// User can retry RDS scan separately
```

---

## Extensibility

### Custom Service Scanners

Implement the `ServiceScanner` trait for custom AWS services:

```rust
use cloudops_estate_scanner::{ServiceScanner, ScanContext, RawResource};

pub struct CustomScanner {
    execution_engine: Arc<ExecutionEngine>,
}

#[async_trait]
impl ServiceScanner for CustomScanner {
    fn service_name(&self) -> &str {
        "custom-service"
    }

    async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
        // Execute AWS CLI command
        let request = ExecutionRequest {
            command: Command::AwsCli {
                service: "custom".to_string(),
                operation: "list-resources".to_string(),
                // ...
            },
            // ...
        };

        let result = self.execution_engine.execute(request).await?;

        // Parse JSON and return raw resources
        // ...
    }

    async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource> {
        // Transform to AWSResource
        // ...
    }
}

// Register custom scanner
scanner.register_scanner(Box::new(CustomScanner::new(execution_engine)));
```

---

## See Also

- [Architecture](architecture.md) - Detailed architecture and design patterns
- [API Reference](api.md) - Complete API documentation
- [Service Scanners](service-scanners.md) - Built-in and custom scanners
- [IAM Discovery](iam-discovery.md) - IAM permission discovery
- [Types](types.md) - Data structures and schemas
- [Usage](usage.md) - Advanced usage patterns

---

## Module Status

| Component | Status |
|-----------|--------|
| Core Architecture | ✅ Design Complete |
| Service Scanners | 🔄 In Progress |
| IAM Discovery | 🔄 In Progress |
| Embedding Generation | 🔄 In Progress |
| API Documentation | 🔄 In Progress |
| Usage Examples | 📝 Pending |

**Next Steps**: Complete architecture document with detailed component design.