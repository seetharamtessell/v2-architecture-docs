# Module Interaction Analysis

**Date**: 2025-10-09
**Status**: Module Sync Analysis Complete ✅

---

## Purpose

This document analyzes the interaction between **Execution Engine**, **Estate Scanner**, and **Storage Service** modules to ensure proper data flow and interface compatibility.

---

## Module Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        Estate Scanner                            │
│                     (Thin Orchestrator)                          │
│                                                                   │
│  ┌─────────────┐      ┌──────────────┐      ┌────────────┐     │
│  │   AWS CLI   │ ───▶ │ Transformation│ ───▶ │  Storage   │     │
│  │  Execution  │      │   + Enrichment│      │   Upsert   │     │
│  └─────────────┘      └──────────────┘      └────────────┘     │
│         │                     │                      │           │
└─────────│─────────────────────│──────────────────────│───────────┘
          ▼                     ▼                      ▼
    Execution Engine      Estate Scanner         Storage Service
```

---

## 1. Execution Engine Output Format

### ExecutionResult Structure

```rust
pub struct ExecutionResult {
    pub id: Uuid,
    pub status: ExecutionStatus,  // "completed" | "failed" | "cancelled"
    pub exit_code: Option<i32>,
    pub stdout: String,  // ◀── Contains AWS CLI JSON output
    pub stderr: String,
    pub duration: String,
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub log_path: Option<PathBuf>,
}
```

### Example: AWS CLI Output (stdout)

**Command**: `aws ec2 describe-instances --region us-west-2`

**stdout** contains:
```json
{
  "Reservations": [
    {
      "Instances": [
        {
          "InstanceId": "i-0123456789abcdef0",
          "InstanceType": "t3.medium",
          "State": {
            "Name": "running"
          },
          "PrivateIpAddress": "10.0.1.10",
          "PublicIpAddress": "54.123.45.67",
          "LaunchTime": "2025-10-08T10:30:00.000Z",
          "Tags": [
            {"Key": "Name", "Value": "web-server-1"},
            {"Key": "env", "Value": "production"},
            {"Key": "app", "Value": "api"}
          ],
          "VpcId": "vpc-12345678",
          "SubnetId": "subnet-abcdef12",
          "SecurityGroups": [
            {"GroupId": "sg-12345678", "GroupName": "web-sg"}
          ]
        }
      ]
    }
  ]
}
```

---

## 2. Storage Service Input Format

### AWSResource Structure

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AWSResource {
    // Plain text (for filtering/indexing)
    pub resource_type: String,        // "ec2_instance"
    pub identifier: String,           // "i-0123456789abcdef0"
    pub account_id: String,           // "123456789012"
    pub account_name: String,         // "production-account"
    pub region: String,               // "us-west-2"
    pub service: String,              // "ec2"
    pub arn: String,                  // Full ARN
    pub name: String,                 // From tags or identifier
    pub state: String,                // "running"
    pub tags: HashMap<String, String>,// All resource tags
    pub last_synced: DateTime<Utc>,  // Timestamp

    // Encrypted sensitive data
    pub iam: IAMPermissions,          // Resource-specific permissions
    pub constraints: ResourceConstraints,
    pub metadata: serde_json::Value,  // Service-specific metadata
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IAMPermissions {
    pub allowed_actions: Vec<String>,  // ["ec2:StopInstances", "ec2:StartInstances", ...]
    pub denied_actions: Vec<String>,   // ["ec2:TerminateInstances"]
    pub user_context: UserContext,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserContext {
    pub username: String,              // "john.doe"
    pub role_arn: Option<String>,      // "arn:aws:iam::123456789012:role/DeveloperRole"
    pub session_expires: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceConstraints {
    pub can_stop: bool,
    pub can_start: bool,
    pub can_delete: bool,
    pub has_backups: bool,
    pub has_dependencies: bool,
    pub custom_constraints: HashMap<String, serde_json::Value>,
}
```

### Qdrant Point ID Format

**Deterministic ID**: `{account_id}-{region}-{identifier}`

Example: `"123456789012-us-west-2-i-0123456789abcdef0"`

**Why Deterministic?**
- Same resource = same point ID
- Upsert updates existing point (no duplicates)
- Idempotent sync operations

---

## 3. Transformation Requirements

Estate Scanner must transform `ExecutionResult.stdout` (AWS CLI JSON) → `AWSResource` (Storage Service format).

### Transformation Steps

```rust
impl EstateScanner {
    /// Transform AWS CLI JSON output to AWSResource
    async fn transform_ec2_instance(
        &self,
        raw: &serde_json::Value,
        account_id: &str,
        account_name: &str,
        region: &str,
    ) -> Result<AWSResource> {
        // 1. Parse AWS CLI JSON
        let instance_id = raw["InstanceId"].as_str()
            .ok_or(Error::MissingField("InstanceId"))?;
        let state = raw["State"]["Name"].as_str()
            .ok_or(Error::MissingField("State.Name"))?;
        let instance_type = raw["InstanceType"].as_str()
            .unwrap_or("unknown");

        // 2. Extract tags
        let tags = self.parse_tags(raw["Tags"].as_array())?;
        let name = tags.get("Name")
            .cloned()
            .unwrap_or_else(|| instance_id.to_string());

        // 3. Build ARN
        let arn = format!(
            "arn:aws:ec2:{}:{}:instance/{}",
            region, account_id, instance_id
        );

        // 4. Discover IAM permissions (via SimulatePrincipalPolicy)
        let iam = self.discover_iam_permissions(&arn).await?;

        // 5. Analyze constraints
        let constraints = self.analyze_constraints(raw, &iam).await?;

        // 6. Build metadata
        let metadata = json!({
            "instance_type": instance_type,
            "private_ip": raw["PrivateIpAddress"],
            "public_ip": raw["PublicIpAddress"],
            "launch_time": raw["LaunchTime"],
            "vpc_id": raw["VpcId"],
            "subnet_id": raw["SubnetId"],
            "security_groups": raw["SecurityGroups"],
        });

        // 7. Create AWSResource
        Ok(AWSResource {
            resource_type: "ec2_instance".to_string(),
            identifier: instance_id.to_string(),
            account_id: account_id.to_string(),
            account_name: account_name.to_string(),
            region: region.to_string(),
            service: "ec2".to_string(),
            arn,
            name,
            state: state.to_string(),
            tags,
            iam,
            constraints,
            metadata,
            last_synced: Utc::now(),
        })
    }
}
```

---

## 4. IAM Permission Discovery

Estate Scanner must discover IAM permissions for each resource using AWS IAM SimulatePrincipalPolicy API.

### IAM Discovery Flow

```
1. Get Current User/Role ARN
   ↓
2. For Each Resource:
   ├─ Generate list of relevant actions (e.g., ec2:StopInstances, ec2:StartInstances)
   ├─ Call SimulatePrincipalPolicy API
   ├─ Parse allowed/denied actions
   └─ Create IAMPermissions struct
   ↓
3. Attach IAMPermissions to AWSResource
```

### Implementation

```rust
impl EstateScanner {
    /// Discover IAM permissions for a resource
    async fn discover_iam_permissions(&self, resource_arn: &str) -> Result<IAMPermissions> {
        // 1. Get current principal ARN (user or role)
        let caller_identity = self.sts_client.get_caller_identity().send().await?;
        let principal_arn = caller_identity.arn()
            .ok_or(Error::MissingPrincipalArn)?;

        // 2. Determine relevant actions based on resource type
        let actions = self.get_relevant_actions(resource_arn)?;

        // 3. Simulate policy
        let result = self.iam_client
            .simulate_principal_policy()
            .policy_source_arn(principal_arn)
            .resource_arns(resource_arn)
            .set_action_names(Some(actions.clone()))
            .send()
            .await?;

        // 4. Parse results
        let mut allowed_actions = Vec::new();
        let mut denied_actions = Vec::new();

        for eval_result in result.evaluation_results() {
            let action = eval_result.eval_action_name()
                .ok_or(Error::MissingActionName)?;

            match eval_result.eval_decision() {
                Some(PolicyEvaluationDecisionType::Allowed) => {
                    allowed_actions.push(action.to_string());
                }
                Some(PolicyEvaluationDecisionType::ExplicitDeny) |
                Some(PolicyEvaluationDecisionType::ImplicitDeny) => {
                    denied_actions.push(action.to_string());
                }
                _ => {}
            }
        }

        // 5. Get user context
        let user_context = self.get_user_context(&caller_identity).await?;

        Ok(IAMPermissions {
            allowed_actions,
            denied_actions,
            user_context,
        })
    }

    /// Get relevant actions for resource type
    fn get_relevant_actions(&self, resource_arn: &str) -> Result<Vec<String>> {
        // Parse ARN to determine service
        // Example: "arn:aws:ec2:us-west-2:123456789012:instance/i-123"
        let parts: Vec<&str> = resource_arn.split(':').collect();
        let service = parts.get(2).ok_or(Error::InvalidArn)?;

        // Return service-specific actions
        match *service {
            "ec2" => Ok(vec![
                "ec2:DescribeInstances".to_string(),
                "ec2:StartInstances".to_string(),
                "ec2:StopInstances".to_string(),
                "ec2:RebootInstances".to_string(),
                "ec2:TerminateInstances".to_string(),
                "ec2:ModifyInstanceAttribute".to_string(),
            ]),
            "rds" => Ok(vec![
                "rds:DescribeDBInstances".to_string(),
                "rds:StartDBInstance".to_string(),
                "rds:StopDBInstance".to_string(),
                "rds:RebootDBInstance".to_string(),
                "rds:DeleteDBInstance".to_string(),
                "rds:CreateDBSnapshot".to_string(),
            ]),
            "s3" => Ok(vec![
                "s3:ListBucket".to_string(),
                "s3:GetObject".to_string(),
                "s3:PutObject".to_string(),
                "s3:DeleteObject".to_string(),
                "s3:DeleteBucket".to_string(),
            ]),
            _ => Err(Error::UnsupportedService(service.to_string())),
        }
    }
}
```

---

## 5. Constraint Analysis

Estate Scanner must analyze resource constraints (operational checks).

### Constraint Analysis Flow

```rust
impl EstateScanner {
    /// Analyze resource constraints
    async fn analyze_constraints(
        &self,
        raw: &serde_json::Value,
        iam: &IAMPermissions,
    ) -> Result<ResourceConstraints> {
        // Check IAM permissions
        let can_stop = iam.allowed_actions.contains(&"ec2:StopInstances".to_string());
        let can_start = iam.allowed_actions.contains(&"ec2:StartInstances".to_string());
        let can_delete = iam.allowed_actions.contains(&"ec2:TerminateInstances".to_string());

        // Check resource state
        let state = raw["State"]["Name"].as_str().unwrap_or("unknown");
        let can_stop = can_stop && (state == "running");
        let can_start = can_start && (state == "stopped");

        // Check dependencies (example: attached volumes)
        let has_dependencies = raw["BlockDeviceMappings"]
            .as_array()
            .map(|arr| !arr.is_empty())
            .unwrap_or(false);

        // Check backups (example: snapshots exist)
        let has_backups = self.check_backups_exist(raw).await?;

        Ok(ResourceConstraints {
            can_stop,
            can_start,
            can_delete: can_delete && !has_dependencies,
            has_backups,
            has_dependencies,
            custom_constraints: HashMap::new(),
        })
    }
}
```

---

## 6. Embedding Generation

Estate Scanner must generate 384D embeddings for semantic search.

### Embedding Generation Flow

```rust
impl EstateScanner {
    /// Generate embedding for resource
    async fn generate_embedding(&self, resource: &AWSResource) -> Result<Vec<f32>> {
        // 1. Create embedding text (combines multiple fields)
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

        // Example: "ec2_instance web-server-1 i-0123456789abcdef0 env: production app: api"

        // 2. Generate embedding using model (e.g., all-MiniLM-L6-v2)
        let embedding = self.embedding_model.embed(&text).await?;

        // 3. Validate dimension (must be 384 for all-MiniLM-L6-v2)
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

**Embedding Model**: `all-MiniLM-L6-v2` (384 dimensions)

---

## 7. Complete Data Flow Example

### Step-by-Step Flow: EC2 Instance Scan

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Estate Scanner: Execute AWS CLI                              │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Execution Engine                                                 │
│                                                                   │
│ let request = ExecutionRequest {                                 │
│     command: Command::AwsCli {                                   │
│         service: "ec2".to_string(),                              │
│         operation: "describe-instances".to_string(),             │
│         args: vec!["--region".to_string(), "us-west-2".to_string()],│
│         profile: Some("production".to_string()),                 │
│         region: Some("us-west-2".to_string()),                   │
│     },                                                            │
│     ...                                                           │
│ };                                                                │
│                                                                   │
│ let execution_id = engine.execute(request).await?;               │
│ let result = engine.get_result(execution_id).await?;             │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ ExecutionResult                                                  │
│                                                                   │
│ {                                                                 │
│   "status": "completed",                                          │
│   "exit_code": 0,                                                 │
│   "stdout": "{\"Reservations\": [{\"Instances\": [...]}]}",      │
│   "stderr": ""                                                    │
│ }                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Estate Scanner: Parse JSON                                    │
│                                                                   │
│ let aws_response: serde_json::Value = serde_json::from_str(     │
│     &result.stdout                                                │
│ )?;                                                               │
│                                                                   │
│ let instances = aws_response["Reservations"]                     │
│     .as_array()                                                   │
│     .unwrap()                                                     │
│     .iter()                                                       │
│     .flat_map(|r| r["Instances"].as_array().unwrap())            │
│     .collect::<Vec<_>>();                                         │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Estate Scanner: Transform Each Instance                       │
│                                                                   │
│ for raw_instance in instances {                                  │
│     // a. Transform to AWSResource                               │
│     let resource = self.transform_ec2_instance(                  │
│         raw_instance,                                             │
│         "123456789012",  // account_id                            │
│         "production-account",  // account_name                    │
│         "us-west-2",  // region                                   │
│     ).await?;                                                     │
│                                                                   │
│     // b. Discover IAM permissions                               │
│     let iam = self.discover_iam_permissions(&resource.arn).await?;│
│     resource.iam = iam;                                           │
│                                                                   │
│     // c. Analyze constraints                                    │
│     let constraints = self.analyze_constraints(                  │
│         raw_instance,                                             │
│         &resource.iam                                             │
│     ).await?;                                                     │
│     resource.constraints = constraints;                           │
│                                                                   │
│     // d. Generate embedding                                     │
│     let embedding = self.generate_embedding(&resource).await?;   │
│                                                                   │
│     // e. Upsert to Storage Service                              │
│     storage.upsert_resource(resource, embedding).await?;         │
│ }                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Storage Service: Upsert to Qdrant                             │
│                                                                   │
│ pub async fn upsert_resource(                                    │
│     &self,                                                        │
│     resource: AWSResource,                                        │
│     embedding: Vec<f32>,                                          │
│ ) -> Result<()> {                                                 │
│     // a. Generate deterministic point ID                        │
│     let point_id = format!("{}-{}-{}",                           │
│         resource.account_id,                                      │
│         resource.region,                                          │
│         resource.identifier                                       │
│     );                                                            │
│     // Example: "123456789012-us-west-2-i-0123456789abcdef0"     │
│                                                                   │
│     // b. Encrypt sensitive data                                 │
│     let encrypted_data = self.encrypt(json!({                    │
│         "identifier": resource.identifier,                        │
│         "arn": resource.arn,                                      │
│         "name": resource.name,                                    │
│         "iam": resource.iam,                                      │
│         "constraints": resource.constraints,                      │
│         "metadata": resource.metadata,                            │
│     }))?;                                                         │
│                                                                   │
│     // c. Create Qdrant point                                    │
│     let point = Point {                                           │
│         id: point_id,                                             │
│         vector: embedding,  // 384D vector                        │
│         payload: json!({                                          │
│             // Plain text (indexed)                               │
│             "resource_type": resource.resource_type,              │
│             "account_id": resource.account_id,                    │
│             "region": resource.region,                            │
│             "state": resource.state,                              │
│             "last_synced": Utc::now().timestamp(),                │
│             "tags": resource.tags,                                │
│                                                                   │
│             // Encrypted (sensitive)                              │
│             "encrypted_data": encrypted_data,                     │
│         }),                                                        │
│     };                                                            │
│                                                                   │
│     // d. Upsert to Qdrant (creates if new, updates if exists)   │
│     self.qdrant.upsert_points("aws_estate", vec![point]).await?; │
│                                                                   │
│     Ok(())                                                        │
│ }                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Interface Contracts

### Estate Scanner → Execution Engine

**Input**: `ExecutionRequest`
```rust
ExecutionRequest {
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
    timeout_ms: Some(120000),
    metadata: ExecutionMetadata {
        source: "estate_scanner".to_string(),
        tags: HashMap::from([
            ("scan_type".to_string(), "ec2".to_string()),
            ("account".to_string(), "production".to_string()),
        ]),
    },
}
```

**Output**: `ExecutionResult`
```rust
ExecutionResult {
    id: <uuid>,
    status: ExecutionStatus::Completed,
    exit_code: Some(0),
    stdout: "{\"Reservations\": [...]}",  // ◀── AWS CLI JSON
    stderr: "",
    duration: "2.5s",
    started_at: <timestamp>,
    completed_at: Some(<timestamp>),
    log_path: Some("/tmp/cloudops-executions/<uuid>.log"),
}
```

**Contract**:
- ✅ Estate Scanner expects `stdout` to contain valid JSON
- ✅ Estate Scanner checks `status == Completed` and `exit_code == 0`
- ✅ Estate Scanner handles errors if `status == Failed`

---

### Estate Scanner → Storage Service

**Input**: `AWSResource` + `Vec<f32>` (embedding)
```rust
// AWSResource
AWSResource {
    resource_type: "ec2_instance",
    identifier: "i-0123456789abcdef0",
    account_id: "123456789012",
    account_name: "production-account",
    region: "us-west-2",
    service: "ec2",
    arn: "arn:aws:ec2:us-west-2:123456789012:instance/i-0123456789abcdef0",
    name: "web-server-1",
    state: "running",
    tags: {
        "Name": "web-server-1",
        "env": "production",
        "app": "api",
    },
    iam: IAMPermissions {
        allowed_actions: vec![
            "ec2:DescribeInstances",
            "ec2:StopInstances",
            "ec2:StartInstances",
        ],
        denied_actions: vec!["ec2:TerminateInstances"],
        user_context: UserContext {
            username: "john.doe",
            role_arn: Some("arn:aws:iam::123456789012:role/DeveloperRole"),
            session_expires: Some(<timestamp>),
        },
    },
    constraints: ResourceConstraints {
        can_stop: true,
        can_start: false,  // Not stopped
        can_delete: false,  // Denied by IAM
        has_backups: true,
        has_dependencies: true,
        custom_constraints: HashMap::new(),
    },
    metadata: json!({
        "instance_type": "t3.medium",
        "private_ip": "10.0.1.10",
        "public_ip": "54.123.45.67",
        "vpc_id": "vpc-12345678",
    }),
    last_synced: Utc::now(),
}

// Embedding
vec![0.123, 0.456, 0.789, ...] // 384 dimensions
```

**Output**: `Result<()>`

**Contract**:
- ✅ Storage Service expects `embedding.len() == 384`
- ✅ Storage Service generates deterministic point ID: `{account_id}-{region}-{identifier}`
- ✅ Storage Service encrypts sensitive fields (IAM, constraints, metadata)
- ✅ Storage Service uses upsert (creates if new, updates if exists)

---

## 9. Error Handling

### Execution Engine Errors

```rust
match engine.execute(request).await {
    Ok(execution_id) => {
        let result = engine.get_result(execution_id).await?;

        match result.status {
            ExecutionStatus::Completed if result.exit_code == Some(0) => {
                // Success: Parse stdout
                let json: serde_json::Value = serde_json::from_str(&result.stdout)?;
                // ... transform
            }
            ExecutionStatus::Completed => {
                // Non-zero exit code
                error!("AWS CLI failed: exit_code={}, stderr={}",
                    result.exit_code.unwrap_or(-1), result.stderr);
                // Retry or skip
            }
            ExecutionStatus::Failed => {
                error!("Execution failed: {}", result.stderr);
                // Retry or skip
            }
            ExecutionStatus::Cancelled => {
                warn!("Execution cancelled");
                // Skip
            }
            _ => {
                warn!("Unexpected status: {:?}", result.status);
            }
        }
    }
    Err(e) => {
        error!("Failed to execute AWS CLI: {}", e);
        // Retry or skip
    }
}
```

### Storage Service Errors

```rust
match storage.upsert_resource(resource, embedding).await {
    Ok(_) => {
        info!("Resource upserted: {}", resource.identifier);
    }
    Err(StorageError::Encryption(e)) => {
        error!("Encryption failed for {}: {}", resource.identifier, e);
        // Retry or skip
    }
    Err(StorageError::Qdrant(e)) => {
        error!("Qdrant upsert failed for {}: {}", resource.identifier, e);
        // Retry (network error?)
    }
    Err(e) => {
        error!("Unexpected error for {}: {}", resource.identifier, e);
    }
}
```

---

## 10. Performance Considerations

### Batch Operations

**Problem**: Processing 1000 EC2 instances one-by-one is slow.

**Solution**: Batch transformations and upserts.

```rust
impl EstateScanner {
    async fn scan_ec2_instances(&self, account: &Account, region: &str) -> Result<()> {
        // 1. Execute AWS CLI (single call)
        let result = self.execute_aws_cli("ec2", "describe-instances", region).await?;

        // 2. Parse all instances
        let raw_instances = self.parse_ec2_response(&result.stdout)?;

        // 3. Transform in parallel
        let mut resources = Vec::new();
        for raw in raw_instances {
            let resource = self.transform_ec2_instance(
                &raw,
                &account.id,
                &account.name,
                region,
            ).await?;
            resources.push(resource);
        }

        // 4. Generate embeddings in parallel
        let embeddings = futures::future::try_join_all(
            resources.iter().map(|r| self.generate_embedding(r))
        ).await?;

        // 5. Batch upsert to Storage Service
        storage.batch_upsert_resources(resources, embeddings).await?;

        Ok(())
    }
}
```

### Parallel Scanning

**Problem**: Scanning multiple services/regions sequentially is slow.

**Solution**: Parallel scanning with concurrency limit.

```rust
impl EstateScanner {
    async fn scan_account(&self, account: &Account) -> Result<()> {
        let regions = vec!["us-east-1", "us-west-2", "eu-west-1"];
        let services = vec!["ec2", "rds", "s3", "lambda"];

        // Create scan tasks
        let tasks: Vec<_> = regions.iter()
            .flat_map(|region| {
                services.iter().map(move |service| {
                    self.scan_service(account, service, region)
                })
            })
            .collect();

        // Execute with concurrency limit (e.g., 10 concurrent scans)
        let results = futures::stream::iter(tasks)
            .buffer_unordered(10)
            .collect::<Vec<_>>()
            .await;

        // Check for errors
        for result in results {
            if let Err(e) = result {
                error!("Scan failed: {}", e);
            }
        }

        Ok(())
    }
}
```

---

## 11. Data Consistency

### Idempotent Upserts

**Guarantee**: Running the same scan multiple times produces the same result (no duplicates).

**How**:
- Execution Engine: Same command can be executed multiple times
- Estate Scanner: Same transformation always produces same AWSResource
- Storage Service: Deterministic point IDs ensure upsert updates existing points

### Stale Data Cleanup

**Problem**: Resources deleted from AWS still exist in Qdrant.

**Solution**: Compare Qdrant point IDs with current AWS resource IDs.

```rust
impl EstateScanner {
    async fn sync_with_cleanup(&self, account: &Account, region: &str) -> Result<()> {
        // 1. Get existing point IDs from Qdrant
        let existing_ids = storage.get_all_point_ids_for_region(
            &account.id,
            region
        ).await?;

        // 2. Scan current AWS resources
        let current_resources = self.scan_all_services(account, region).await?;

        // 3. Upsert current resources
        storage.batch_upsert_resources(current_resources.clone()).await?;

        // 4. Find stale IDs (in Qdrant but not in AWS)
        let current_ids: HashSet<_> = current_resources.iter()
            .map(|r| format!("{}-{}-{}", r.account_id, r.region, r.identifier))
            .collect();

        let stale_ids: Vec<_> = existing_ids
            .into_iter()
            .filter(|id| !current_ids.contains(id))
            .collect();

        // 5. Delete stale resources
        if !stale_ids.is_empty() {
            storage.delete_resources(&stale_ids).await?;
            info!("Deleted {} stale resources", stale_ids.len());
        }

        Ok(())
    }
}
```

---

## 12. Summary

### Module Responsibilities

| Module | Responsibility |
|--------|---------------|
| **Execution Engine** | Execute AWS CLI commands, return JSON output |
| **Estate Scanner** | Transform JSON → AWSResource, discover IAM, generate embeddings |
| **Storage Service** | Encrypt sensitive data, generate point IDs, upsert to Qdrant |

### Data Flow

```
AWS CLI Command
    ↓
Execution Engine (ExecutionResult with JSON stdout)
    ↓
Estate Scanner Transformation
    ├─ Parse JSON
    ├─ Discover IAM permissions
    ├─ Analyze constraints
    └─ Generate embeddings (384D)
    ↓
Storage Service Upsert
    ├─ Generate deterministic point ID
    ├─ Encrypt sensitive data
    └─ Upsert to Qdrant
```

### Interface Contracts

✅ **Execution Engine → Estate Scanner**:
- `stdout` contains valid JSON
- `status == Completed` and `exit_code == 0` for success

✅ **Estate Scanner → Storage Service**:
- `AWSResource` with all required fields
- `embedding.len() == 384`
- Deterministic IDs prevent duplicates

### Key Design Principles

1. **Idempotency**: Same scan produces same result
2. **Deterministic IDs**: Prevent duplicates
3. **Batch Operations**: Optimize performance
4. **Parallel Scanning**: Scan multiple services/regions concurrently
5. **Error Handling**: Graceful degradation, retry logic
6. **Data Consistency**: Cleanup stale resources

---

## Next Steps

1. **Design Estate Scanner Module**: Complete architecture with service scanners
2. **Define Service Scanner Traits**: Modular architecture for EC2, RDS, S3, Lambda, etc.
3. **IAM Integration**: Implement SimulatePrincipalPolicy API calls
4. **Embedding Model**: Integrate `all-MiniLM-L6-v2` (384D)
5. **Orchestration**: Multi-account, multi-region coordination

---

**Status**: ✅ Module interaction analysis complete. Ready to design Estate Scanner module.