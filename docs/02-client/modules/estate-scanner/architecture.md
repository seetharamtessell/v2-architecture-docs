# Estate Scanner Architecture

**Version**: 1.0.0
**Last Updated**: 2025-10-09

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Core Components](#core-components)
4. [Service Scanner Framework](#service-scanner-framework)
5. [Data Flow](#data-flow)
6. [Concurrency Model](#concurrency-model)
7. [Error Handling Strategy](#error-handling-strategy)
8. [Performance Optimization](#performance-optimization)

---

## Overview

Estate Scanner is a **thin orchestration module** that coordinates between the Execution Engine (for AWS CLI execution) and Storage Service (for persistence). It transforms raw AWS resource data into enriched, searchable resources with IAM permissions and semantic embeddings.

### Design Goals

1. **Separation of Concerns**: Delegate execution and storage to specialized modules
2. **Extensibility**: Easy to add new AWS service scanners via traits
3. **Performance**: Maximize parallelism while respecting AWS API rate limits
4. **Resilience**: Graceful degradation with partial success
5. **Reusability**: Pure Rust crate, framework-agnostic

### Key Principles

- **Thin Coordinator**: No heavy lifting, just orchestration
- **Trait-Based Plugins**: ServiceScanner trait for extensibility
- **Idempotent Operations**: Same scan produces same result
- **Structured Concurrency**: Clear parallelism levels

---

## Architecture Diagram

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                      Estate Scanner                            │
│                   (Thin Orchestrator)                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    Scan Orchestrator                     │  │
│  │  - Coordinates multi-account/region scanning            │  │
│  │  - Manages concurrency and parallelism                  │  │
│  │  - Handles error aggregation                            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                    │
│         ┌─────────────────┼─────────────────┐                 │
│         ▼                 ▼                 ▼                  │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐            │
│  │   EC2    │      │   RDS    │      │   S3     │            │
│  │ Scanner  │      │ Scanner  │      │ Scanner  │  ... more  │
│  └──────────┘      └──────────┘      └──────────┘            │
│         │                 │                 │                  │
│         └─────────────────┼─────────────────┘                 │
│                           ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Resource Enrichment Pipeline                │  │
│  │  1. IAM Discovery    2. Embedding Gen    3. Validation  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                    │
└───────────────────────────┼────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌──────────────┐ ┌─────────────┐ ┌──────────────┐
    │  Execution   │ │ IAM/STS SDK │ │   Storage    │
    │   Engine     │ │   Clients   │ │   Service    │
    └──────────────┘ └─────────────┘ └──────────────┘
```

### Component Layer Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Public API Layer                        │
│  - EstateScanner::scan()                                     │
│  - EstateScanner::scan_service()                             │
│  - EstateScanner::register_scanner()                         │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Orchestration Layer                         │
│  - ScanOrchestrator (coordinates multi-level scanning)      │
│  - ConcurrencyManager (manages parallel execution)          │
│  - ErrorAggregator (collects and categorizes errors)        │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Service Scanner Layer                       │
│  - ServiceScannerRegistry (scanner lookup)                  │
│  - Built-in Scanners: EC2, RDS, S3, Lambda, VPC, etc.      │
│  - Custom Scanners (user-defined)                           │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                 Transformation Layer                         │
│  - RawResourceParser (parse AWS CLI JSON)                   │
│  - AWSResourceBuilder (build standardized resources)        │
│  - TagExtractor, ARNBuilder, MetadataExtractor             │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Enrichment Layer                            │
│  - IAMDiscovery (SimulatePrincipalPolicy)                   │
│  - ConstraintAnalyzer (operational checks)                  │
│  - EmbeddingGenerator (all-MiniLM-L6-v2)                    │
└─────────────────────────────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Integration Layer                          │
│  - ExecutionEngineClient (AWS CLI execution)                │
│  - StorageServiceClient (resource persistence)              │
│  - AWSSDKClients (IAM, STS for permission discovery)        │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. EstateScanner (Main Entry Point)

**Purpose**: Public API and dependency injection container.

```rust
pub struct EstateScanner {
    config: ScanConfig,
    orchestrator: Arc<ScanOrchestrator>,
    scanner_registry: Arc<ServiceScannerRegistry>,
    execution_engine: Arc<ExecutionEngine>,
    storage_service: Arc<StorageService>,
}

impl EstateScanner {
    /// Create new scanner with dependencies
    pub fn new(
        config: ScanConfig,
        execution_engine: ExecutionEngine,
        storage_service: StorageService,
    ) -> Self {
        // Initialize components
        let iam_discovery = IAMDiscovery::new(config.aws_region.clone());
        let embedding_generator = EmbeddingGenerator::new(config.embedding_model.clone());

        // Create scanner registry with built-in scanners
        let scanner_registry = ServiceScannerRegistry::new();
        scanner_registry.register(EC2Scanner::new(execution_engine.clone()));
        scanner_registry.register(RDSScanner::new(execution_engine.clone()));
        scanner_registry.register(S3Scanner::new(execution_engine.clone()));
        // ... more scanners

        // Create orchestrator
        let orchestrator = ScanOrchestrator::new(
            config.clone(),
            scanner_registry.clone(),
            iam_discovery,
            embedding_generator,
            storage_service.clone(),
        );

        Self {
            config,
            orchestrator: Arc::new(orchestrator),
            scanner_registry: Arc::new(scanner_registry),
            execution_engine: Arc::new(execution_engine),
            storage_service: Arc::new(storage_service),
        }
    }

    /// Execute full scan
    pub async fn scan(&self, request: ScanRequest) -> Result<ScanResult> {
        self.orchestrator.scan(request).await
    }

    /// Register custom scanner
    pub fn register_scanner(&self, scanner: Box<dyn ServiceScanner>) {
        self.scanner_registry.register(scanner);
    }
}
```

### 2. ScanOrchestrator (Core Coordination)

**Purpose**: Orchestrates multi-level scanning with parallelism.

```rust
pub struct ScanOrchestrator {
    config: ScanConfig,
    scanner_registry: Arc<ServiceScannerRegistry>,
    iam_discovery: Arc<IAMDiscovery>,
    embedding_generator: Arc<EmbeddingGenerator>,
    storage_service: Arc<StorageService>,
    concurrency_manager: ConcurrencyManager,
    error_aggregator: Arc<Mutex<ErrorAggregator>>,
}

impl ScanOrchestrator {
    /// Execute multi-account, multi-region, multi-service scan
    pub async fn scan(&self, request: ScanRequest) -> Result<ScanResult> {
        let start_time = Instant::now();
        let mut total_discovered = 0;
        let mut total_upserted = 0;
        let mut total_deleted = 0;

        // Level 1: Accounts (Sequential - different credentials)
        for account in &request.accounts {
            let result = self.scan_account(account, &request).await?;
            total_discovered += result.resources_discovered;
            total_upserted += result.resources_upserted;
            total_deleted += result.resources_deleted;
        }

        // Collect errors
        let errors = self.error_aggregator.lock().await.drain();

        Ok(ScanResult {
            resources_discovered: total_discovered,
            resources_upserted: total_upserted,
            resources_deleted: total_deleted,
            errors,
            duration: start_time.elapsed(),
        })
    }

    /// Scan single account across regions and services
    async fn scan_account(
        &self,
        account: &Account,
        request: &ScanRequest,
    ) -> Result<AccountScanResult> {
        // Level 2: Regions (Parallel - independent)
        let region_tasks: Vec<_> = request.regions.iter()
            .map(|region| self.scan_region(account, region, &request.services))
            .collect();

        let results = futures::future::join_all(region_tasks).await;

        // Aggregate results
        let mut account_result = AccountScanResult::default();
        for result in results {
            match result {
                Ok(r) => account_result.merge(r),
                Err(e) => self.error_aggregator.lock().await.add(e),
            }
        }

        // Cleanup stale resources if enabled
        if request.cleanup_stale {
            let deleted = self.cleanup_stale_resources(account).await?;
            account_result.resources_deleted = deleted;
        }

        Ok(account_result)
    }

    /// Scan single region across services
    async fn scan_region(
        &self,
        account: &Account,
        region: &str,
        services: &[String],
    ) -> Result<RegionScanResult> {
        // Create scan context
        let context = ScanContext {
            account_id: account.id.clone(),
            account_name: account.name.clone(),
            region: region.to_string(),
            profile: Some(account.profile.clone()),
            role_arn: None,
            user_context: UserContext::default(),
        };

        // Level 3: Services (Parallel with concurrency limit)
        let service_tasks: Vec<_> = services.iter()
            .map(|service| self.scan_service(&context, service))
            .collect();

        // Use semaphore to limit concurrent service scans
        let results = self.concurrency_manager
            .run_with_limit(service_tasks, self.config.max_concurrent_services)
            .await;

        // Aggregate results
        let mut region_result = RegionScanResult::default();
        for result in results {
            match result {
                Ok(r) => region_result.merge(r),
                Err(e) => self.error_aggregator.lock().await.add(e),
            }
        }

        Ok(region_result)
    }

    /// Scan single service
    async fn scan_service(
        &self,
        context: &ScanContext,
        service: &str,
    ) -> Result<ServiceScanResult> {
        // Get scanner for service
        let scanner = self.scanner_registry.get(service)
            .ok_or(Error::UnsupportedService(service.to_string()))?;

        // Step 1: Discover raw resources (execute AWS CLI)
        let raw_resources = scanner.discover(context).await
            .map_err(|e| Error::DiscoveryFailed {
                service: service.to_string(),
                region: context.region.clone(),
                error: e,
            })?;

        // Step 2: Transform and enrich resources (parallel)
        let resources = self.transform_and_enrich_resources(
            raw_resources,
            context,
            scanner.as_ref(),
        ).await?;

        // Step 3: Batch upsert to storage
        let upserted = self.storage_service
            .batch_upsert_resources(resources.clone())
            .await?;

        Ok(ServiceScanResult {
            service: service.to_string(),
            resources_discovered: resources.len(),
            resources_upserted: upserted,
        })
    }

    /// Transform raw resources and enrich with IAM + embeddings
    async fn transform_and_enrich_resources(
        &self,
        raw_resources: Vec<RawResource>,
        context: &ScanContext,
        scanner: &dyn ServiceScanner,
    ) -> Result<Vec<EnrichedResource>> {
        // Level 4: Resources (Parallel transformation)
        let transform_tasks: Vec<_> = raw_resources.into_iter()
            .map(|raw| {
                let context = context.clone();
                let scanner = scanner.clone();
                async move {
                    // Transform to AWSResource
                    let mut resource = scanner.transform(raw, &context).await?;

                    // Enrich with IAM permissions
                    resource.iam = self.iam_discovery
                        .discover_permissions(&resource.arn)
                        .await?;

                    // Analyze constraints
                    resource.constraints = self.analyze_constraints(&resource).await?;

                    // Generate embedding
                    let embedding = self.embedding_generator
                        .generate(&resource)
                        .await?;

                    Ok::<_, Error>(EnrichedResource { resource, embedding })
                }
            })
            .collect();

        // Execute with concurrency limit
        let results = self.concurrency_manager
            .run_with_limit(transform_tasks, self.config.max_concurrent_transforms)
            .await;

        // Collect successes, log errors
        let mut enriched = Vec::new();
        for result in results {
            match result {
                Ok(r) => enriched.push(r),
                Err(e) => {
                    self.error_aggregator.lock().await.add(e);
                    // Continue with other resources (graceful degradation)
                }
            }
        }

        Ok(enriched)
    }

    /// Cleanup stale resources (deleted from AWS)
    async fn cleanup_stale_resources(&self, account: &Account) -> Result<usize> {
        // Get all point IDs for this account from Storage Service
        let existing_ids = self.storage_service
            .get_all_point_ids_for_account(&account.id)
            .await?;

        // Get current resource IDs from latest scan
        let current_ids = self.storage_service
            .get_recent_resource_ids(&account.id)
            .await?;

        // Find stale IDs
        let stale_ids: Vec<_> = existing_ids
            .into_iter()
            .filter(|id| !current_ids.contains(id))
            .collect();

        if !stale_ids.is_empty() {
            self.storage_service
                .delete_resources(&stale_ids)
                .await?;
        }

        Ok(stale_ids.len())
    }
}
```

### 3. ServiceScannerRegistry

**Purpose**: Manages service scanner plugins.

```rust
pub struct ServiceScannerRegistry {
    scanners: Arc<RwLock<HashMap<String, Arc<dyn ServiceScanner>>>>,
}

impl ServiceScannerRegistry {
    pub fn new() -> Self {
        Self {
            scanners: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    /// Register a scanner
    pub fn register(&self, scanner: Box<dyn ServiceScanner>) {
        let service_name = scanner.service_name().to_string();
        self.scanners.write().unwrap().insert(service_name, scanner.into());
    }

    /// Get scanner for service
    pub fn get(&self, service: &str) -> Option<Arc<dyn ServiceScanner>> {
        self.scanners.read().unwrap().get(service).cloned()
    }

    /// List all registered services
    pub fn list_services(&self) -> Vec<String> {
        self.scanners.read().unwrap().keys().cloned().collect()
    }
}
```

### 4. IAMDiscovery

**Purpose**: Discover per-resource IAM permissions.

```rust
pub struct IAMDiscovery {
    iam_client: aws_sdk_iam::Client,
    sts_client: aws_sdk_sts::Client,
    cache: Arc<RwLock<PermissionCache>>,
}

impl IAMDiscovery {
    /// Discover IAM permissions for a resource
    pub async fn discover_permissions(&self, resource_arn: &str) -> Result<IAMPermissions> {
        // Check cache first
        if let Some(cached) = self.cache.read().await.get(resource_arn) {
            return Ok(cached);
        }

        // Get current principal
        let identity = self.sts_client.get_caller_identity().send().await?;
        let principal_arn = identity.arn().unwrap();

        // Get relevant actions for resource type
        let actions = self.get_relevant_actions(resource_arn)?;

        // Simulate policy
        let result = self.iam_client
            .simulate_principal_policy()
            .policy_source_arn(principal_arn)
            .resource_arns(resource_arn)
            .set_action_names(Some(actions))
            .send()
            .await?;

        // Parse results
        let mut allowed = Vec::new();
        let mut denied = Vec::new();

        for eval in result.evaluation_results() {
            let action = eval.eval_action_name().unwrap();
            match eval.eval_decision() {
                Some(PolicyEvaluationDecisionType::Allowed) => {
                    allowed.push(action.to_string());
                }
                _ => {
                    denied.push(action.to_string());
                }
            }
        }

        let permissions = IAMPermissions {
            allowed_actions: allowed,
            denied_actions: denied,
            user_context: UserContext {
                username: identity.user_id().unwrap().to_string(),
                role_arn: identity.arn().map(|s| s.to_string()),
                session_expires: None,
            },
        };

        // Cache result
        self.cache.write().await.insert(resource_arn.to_string(), permissions.clone());

        Ok(permissions)
    }

    /// Get relevant actions for resource type
    fn get_relevant_actions(&self, resource_arn: &str) -> Result<Vec<String>> {
        // Parse service from ARN
        let parts: Vec<&str> = resource_arn.split(':').collect();
        let service = parts.get(2).ok_or(Error::InvalidArn)?;

        // Return service-specific actions
        Ok(match *service {
            "ec2" => ACTION_SETS.ec2.clone(),
            "rds" => ACTION_SETS.rds.clone(),
            "s3" => ACTION_SETS.s3.clone(),
            "lambda" => ACTION_SETS.lambda.clone(),
            _ => vec![],
        })
    }
}

/// Predefined action sets for common AWS services
struct ActionSets {
    ec2: Vec<String>,
    rds: Vec<String>,
    s3: Vec<String>,
    lambda: Vec<String>,
}

lazy_static! {
    static ref ACTION_SETS: ActionSets = ActionSets {
        ec2: vec![
            "ec2:DescribeInstances".to_string(),
            "ec2:StartInstances".to_string(),
            "ec2:StopInstances".to_string(),
            "ec2:RebootInstances".to_string(),
            "ec2:TerminateInstances".to_string(),
            "ec2:ModifyInstanceAttribute".to_string(),
        ],
        rds: vec![
            "rds:DescribeDBInstances".to_string(),
            "rds:StartDBInstance".to_string(),
            "rds:StopDBInstance".to_string(),
            "rds:RebootDBInstance".to_string(),
            "rds:DeleteDBInstance".to_string(),
            "rds:CreateDBSnapshot".to_string(),
        ],
        s3: vec![
            "s3:ListBucket".to_string(),
            "s3:GetObject".to_string(),
            "s3:PutObject".to_string(),
            "s3:DeleteObject".to_string(),
            "s3:DeleteBucket".to_string(),
        ],
        lambda: vec![
            "lambda:GetFunction".to_string(),
            "lambda:InvokeFunction".to_string(),
            "lambda:UpdateFunctionCode".to_string(),
            "lambda:UpdateFunctionConfiguration".to_string(),
            "lambda:DeleteFunction".to_string(),
        ],
    };
}
```

### 5. EmbeddingGenerator

**Purpose**: Generate semantic embeddings for resources.

```rust
pub struct EmbeddingGenerator {
    model: SentenceEmbedder,
    config: EmbeddingModelConfig,
}

impl EmbeddingGenerator {
    pub fn new(config: EmbeddingModelConfig) -> Self {
        let model = SentenceEmbedder::new(&config.model_name).unwrap();
        Self { model, config }
    }

    /// Generate embedding for single resource
    pub async fn generate(&self, resource: &AWSResource) -> Result<Vec<f32>> {
        let text = self.create_embedding_text(resource);
        let embedding = self.model.embed(&text).await?;

        // Validate dimension
        if embedding.len() != self.config.dimension {
            return Err(Error::InvalidEmbeddingDimension {
                expected: self.config.dimension,
                actual: embedding.len(),
            });
        }

        Ok(embedding)
    }

    /// Generate embeddings for batch of resources
    pub async fn generate_batch(&self, resources: &[AWSResource]) -> Result<Vec<Vec<f32>>> {
        let texts: Vec<_> = resources.iter()
            .map(|r| self.create_embedding_text(r))
            .collect();

        let embeddings = self.model.embed_batch(&texts).await?;

        // Validate dimensions
        for embedding in &embeddings {
            if embedding.len() != self.config.dimension {
                return Err(Error::InvalidEmbeddingDimension {
                    expected: self.config.dimension,
                    actual: embedding.len(),
                });
            }
        }

        Ok(embeddings)
    }

    /// Create rich text for embedding
    fn create_embedding_text(&self, resource: &AWSResource) -> String {
        format!(
            "{} {} {} {}",
            resource.resource_type,
            resource.name,
            resource.identifier,
            resource.tags.iter()
                .map(|(k, v)| format!("{}: {}", k, v))
                .collect::<Vec<_>>()
                .join(" ")
        )
    }
}
```

### 6. ConcurrencyManager

**Purpose**: Manage concurrency limits and parallelism.

```rust
pub struct ConcurrencyManager {
    semaphore: Arc<Semaphore>,
}

impl ConcurrencyManager {
    pub fn new(max_concurrent: usize) -> Self {
        Self {
            semaphore: Arc::new(Semaphore::new(max_concurrent)),
        }
    }

    /// Run tasks with concurrency limit
    pub async fn run_with_limit<T, F>(
        &self,
        tasks: Vec<F>,
        limit: usize,
    ) -> Vec<Result<T>>
    where
        F: Future<Output = Result<T>> + Send + 'static,
        T: Send + 'static,
    {
        let semaphore = Arc::new(Semaphore::new(limit));
        let mut handles = Vec::new();

        for task in tasks {
            let permit = semaphore.clone().acquire_owned().await.unwrap();
            let handle = tokio::spawn(async move {
                let result = task.await;
                drop(permit);
                result
            });
            handles.push(handle);
        }

        // Wait for all tasks
        let mut results = Vec::new();
        for handle in handles {
            results.push(handle.await.unwrap());
        }

        results
    }
}
```

### 7. ErrorAggregator

**Purpose**: Collect and categorize errors during scanning.

```rust
pub struct ErrorAggregator {
    errors: Vec<ScanError>,
}

impl ErrorAggregator {
    pub fn new() -> Self {
        Self { errors: Vec::new() }
    }

    pub fn add(&mut self, error: ScanError) {
        self.errors.push(error);
    }

    pub fn drain(&mut self) -> Vec<ScanError> {
        std::mem::take(&mut self.errors)
    }

    pub fn error_count(&self) -> usize {
        self.errors.len()
    }

    pub fn errors_by_category(&self) -> HashMap<ErrorCategory, Vec<&ScanError>> {
        let mut categorized = HashMap::new();
        for error in &self.errors {
            categorized
                .entry(error.category())
                .or_insert_with(Vec::new)
                .push(error);
        }
        categorized
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum ErrorCategory {
    Execution,      // AWS CLI execution failed
    AwsCli,         // AWS CLI returned error
    Parsing,        // JSON parsing failed
    IAM,            // IAM discovery failed
    Embedding,      // Embedding generation failed
    Storage,        // Storage upsert failed
}
```

---

## Service Scanner Framework

### ServiceScanner Trait

The core extensibility mechanism for supporting different AWS services.

```rust
#[async_trait]
pub trait ServiceScanner: Send + Sync {
    /// Service name (e.g., "ec2", "rds", "s3")
    fn service_name(&self) -> &str;

    /// Execute AWS CLI command to discover resources
    ///
    /// Returns raw AWS CLI JSON output wrapped in RawResource
    async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>>;

    /// Transform raw AWS CLI output to AWSResource
    ///
    /// Parses service-specific JSON and builds standardized AWSResource
    async fn transform(
        &self,
        raw: RawResource,
        context: &ScanContext,
    ) -> Result<AWSResource>;

    /// Optional: Custom validation logic
    async fn validate(&self, resource: &AWSResource) -> Result<()> {
        Ok(()) // Default: no validation
    }
}
```

### RawResource Type

```rust
/// Raw resource data from AWS CLI
#[derive(Debug, Clone)]
pub struct RawResource {
    /// Service name
    pub service: String,

    /// Resource type (e.g., "instance", "db_instance", "bucket")
    pub resource_type: String,

    /// Raw JSON data from AWS CLI
    pub data: serde_json::Value,
}
```

### ScanContext Type

```rust
/// Context for a scan operation
#[derive(Debug, Clone)]
pub struct ScanContext {
    pub account_id: String,
    pub account_name: String,
    pub region: String,
    pub profile: Option<String>,
    pub role_arn: Option<String>,
    pub user_context: UserContext,
}
```

---

## Data Flow

### Complete Scan Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Scan Request                                                  │
│    {                                                              │
│      accounts: [Account],                                        │
│      regions: [String],                                          │
│      services: [String],                                         │
│      cleanup_stale: bool                                         │
│    }                                                              │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. For Each Account (Sequential)                                │
│    - Different AWS credentials per account                      │
│    - Load profile from ~/.aws/credentials                       │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. For Each Region (Parallel - max 10)                          │
│    - Independent regions can scan concurrently                  │
│    - Create ScanContext per region                              │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. For Each Service (Parallel - max 10)                         │
│    - Get ServiceScanner from registry                           │
│    - Execute AWS CLI via Execution Engine                       │
│    - Parse JSON output                                          │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. For Each Raw Resource (Parallel - max 100)                   │
│    a. Transform: raw JSON → AWSResource                         │
│    b. Enrich: discover IAM permissions                          │
│    c. Analyze: compute constraints                              │
│    d. Embed: generate 384D vector                               │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Batch Upsert to Storage Service                              │
│    - Generate deterministic point IDs                           │
│    - Encrypt sensitive data                                     │
│    - Upsert to Qdrant (batch operation)                         │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Cleanup Stale Resources (if enabled)                         │
│    - Compare Qdrant IDs vs current scan IDs                     │
│    - Delete stale points                                        │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. Return ScanResult                                             │
│    {                                                              │
│      resources_discovered: usize,                                │
│      resources_upserted: usize,                                  │
│      resources_deleted: usize,                                   │
│      errors: Vec<ScanError>,                                     │
│      duration: Duration                                          │
│    }                                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Concurrency Model

### Parallelism Levels

```rust
// Level 1: Accounts (Sequential)
for account in accounts {
    // Different credentials, must be sequential

    // Level 2: Regions (Parallel - semaphore limit)
    let region_tasks = regions.map(|r| async {
        // Level 3: Services (Parallel - semaphore limit)
        let service_tasks = services.map(|s| async {
            // Execute AWS CLI
            let raw_resources = discover(s).await?;

            // Level 4: Resources (Parallel - semaphore limit)
            let resource_tasks = raw_resources.map(|r| async {
                // Transform + Enrich (CPU + I/O bound)
                transform_and_enrich(r).await
            });

            futures::future::try_join_all(resource_tasks).await
        });

        futures::future::try_join_all(service_tasks).await
    });

    futures::future::try_join_all(region_tasks).await?;
}
```

### Concurrency Limits

| Level | Operation | Default Limit | Reason |
|-------|-----------|--------------|---------|
| 1 | Accounts | 1 (sequential) | Different AWS credentials |
| 2 | Regions | 10 concurrent | Network I/O bound |
| 3 | Services | 10 concurrent | AWS API rate limits |
| 4 | Resources | 100 concurrent | CPU + I/O bound |
| 5 | IAM Checks | 50 concurrent | AWS IAM API rate limits |

### Rate Limiting Strategy

```rust
pub struct RateLimiter {
    permits: Arc<Semaphore>,
    refill_rate: Duration,
}

impl RateLimiter {
    pub async fn acquire(&self) -> RateLimitPermit {
        let permit = self.permits.acquire().await.unwrap();

        // Schedule refill
        let permits = self.permits.clone();
        let refill_rate = self.refill_rate;
        tokio::spawn(async move {
            tokio::time::sleep(refill_rate).await;
            drop(permit);
        });

        RateLimitPermit { _permit: permit }
    }
}
```

---

## Error Handling Strategy

### Error Types

```rust
#[derive(Debug, thiserror::Error)]
pub enum ScanError {
    #[error("Execution failed for {service} in {region}: {error}")]
    ExecutionFailed {
        service: String,
        region: String,
        error: ExecutionError,
    },

    #[error("AWS CLI error for {service}: exit_code={exit_code}, stderr={stderr}")]
    AwsCliError {
        service: String,
        exit_code: i32,
        stderr: String,
    },

    #[error("Parse error for {service}: {error}")]
    ParseError {
        service: String,
        raw_output: String,
        error: serde_json::Error,
    },

    #[error("IAM discovery failed for {resource_arn}: {error}")]
    IAMDiscoveryFailed {
        resource_arn: String,
        error: String,
    },

    #[error("Embedding generation failed for {resource_id}: {error}")]
    EmbeddingError {
        resource_id: String,
        error: String,
    },

    #[error("Storage upsert failed for {resource_id}: {error}")]
    StorageFailed {
        resource_id: String,
        error: String,
    },
}
```

### Graceful Degradation

**Principle**: Continue scanning even if some operations fail.

```rust
// Example: Continue scanning if one service fails
for service in services {
    match scan_service(service).await {
        Ok(result) => {
            total_discovered += result.resources_discovered;
        }
        Err(e) => {
            error_aggregator.add(ScanError::from(e));
            // Continue with other services
            continue;
        }
    }
}
```

### Retry Strategy

```rust
pub struct RetryPolicy {
    max_retries: usize,
    initial_delay: Duration,
    max_delay: Duration,
    exponential_backoff: bool,
}

impl RetryPolicy {
    pub async fn execute<F, T, E>(&self, mut f: F) -> Result<T, E>
    where
        F: FnMut() -> Future<Output = Result<T, E>>,
    {
        let mut attempt = 0;
        let mut delay = self.initial_delay;

        loop {
            match f().await {
                Ok(result) => return Ok(result),
                Err(e) if attempt >= self.max_retries => return Err(e),
                Err(_) => {
                    attempt += 1;
                    tokio::time::sleep(delay).await;

                    if self.exponential_backoff {
                        delay = (delay * 2).min(self.max_delay);
                    }
                }
            }
        }
    }
}
```

---

## Performance Optimization

### 1. Batch Operations

```rust
// Batch embedding generation (100 at a time)
let embeddings = embedding_generator
    .generate_batch(&resources[..100])
    .await?;

// Batch storage upsert
storage_service
    .batch_upsert_resources(resources)
    .await?;
```

### 2. Caching

```rust
// Cache IAM permissions per resource type
pub struct PermissionCache {
    cache: Arc<RwLock<HashMap<String, (IAMPermissions, Instant)>>>,
    ttl: Duration,
}

impl PermissionCache {
    pub fn get(&self, arn: &str) -> Option<IAMPermissions> {
        let cache = self.cache.read().unwrap();
        cache.get(arn).and_then(|(perms, timestamp)| {
            if timestamp.elapsed() < self.ttl {
                Some(perms.clone())
            } else {
                None
            }
        })
    }
}
```

### 3. Incremental Scanning

```rust
// Only scan resources modified since last scan
pub async fn incremental_scan(
    &self,
    request: ScanRequest,
    last_scan_time: DateTime<Utc>,
) -> Result<ScanResult> {
    // Add filter to AWS CLI commands
    let filter = format!("--filters Name=launch-time,Values={}/..",
        last_scan_time.to_rfc3339());

    // Execute scan with filter
    self.scan_with_filter(request, filter).await
}
```

### 4. Resource Pooling

```rust
// Pool of AWS SDK clients
pub struct ClientPool {
    iam_clients: Pool<aws_sdk_iam::Client>,
    sts_clients: Pool<aws_sdk_sts::Client>,
}
```

---

## See Also

- [Service Scanners](service-scanners.md) - Detailed scanner implementations
- [IAM Discovery](iam-discovery.md) - IAM permission discovery
- [API Reference](api.md) - Complete API documentation
- [Types](types.md) - Data structures and schemas