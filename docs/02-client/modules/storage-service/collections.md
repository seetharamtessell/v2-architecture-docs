# Collection Schemas

Detailed schemas for Qdrant collections used by the Storage Service.

## Overview

**Target Architecture**: The Storage Service will support **5 RAG collections** as defined in the PRODUCT-VISION:

1. **Cloud Estate Inventory** - Multi-cloud resource metadata (AWS, Azure, GCP)
2. **Chat History** - Conversational history with AI
3. **Executed Operations** - History of operations executed
4. **Immutable Reports** - Cost reports, audit logs, compliance reports
5. **Alerts & Events** - Alert rules, alert history, scan results, morning reports

**Current Implementation Status** (Phase 1):
- ✅ **chat_history** - Stores conversation messages (no vector search needed)
- ✅ **cloud_estate** - Stores multi-cloud resources (with semantic search)
- 🚧 **Executed Operations, Immutable Reports, Alerts & Events** - To be implemented

This document describes the **currently implemented collections** (Phase 1). Additional collections will be added as the storage service evolves.

## Collection Comparison

| Aspect | chat_history | cloud_estate |
|--------|-------------|------------|
| **Purpose** | Store conversation messages | Store multi-cloud resource metadata (AWS/Azure/GCP) |
| **Vector Search** | No (dummy vectors) | Yes (semantic search) |
| **Vector Size** | 1 dimension | 384/768/1536 (model-dependent) |
| **Access Pattern** | Filter by context_id | Vector search + filters |
| **Encryption** | Encrypted content | Encrypted metadata |
| **Point ID** | Random UUID | Deterministic (account-region-id) |
| **Update Pattern** | Immutable (append-only) | Mutable (upsert) |

## Chat History Collection

### Design Rationale

The chat history collection uses **dummy vectors** because:
- No semantic search needed (conversations are sequential)
- Access by context_id only (exact match)
- Reduces memory overhead (1 byte vs 1536+ bytes per message)
- Faster writes (no embedding generation)

### Schema

```rust
CollectionConfig {
    name: "chat_history".to_string(),
    vector_size: 1, // Dummy vector
    distance: DistanceMetric::Cosine, // Not used, but required
    indexed_fields: vec![
        IndexedField {
            name: "context_id".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "message_index".to_string(),
            field_type: IndexFieldType::Integer,
            indexed: true,
        },
        IndexedField {
            name: "timestamp".to_string(),
            field_type: IndexFieldType::Integer,
            indexed: true,
        },
        IndexedField {
            name: "role".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
    ],
    hnsw_config: None, // Not needed for dummy vectors
}
```

### Point Structure

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "vector": [0.0],
  "payload": {
    "context_id": "conv-123e4567-e89b-12d3-a456-426614174000",
    "message_index": 5,
    "role": "user",
    "timestamp": 1728391162,
    "encrypted_content": "xK9mP2vL8nQ5...base64..."
  }
}
```

### Payload Fields

#### Plain Text (Indexed)

| Field | Type | Purpose | Indexed |
|-------|------|---------|---------|
| `context_id` | Keyword | Conversation identifier | Yes |
| `message_index` | Integer | Sequential position in conversation | Yes |
| `role` | Keyword | "user", "assistant", or "system" | Yes |
| `timestamp` | Integer | Unix timestamp | Yes |

#### Encrypted

| Field | Type | Content |
|-------|------|---------|
| `encrypted_content` | String | Base64-encoded encrypted JSON |

**Encrypted Content Structure:**
```json
{
  "content": "Stop pg-instance-main1",
  "metadata": {
    "resources_mentioned": ["rds-pg-instance-main1"],
    "commands_executed": ["aws rds stop-db-instance ..."]
  }
}
```

### Point ID Strategy

**Random UUID** - Messages are immutable and never updated.

```rust
// Generate unique ID for each message
let point_id = Uuid::new_v4().to_string();
// "550e8400-e29b-41d4-a716-446655440000"
```

**Benefits:**
- Guaranteed uniqueness
- No collision risk
- Simple implementation

### Access Patterns

#### 1. Get Conversation History

```rust
// Use scroll API (not search)
let results = qdrant.scroll(Scroll {
    collection_name: "chat_history".to_string(),
    filter: Some(Filter {
        must: vec![
            Condition::matches("context_id", context_id),
        ],
    }),
    order_by: Some(OrderBy {
        key: "message_index".to_string(),
        direction: Some(Direction::Asc),
    }),
    limit: Some(100),
    with_payload: Some(WithPayloadSelector::enable(true)),
}).await?;
```

#### 2. Get Recent Messages

```rust
// Filter by context_id and limit
let results = qdrant.scroll(Scroll {
    collection_name: "chat_history".to_string(),
    filter: Some(Filter {
        must: vec![
            Condition::matches("context_id", context_id),
        ],
    }),
    order_by: Some(OrderBy {
        key: "message_index".to_string(),
        direction: Some(Direction::Desc),
    }),
    limit: Some(10), // Last 10 messages
    with_payload: Some(WithPayloadSelector::enable(true)),
}).await?;
```

#### 3. Delete Conversation

```rust
// Delete all messages with context_id
qdrant.delete_points(
    "chat_history",
    Some(Filter {
        must: vec![
            Condition::matches("context_id", context_id),
        ],
    }),
    None,
).await?;
```

## AWS Estate Collection

### Design Rationale

The AWS estate collection uses **real vectors** because:
- Semantic search required ("postgres database" → RDS instances)
- Fuzzy matching needed (typos, abbreviations)
- Natural language queries
- Combined vector + filter search

### Schema

```rust
CollectionConfig {
    name: "aws_estate".to_string(),
    vector_size: 384, // Match embedding model dimension
    distance: DistanceMetric::Cosine,
    indexed_fields: vec![
        IndexedField {
            name: "resource_type".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "account_id".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "region".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "state".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "last_synced".to_string(),
            field_type: IndexFieldType::Integer,
            indexed: true,
        },
        IndexedField {
            name: "tags.env".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
        IndexedField {
            name: "tags.app".to_string(),
            field_type: IndexFieldType::Keyword,
            indexed: true,
        },
    ],
    hnsw_config: Some(HnswConfig {
        m: 16,
        ef_construct: 100,
        full_scan_threshold: 10000,
    }),
}
```

### AWS Estate Hierarchy

Each AWS resource contains IAM permissions for that specific resource:

```
AWS Resource (e.g., RDS Instance "pg-main1")
├── Resource Metadata (ARN, name, state, etc.)
└── IAM Permissions for THIS resource
    ├── Allowed Actions (rds:StopDBInstance, rds:StartDBInstance, ...)
    ├── Denied Actions
    └── Role/User context
```

**Benefit**: When user asks "stop pg-main1", we can immediately show:
- Resource details (state, region, etc.)
- What permissions you have (can stop? can start? read-only?)

### Point Structure

```json
{
  "id": "123456789012-us-west-2-rds-pg-instance-main1",
  "vector": [0.123, 0.456, 0.789, ...],
  "payload": {
    // ✅ PLAIN TEXT (for filtering)
    "resource_type": "rds_instance",
    "account_id": "123456789012",
    "account_name": "production-account",
    "region": "us-west-2",
    "service": "rds",
    "state": "available",
    "last_synced": 1728391162,
    "tags": {
      "env": "production",
      "app": "main"
    },

    // ✅ ENCRYPTED (sensitive metadata + IAM permissions)
    "encrypted_data": "xK9mP2vL8nQ5...base64..."

    // After decryption, encrypted_data contains:
    // {
    //   "identifier": "pg-instance-main1",
    //   "arn": "arn:aws:rds:us-west-2:123456789012:db:pg-instance-main1",
    //   "name": "pg-instance-main1",
    //   "engine": "postgres",
    //   "engine_version": "14.7",
    //
    //   // IAM permissions for THIS resource
    //   "iam": {
    //     "allowed_actions": [
    //       "rds:StopDBInstance",
    //       "rds:StartDBInstance",
    //       "rds:DescribeDBInstances",
    //       "rds:CreateDBSnapshot"
    //     ],
    //     "denied_actions": [
    //       "rds:DeleteDBInstance"
    //     ],
    //     "user_context": {
    //       "username": "john.doe",
    //       "role_arn": "arn:aws:iam::123456789012:role/DeveloperRole",
    //       "session_expires": 1728477562
    //     }
    //   },
    //
    //   "metadata": {
    //     "instance_class": "db.t3.medium",
    //     "allocated_storage": 100,
    //     "vpc_id": "vpc-123456",
    //     "subnet_group": "default",
    //     "publicly_accessible": false,
    //     "multi_az": true
    //   }
    // }
  }
}
```

### Payload Fields

#### Plain Text (Indexed)

| Field | Type | Purpose | Indexed |
|-------|------|---------|---------|
| `resource_type` | Keyword | "ec2_instance", "rds_instance", "s3_bucket", etc. | Yes |
| `account_id` | Keyword | AWS account ID | Yes |
| `account_name` | Keyword | Human-readable account name | Yes |
| `region` | Keyword | AWS region ("us-west-2", etc.) | Yes |
| `service` | Keyword | "ec2", "rds", "s3", "lambda", etc. | Yes |
| `state` | Keyword | Resource state ("available", "running", etc.) | Yes |
| `last_synced` | Integer | Last sync timestamp | Yes |
| `tags.*` | Keyword | Resource tags (nested, selective indexing) | Selective |

#### Encrypted

| Field | Type | Content |
|-------|------|---------|
| `encrypted_data` | String | Base64-encoded encrypted JSON |

**Encrypted Data Structure:**
```json
{
  "identifier": "pg-instance-main1",
  "arn": "arn:aws:rds:us-west-2:123456789012:db:pg-instance-main1",
  "name": "pg-instance-main1",
  "engine": "postgres",
  "engine_version": "14.7",
  "permissions": ["rds:StopDBInstance", "rds:StartDBInstance"],
  "constraints": {
    "can_stop": true,
    "has_read_replicas": false,
    "multi_az": true
  },
  "metadata": {
    "instance_class": "db.t3.medium",
    "allocated_storage": 100,
    "vpc_id": "vpc-123456",
    "subnet_group": "default"
  }
}
```

### Point ID Strategy

**Deterministic ID** - Resources are updated frequently.

```rust
// Generate deterministic ID
fn generate_resource_id(resource: &AWSResource) -> String {
    format!("{}-{}-{}",
        resource.account_id,
        resource.region,
        resource.identifier
    )
}

// Example: "123456789012-us-west-2-pg-instance-main1"
```

**Benefits:**
- Same resource = same point ID
- Upsert updates existing point (no duplicates)
- Idempotent sync operations

**Why This Matters:**

```rust
// First sync: Creates point
upsert_resource(resource); // point_id: "123456789012-us-west-2-pg-instance-main1"

// Later sync: Updates same point (state changed)
upsert_resource(resource); // Same point_id, updates in place

// ❌ Without deterministic ID:
// - Random UUID would create new point
// - Result: Duplicate entries for same resource
```

### Vector Embedding

#### What Gets Embedded

The embedding combines multiple fields for rich semantic search:

```rust
fn generate_embedding_text(resource: &AWSResource) -> String {
    format!(
        "{} {} {} {} {}",
        resource.resource_type,
        resource.name,
        resource.identifier,
        resource.tags.get("name").unwrap_or(&"".to_string()),
        resource.tags.iter()
            .map(|(k, v)| format!("{}: {}", k, v))
            .collect::<Vec<_>>()
            .join(" ")
    )
}

// Example result:
// "rds_instance pg-instance-main1 pg-instance-main1 env: production app: main"
```

#### Why Embed Tags

Tags enable semantic queries like:
- "production database" → Matches resources with `tags.env = "production"` and `resource_type = "rds_instance"`
- "main app instances" → Matches resources with `tags.app = "main"` and `resource_type = "ec2_instance"`

### Access Patterns

#### 1. Semantic Search with Filters

```rust
// User query: "postgres database in production"
let vector = embedder.embed("postgres database in production").await?;

let results = qdrant.search_points(SearchPoints {
    collection_name: "aws_estate".to_string(),
    vector,
    filter: Some(Filter {
        must: vec![
            Condition::matches("resource_type", "rds_instance"),
            Condition::matches("tags.env", "production"),
            Condition::matches("state", "available"),
        ],
    }),
    limit: 10,
    with_payload: Some(WithPayloadSelector::enable(true)),
}).await?;
```

#### 2. Get All Resources by Type

```rust
// Get all EC2 instances
let results = qdrant.scroll(Scroll {
    collection_name: "aws_estate".to_string(),
    filter: Some(Filter {
        must: vec![
            Condition::matches("resource_type", "ec2_instance"),
        ],
    }),
    limit: Some(1000),
    with_payload: Some(WithPayloadSelector::enable(true)),
}).await?;
```

#### 3. Get Resources by Account and Region

```rust
// Get all resources in specific account and region
let results = qdrant.scroll(Scroll {
    collection_name: "aws_estate".to_string(),
    filter: Some(Filter {
        must: vec![
            Condition::matches("account_id", "123456789012"),
            Condition::matches("region", "us-west-2"),
        ],
    }),
    limit: Some(1000),
    with_payload: Some(WithPayloadSelector::enable(true)),
}).await?;
```

#### 4. Sync Update Pattern

```rust
// Get all existing point IDs
let existing_ids = get_all_point_ids("aws_estate").await?;

// Upsert current resources (create or update)
for resource in current_resources {
    upsert_resource(resource).await?; // Uses deterministic ID
}

// Delete stale resources (deleted from AWS)
let current_ids: HashSet<_> = current_resources
    .iter()
    .map(|r| generate_resource_id(r))
    .collect();

let stale_ids: Vec<_> = existing_ids
    .into_iter()
    .filter(|id| !current_ids.contains(id))
    .collect();

if !stale_ids.is_empty() {
    qdrant.delete_points("aws_estate", &stale_ids).await?;
}
```

## Indexing Strategy

### When to Index

Index fields that are:
- Used in filters frequently
- Have high cardinality (many unique values)
- Need fast exact-match lookups

### When NOT to Index

Don't index fields that are:
- Rarely used in filters
- Have low cardinality (few unique values)
- Only used in vector search

### Nested Field Indexing

Tags are stored as nested objects, but specific tags can be indexed:

```rust
IndexedField {
    name: "tags.env".to_string(),  // Index environment tag
    field_type: IndexFieldType::Keyword,
    indexed: true,
}
```

**Example Query:**
```rust
Condition::matches("tags.env", "production")
```

**All Tags Access:**
```rust
// Even non-indexed tags are accessible
let env = point.payload["tags"]["env"].as_str();
```

## Collection Maintenance

### Point Count Monitoring

```rust
let chat_count = qdrant.count_points("chat_history").await?;
let estate_count = qdrant.count_points("aws_estate").await?;

// Alert if unexpected growth
if estate_count > expected_resource_count * 2 {
    warn!("Possible duplicate resources in aws_estate: {}", estate_count);
}
```

### Cleanup Strategies

#### Chat History

```rust
// Delete old conversations (older than 90 days)
let cutoff = Utc::now().timestamp() - (90 * 24 * 60 * 60);

qdrant.delete_points(
    "chat_history",
    Some(Filter {
        must: vec![
            Condition::range("timestamp", Range {
                lt: Some(cutoff as f64),
                ..Default::default()
            }),
        ],
    }),
    None,
).await?;
```

#### AWS Estate

```rust
// Delete resources not synced in 7 days (likely deleted from AWS)
let cutoff = Utc::now().timestamp() - (7 * 24 * 60 * 60);

qdrant.delete_points(
    "aws_estate",
    Some(Filter {
        must: vec![
            Condition::range("last_synced", Range {
                lt: Some(cutoff as f64),
                ..Default::default()
            }),
        ],
    }),
    None,
).await?;
```

## Performance Considerations

### Chat History

- **No HNSW index** - Reduces memory overhead
- **Scroll API** - More efficient than search for filtering
- **Indexed context_id** - Fast conversation lookup
- **Indexed message_index** - Fast ordering

**Expected Performance:**
- Append: <1ms
- Get conversation (100 messages): <10ms
- Delete conversation: <100ms

### AWS Estate

- **HNSW index** - Fast vector search
- **Indexed filters** - Combined vector + filter queries
- **Upsert strategy** - Prevents duplicates

**Expected Performance:**
- Upsert: 1-5ms (with embedding)
- Search (10k resources): 10-50ms
- Bulk upsert (1000 resources): 1-3s

### Optimization Tips

1. **Batch Operations**: Use bulk_upsert instead of individual upserts
2. **Limit Indexed Fields**: Only index frequently-filtered fields
3. **Appropriate HNSW Settings**: Balance recall vs performance
4. **Regular Cleanup**: Remove stale data to maintain performance

## See Also

- [API Reference](api.md) - Storage Service API
- [Configuration](configuration.md) - Collection configuration
- [Point Management](point-management.md) - Point ID strategies
- [Operations](operations.md) - Common operations