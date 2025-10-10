# Service Scanners

Detailed design specifications for AWS service scanners.

---

## Table of Contents

1. [Overview](#overview)
2. [Scanner Pattern](#scanner-pattern)
3. [Built-in Scanners](#built-in-scanners)
4. [Custom Scanners](#custom-scanners)

---

## Overview

Service scanners are pluggable modules that know how to:
1. Execute AWS CLI commands for a specific service
2. Parse service-specific JSON output
3. Transform to standardized `AWSResource` format

Each scanner implements the `ServiceScanner` trait.

---

## Scanner Pattern

### ServiceScanner Trait

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

### Common Pattern

All scanners follow this pattern:

```rust
pub struct XxxScanner {
    execution_engine: Arc<ExecutionEngine>,
}

impl XxxScanner {
    pub fn new(execution_engine: Arc<ExecutionEngine>) -> Self {
        Self { execution_engine }
    }
}

#[async_trait]
impl ServiceScanner for XxxScanner {
    fn service_name(&self) -> &str {
        "xxx"
    }

    async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
        // 1. Build AWS CLI command
        // 2. Execute via Execution Engine
        // 3. Parse JSON output
        // 4. Return RawResource array
    }

    async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource> {
        // 1. Parse service-specific fields
        // 2. Extract tags
        // 3. Build ARN
        // 4. Create AWSResource
    }
}
```

---

## Built-in Scanners

### 1. EC2Scanner

**Discovers**: EC2 instances

#### Discovery

```rust
async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    // AWS CLI command
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
        timeout_ms: Some(120000),
        metadata: ExecutionMetadata {
            source: "estate_scanner".to_string(),
            tags: HashMap::from([
                ("service".to_string(), "ec2".to_string()),
                ("operation".to_string(), "describe-instances".to_string()),
            ]),
        },
        ..Default::default()
    };

    // Execute
    let execution_id = self.execution_engine.execute(request).await?;
    let result = self.execution_engine.get_result(execution_id).await?;

    // Check status
    if result.status != ExecutionStatus::Completed || result.exit_code != Some(0) {
        return Err(Error::AwsCliError {
            service: "ec2".to_string(),
            exit_code: result.exit_code.unwrap_or(-1),
            stderr: result.stderr,
        });
    }

    // Parse JSON
    let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

    // Extract instances
    let raw_resources = json["Reservations"]
        .as_array()
        .ok_or(Error::MissingField("Reservations"))?
        .iter()
        .flat_map(|reservation| {
            reservation["Instances"]
                .as_array()
                .unwrap_or(&vec![])
                .iter()
        })
        .map(|instance| RawResource {
            service: "ec2".to_string(),
            resource_type: "instance".to_string(),
            data: instance.clone(),
        })
        .collect();

    Ok(raw_resources)
}
```

#### Transformation

```rust
async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource> {
    let data = &raw.data;

    // Extract fields
    let instance_id = data["InstanceId"]
        .as_str()
        .ok_or(Error::MissingField("InstanceId"))?;
    let state = data["State"]["Name"]
        .as_str()
        .ok_or(Error::MissingField("State.Name"))?;
    let instance_type = data["InstanceType"]
        .as_str()
        .unwrap_or("unknown");

    // Extract tags
    let tags = self.parse_tags(data["Tags"].as_array())?;
    let name = tags.get("Name")
        .cloned()
        .unwrap_or_else(|| instance_id.to_string());

    // Build ARN
    let arn = format!(
        "arn:aws:ec2:{}:{}:instance/{}",
        context.region,
        context.account_id,
        instance_id
    );

    // Build metadata
    let metadata = json!({
        "instance_type": instance_type,
        "private_ip": data["PrivateIpAddress"],
        "public_ip": data["PublicIpAddress"],
        "launch_time": data["LaunchTime"],
        "vpc_id": data["VpcId"],
        "subnet_id": data["SubnetId"],
        "security_groups": data["SecurityGroups"],
        "availability_zone": data["Placement"]["AvailabilityZone"],
    });

    Ok(AWSResource {
        resource_type: "ec2_instance".to_string(),
        identifier: instance_id.to_string(),
        account_id: context.account_id.clone(),
        account_name: context.account_name.clone(),
        region: context.region.clone(),
        service: "ec2".to_string(),
        arn,
        name,
        state: state.to_string(),
        tags,
        iam: IAMPermissions::default(),  // Will be enriched later
        constraints: ResourceConstraints::default(),  // Will be enriched later
        metadata,
        last_synced: Utc::now(),
    })
}

fn parse_tags(&self, tags: Option<&Vec<serde_json::Value>>) -> Result<HashMap<String, String>> {
    let mut tag_map = HashMap::new();
    if let Some(tags) = tags {
        for tag in tags {
            if let (Some(key), Some(value)) = (
                tag["Key"].as_str(),
                tag["Value"].as_str()
            ) {
                tag_map.insert(key.to_string(), value.to_string());
            }
        }
    }
    Ok(tag_map)
}
```

---

### 2. RDSScanner

**Discovers**: RDS database instances

#### Discovery

```rust
async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    let request = ExecutionRequest {
        command: Command::AwsCli {
            service: "rds".to_string(),
            operation: "describe-db-instances".to_string(),
            args: vec![
                "--region".to_string(),
                context.region.clone(),
            ],
            profile: context.profile.clone(),
            region: Some(context.region.clone()),
        },
        timeout_ms: Some(120000),
        ..Default::default()
    };

    let execution_id = self.execution_engine.execute(request).await?;
    let result = self.execution_engine.get_result(execution_id).await?;

    if result.status != ExecutionStatus::Completed || result.exit_code != Some(0) {
        return Err(Error::AwsCliError {
            service: "rds".to_string(),
            exit_code: result.exit_code.unwrap_or(-1),
            stderr: result.stderr,
        });
    }

    let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

    let raw_resources = json["DBInstances"]
        .as_array()
        .ok_or(Error::MissingField("DBInstances"))?
        .iter()
        .map(|instance| RawResource {
            service: "rds".to_string(),
            resource_type: "db_instance".to_string(),
            data: instance.clone(),
        })
        .collect();

    Ok(raw_resources)
}
```

#### Transformation

```rust
async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource> {
    let data = &raw.data;

    let db_instance_identifier = data["DBInstanceIdentifier"]
        .as_str()
        .ok_or(Error::MissingField("DBInstanceIdentifier"))?;
    let state = data["DBInstanceStatus"]
        .as_str()
        .ok_or(Error::MissingField("DBInstanceStatus"))?;
    let engine = data["Engine"]
        .as_str()
        .unwrap_or("unknown");

    // Extract tags
    let tags = self.parse_tags(data["TagList"].as_array())?;
    let name = tags.get("Name")
        .cloned()
        .unwrap_or_else(|| db_instance_identifier.to_string());

    // Build ARN
    let arn = data["DBInstanceArn"]
        .as_str()
        .ok_or(Error::MissingField("DBInstanceArn"))?
        .to_string();

    // Build metadata
    let metadata = json!({
        "engine": engine,
        "engine_version": data["EngineVersion"],
        "instance_class": data["DBInstanceClass"],
        "allocated_storage": data["AllocatedStorage"],
        "storage_type": data["StorageType"],
        "multi_az": data["MultiAZ"],
        "publicly_accessible": data["PubliclyAccessible"],
        "vpc_id": data["DBSubnetGroup"]["VpcId"],
        "availability_zone": data["AvailabilityZone"],
        "endpoint": data["Endpoint"]["Address"],
        "port": data["Endpoint"]["Port"],
    });

    Ok(AWSResource {
        resource_type: "rds_instance".to_string(),
        identifier: db_instance_identifier.to_string(),
        account_id: context.account_id.clone(),
        account_name: context.account_name.clone(),
        region: context.region.clone(),
        service: "rds".to_string(),
        arn,
        name,
        state: state.to_string(),
        tags,
        iam: IAMPermissions::default(),
        constraints: ResourceConstraints::default(),
        metadata,
        last_synced: Utc::now(),
    })
}
```

---

### 3. S3Scanner

**Discovers**: S3 buckets

#### Discovery

```rust
async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    // Note: S3 is global, only scan once (not per-region)
    if context.region != "us-east-1" {
        return Ok(vec![]);  // Skip for other regions
    }

    let request = ExecutionRequest {
        command: Command::AwsCli {
            service: "s3api".to_string(),
            operation: "list-buckets".to_string(),
            args: vec![],
            profile: context.profile.clone(),
            region: Some("us-east-1".to_string()),
        },
        timeout_ms: Some(60000),
        ..Default::default()
    };

    let execution_id = self.execution_engine.execute(request).await?;
    let result = self.execution_engine.get_result(execution_id).await?;

    if result.status != ExecutionStatus::Completed || result.exit_code != Some(0) {
        return Err(Error::AwsCliError {
            service: "s3".to_string(),
            exit_code: result.exit_code.unwrap_or(-1),
            stderr: result.stderr,
        });
    }

    let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

    let raw_resources = json["Buckets"]
        .as_array()
        .ok_or(Error::MissingField("Buckets"))?
        .iter()
        .map(|bucket| RawResource {
            service: "s3".to_string(),
            resource_type: "bucket".to_string(),
            data: bucket.clone(),
        })
        .collect();

    Ok(raw_resources)
}
```

#### Transformation

```rust
async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource> {
    let data = &raw.data;

    let bucket_name = data["Name"]
        .as_str()
        .ok_or(Error::MissingField("Name"))?;

    // Get bucket location (region)
    let bucket_region = self.get_bucket_location(bucket_name, context).await?;

    // Get bucket tags
    let tags = self.get_bucket_tags(bucket_name, context).await?;

    // Build ARN
    let arn = format!("arn:aws:s3:::{}", bucket_name);

    // Get additional metadata
    let metadata = self.get_bucket_metadata(bucket_name, context).await?;

    Ok(AWSResource {
        resource_type: "s3_bucket".to_string(),
        identifier: bucket_name.to_string(),
        account_id: context.account_id.clone(),
        account_name: context.account_name.clone(),
        region: bucket_region,
        service: "s3".to_string(),
        arn,
        name: bucket_name.to_string(),
        state: "available".to_string(),
        tags,
        iam: IAMPermissions::default(),
        constraints: ResourceConstraints::default(),
        metadata,
        last_synced: Utc::now(),
    })
}

async fn get_bucket_location(&self, bucket: &str, context: &ScanContext) -> Result<String> {
    let request = ExecutionRequest {
        command: Command::AwsCli {
            service: "s3api".to_string(),
            operation: "get-bucket-location".to_string(),
            args: vec![
                "--bucket".to_string(),
                bucket.to_string(),
            ],
            profile: context.profile.clone(),
            region: Some("us-east-1".to_string()),
        },
        ..Default::default()
    };

    let execution_id = self.execution_engine.execute(request).await?;
    let result = self.execution_engine.get_result(execution_id).await?;

    let json: serde_json::Value = serde_json::from_str(&result.stdout)?;
    let location = json["LocationConstraint"]
        .as_str()
        .unwrap_or("us-east-1");

    Ok(if location.is_empty() { "us-east-1" } else { location }.to_string())
}
```

---

### 4. LambdaScanner

**Discovers**: Lambda functions

#### Discovery

```rust
async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    let request = ExecutionRequest {
        command: Command::AwsCli {
            service: "lambda".to_string(),
            operation: "list-functions".to_string(),
            args: vec![
                "--region".to_string(),
                context.region.clone(),
            ],
            profile: context.profile.clone(),
            region: Some(context.region.clone()),
        },
        timeout_ms: Some(60000),
        ..Default::default()
    };

    let execution_id = self.execution_engine.execute(request).await?;
    let result = self.execution_engine.get_result(execution_id).await?;

    if result.status != ExecutionStatus::Completed || result.exit_code != Some(0) {
        return Err(Error::AwsCliError {
            service: "lambda".to_string(),
            exit_code: result.exit_code.unwrap_or(-1),
            stderr: result.stderr,
        });
    }

    let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

    let raw_resources = json["Functions"]
        .as_array()
        .ok_or(Error::MissingField("Functions"))?
        .iter()
        .map(|function| RawResource {
            service: "lambda".to_string(),
            resource_type: "function".to_string(),
            data: function.clone(),
        })
        .collect();

    Ok(raw_resources)
}
```

#### Transformation

```rust
async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource> {
    let data = &raw.data;

    let function_name = data["FunctionName"]
        .as_str()
        .ok_or(Error::MissingField("FunctionName"))?;
    let arn = data["FunctionArn"]
        .as_str()
        .ok_or(Error::MissingField("FunctionArn"))?;

    // Get function tags
    let tags = self.get_function_tags(arn, context).await?;

    let metadata = json!({
        "runtime": data["Runtime"],
        "handler": data["Handler"],
        "code_size": data["CodeSize"],
        "memory_size": data["MemorySize"],
        "timeout": data["Timeout"],
        "role": data["Role"],
        "last_modified": data["LastModified"],
        "version": data["Version"],
    });

    Ok(AWSResource {
        resource_type: "lambda_function".to_string(),
        identifier: function_name.to_string(),
        account_id: context.account_id.clone(),
        account_name: context.account_name.clone(),
        region: context.region.clone(),
        service: "lambda".to_string(),
        arn: arn.to_string(),
        name: function_name.to_string(),
        state: data["State"].as_str().unwrap_or("Active").to_string(),
        tags,
        iam: IAMPermissions::default(),
        constraints: ResourceConstraints::default(),
        metadata,
        last_synced: Utc::now(),
    })
}
```

---

### 5. VPCScanner

**Discovers**: VPCs, Subnets, Security Groups

#### Discovery

```rust
async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    let mut raw_resources = Vec::new();

    // Discover VPCs
    raw_resources.extend(self.discover_vpcs(context).await?);

    // Discover Subnets
    raw_resources.extend(self.discover_subnets(context).await?);

    // Discover Security Groups
    raw_resources.extend(self.discover_security_groups(context).await?);

    Ok(raw_resources)
}

async fn discover_vpcs(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    let request = ExecutionRequest {
        command: Command::AwsCli {
            service: "ec2".to_string(),
            operation: "describe-vpcs".to_string(),
            args: vec![
                "--region".to_string(),
                context.region.clone(),
            ],
            profile: context.profile.clone(),
            region: Some(context.region.clone()),
        },
        ..Default::default()
    };

    let execution_id = self.execution_engine.execute(request).await?;
    let result = self.execution_engine.get_result(execution_id).await?;

    let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

    let raw_resources = json["Vpcs"]
        .as_array()
        .ok_or(Error::MissingField("Vpcs"))?
        .iter()
        .map(|vpc| RawResource {
            service: "vpc".to_string(),
            resource_type: "vpc".to_string(),
            data: vpc.clone(),
        })
        .collect();

    Ok(raw_resources)
}
```

---

## Custom Scanners

### Creating a Custom Scanner

```rust
use cloudops_estate_scanner::{ServiceScanner, ScanContext, RawResource, AWSResource};

pub struct CustomServiceScanner {
    execution_engine: Arc<ExecutionEngine>,
}

#[async_trait]
impl ServiceScanner for CustomServiceScanner {
    fn service_name(&self) -> &str {
        "my-custom-service"
    }

    async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
        // 1. Build AWS CLI command
        let request = ExecutionRequest {
            command: Command::AwsCli {
                service: "custom-service".to_string(),
                operation: "list-resources".to_string(),
                args: vec![
                    "--region".to_string(),
                    context.region.clone(),
                ],
                profile: context.profile.clone(),
                region: Some(context.region.clone()),
            },
            ..Default::default()
        };

        // 2. Execute
        let execution_id = self.execution_engine.execute(request).await?;
        let result = self.execution_engine.get_result(execution_id).await?;

        // 3. Parse JSON
        let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

        // 4. Extract resources
        let raw_resources = json["Resources"]
            .as_array()
            .ok_or(Error::MissingField("Resources"))?
            .iter()
            .map(|resource| RawResource {
                service: "custom-service".to_string(),
                resource_type: "custom_resource".to_string(),
                data: resource.clone(),
            })
            .collect();

        Ok(raw_resources)
    }

    async fn transform(&self, raw: RawResource, context: &ScanContext) -> Result<AWSResource> {
        let data = &raw.data;

        // Extract fields specific to your service
        let identifier = data["ResourceId"]
            .as_str()
            .ok_or(Error::MissingField("ResourceId"))?;

        // Build ARN
        let arn = format!(
            "arn:aws:custom:{}:{}:resource/{}",
            context.region,
            context.account_id,
            identifier
        );

        // Create AWSResource
        Ok(AWSResource {
            resource_type: "custom_resource".to_string(),
            identifier: identifier.to_string(),
            account_id: context.account_id.clone(),
            account_name: context.account_name.clone(),
            region: context.region.clone(),
            service: "custom-service".to_string(),
            arn,
            name: identifier.to_string(),
            state: "active".to_string(),
            tags: HashMap::new(),
            iam: IAMPermissions::default(),
            constraints: ResourceConstraints::default(),
            metadata: json!({}),
            last_synced: Utc::now(),
        })
    }
}
```

### Registering Custom Scanner

```rust
// Create scanner
let custom_scanner = CustomServiceScanner::new(execution_engine.clone());

// Register with Estate Scanner
estate_scanner.register_scanner(Box::new(custom_scanner));

// Now you can scan custom service
let request = ScanRequest {
    accounts: vec![account],
    regions: vec!["us-west-2".to_string()],
    services: vec!["my-custom-service".to_string()],  // Your custom service
    cleanup_stale: true,
};

let result = estate_scanner.scan(request).await?;
```

---

## Scanner Best Practices

### 1. Error Handling

```rust
async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    // Execute AWS CLI
    let result = match self.execute_aws_cli(context).await {
        Ok(r) => r,
        Err(e) => {
            error!("AWS CLI execution failed: {}", e);
            return Err(Error::ExecutionFailed {
                service: self.service_name().to_string(),
                region: context.region.clone(),
                error: e,
            });
        }
    };

    // Parse JSON with detailed error
    let json = serde_json::from_str(&result.stdout)
        .map_err(|e| Error::ParseError {
            service: self.service_name().to_string(),
            raw_output: result.stdout.clone(),
            error: e,
        })?;

    // ...
}
```

### 2. Pagination Handling

```rust
async fn discover_all_pages(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
    let mut all_resources = Vec::new();
    let mut next_token: Option<String> = None;

    loop {
        let mut args = vec![
            "--region".to_string(),
            context.region.clone(),
        ];

        if let Some(token) = &next_token {
            args.push("--starting-token".to_string());
            args.push(token.clone());
        }

        let result = self.execute_aws_cli(args, context).await?;
        let json: serde_json::Value = serde_json::from_str(&result.stdout)?;

        // Extract resources from this page
        let page_resources = self.parse_resources(&json)?;
        all_resources.extend(page_resources);

        // Check for next page
        next_token = json["NextToken"].as_str().map(|s| s.to_string());
        if next_token.is_none() {
            break;
        }
    }

    Ok(all_resources)
}
```

### 3. Rate Limiting

```rust
pub struct RateLimitedScanner {
    scanner: Box<dyn ServiceScanner>,
    rate_limiter: Arc<RateLimiter>,
}

#[async_trait]
impl ServiceScanner for RateLimitedScanner {
    async fn discover(&self, context: &ScanContext) -> Result<Vec<RawResource>> {
        // Acquire rate limit permit
        let _permit = self.rate_limiter.acquire().await;

        // Execute discovery
        self.scanner.discover(context).await
    }

    // ... other methods
}
```

---

## See Also

- [Architecture](architecture.md) - Overall architecture
- [IAM Discovery](iam-discovery.md) - IAM permission discovery
- [API Reference](api.md) - Complete API documentation