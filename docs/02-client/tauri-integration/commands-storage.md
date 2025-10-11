# Storage Service Commands

**Module**: `cloudops-storage-service`
**Purpose**: Access local SQLite storage for estate data, chat history, policies, and application state

---

## Overview

The Storage Service provides persistent local storage for all application data. All data is stored in SQLite and accessed via Tauri commands from the frontend.

**Storage Location**: `~/.escher/escher.db` (SQLite database)

---

## Commands

### 1. Estate Data Commands

#### `search_resources`

Search for AWS resources in the local estate database.

**Rust Signature**:
```rust
#[tauri::command]
async fn search_resources(
    query: String,
    filters: Option<ResourceFilters>
) -> Result<Vec<AWSResource>, String>
```

**TypeScript Usage**:
```typescript
interface ResourceFilters {
  accountId?: string;
  region?: string;
  service?: string;
  resourceType?: string;
  tags?: Record<string, string>;
}

interface AWSResource {
  id: string;
  arn: string;
  resourceType: string;
  service: string;
  region: string;
  accountId: string;
  name?: string;
  tags: Record<string, string>;
  metadata: Record<string, any>;
  lastScanned: string;
}

const results = await invoke<AWSResource[]>('search_resources', {
  query: 'prod',
  filters: {
    service: 'ec2',
    region: 'us-east-1'
  }
});
```

**Errors**:
- `"Database query failed: {error}"` - Database error
- `"Invalid filter format: {error}"` - Filter validation failed

---

#### `get_estate_summary`

Get high-level summary of the AWS estate.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_estate_summary() -> Result<EstateSummary, String>
```

**TypeScript Usage**:
```typescript
interface EstateSummary {
  totalResources: number;
  accountCount: number;
  regionCount: number;
  serviceBreakdown: Record<string, number>;
  lastScanTime?: string;
  scanStatus: 'never' | 'in_progress' | 'completed' | 'failed';
}

const summary = await invoke<EstateSummary>('get_estate_summary');
```

**Errors**:
- `"Database not initialized"` - Storage not set up
- `"Failed to fetch summary: {error}"` - Query error

---

#### `get_resource_by_id`

Fetch a specific resource by ID.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_resource_by_id(resource_id: String) -> Result<AWSResource, String>
```

**TypeScript Usage**:
```typescript
const resource = await invoke<AWSResource>('get_resource_by_id', {
  resourceId: 'res_abc123'
});
```

**Errors**:
- `"Resource not found: {id}"` - Resource doesn't exist
- `"Database error: {error}"` - Query failed

---

#### `get_resources_by_account`

Get all resources for a specific AWS account.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_resources_by_account(
    account_id: String,
    pagination: Option<Pagination>
) -> Result<PaginatedResources, String>
```

**TypeScript Usage**:
```typescript
interface Pagination {
  page: number;
  pageSize: number;
}

interface PaginatedResources {
  resources: AWSResource[];
  totalCount: number;
  page: number;
  pageSize: number;
  hasMore: boolean;
}

const result = await invoke<PaginatedResources>('get_resources_by_account', {
  accountId: '123456789012',
  pagination: { page: 1, pageSize: 50 }
});
```

---

### 2. Chat History Commands

#### `save_chat_message`

Save a single chat message (user or assistant).

**Rust Signature**:
```rust
#[tauri::command]
async fn save_chat_message(
    conversation_id: String,
    message: ChatMessage
) -> Result<String, String>
```

**TypeScript Usage**:
```typescript
interface ChatMessage {
  id: string;
  conversationId: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: string;
  metadata?: {
    uiMode?: 'template' | 'dynamic';
    entities?: string[];
    action?: string;
  };
}

const messageId = await invoke<string>('save_chat_message', {
  conversationId: 'conv_123',
  message: {
    id: 'msg_456',
    conversationId: 'conv_123',
    role: 'user',
    content: 'List my EC2 instances',
    timestamp: new Date().toISOString()
  }
});
```

**Errors**:
- `"Invalid message format: {error}"` - Validation failed
- `"Failed to save message: {error}"` - Database error

---

#### `get_conversation_history`

Retrieve chat history for a conversation.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_conversation_history(
    conversation_id: String,
    limit: Option<u32>
) -> Result<Vec<ChatMessage>, String>
```

**TypeScript Usage**:
```typescript
const history = await invoke<ChatMessage[]>('get_conversation_history', {
  conversationId: 'conv_123',
  limit: 50  // Optional, defaults to all messages
});
```

**Errors**:
- `"Conversation not found: {id}"` - Invalid conversation ID
- `"Database error: {error}"` - Query failed

---

#### `list_conversations`

List all conversations for the current user.

**Rust Signature**:
```rust
#[tauri::command]
async fn list_conversations() -> Result<Vec<Conversation>, String>
```

**TypeScript Usage**:
```typescript
interface Conversation {
  id: string;
  title?: string;
  createdAt: string;
  updatedAt: string;
  messageCount: number;
  lastMessage?: string;
}

const conversations = await invoke<Conversation[]>('list_conversations');
```

---

#### `delete_conversation`

Delete a conversation and all its messages.

**Rust Signature**:
```rust
#[tauri::command]
async fn delete_conversation(conversation_id: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('delete_conversation', {
  conversationId: 'conv_123'
});
```

---

### 3. Policy & Permissions Commands

#### `save_policy`

Save or update an AI agent policy.

**Rust Signature**:
```rust
#[tauri::command]
async fn save_policy(policy: AgentPolicy) -> Result<String, String>
```

**TypeScript Usage**:
```typescript
interface AgentPolicy {
  id?: string;
  name: string;
  description?: string;
  rules: PolicyRule[];
  enabled: boolean;
  createdAt?: string;
  updatedAt?: string;
}

interface PolicyRule {
  action: string;  // e.g., "ec2:StopInstances"
  condition?: string;  // e.g., "tag:Environment == 'dev'"
  effect: 'allow' | 'deny';
  requireConfirmation: boolean;
}

const policyId = await invoke<string>('save_policy', {
  policy: {
    name: 'Dev Environment Policy',
    description: 'Allow stopping dev instances',
    rules: [
      {
        action: 'ec2:StopInstances',
        condition: 'tag:Environment == "dev"',
        effect: 'allow',
        requireConfirmation: false
      }
    ],
    enabled: true
  }
});
```

---

#### `get_all_policies`

Retrieve all saved policies.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_all_policies() -> Result<Vec<AgentPolicy>, String>
```

**TypeScript Usage**:
```typescript
const policies = await invoke<AgentPolicy[]>('get_all_policies');
```

---

#### `delete_policy`

Delete a policy by ID.

**Rust Signature**:
```rust
#[tauri::command]
async fn delete_policy(policy_id: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('delete_policy', {
  policyId: 'policy_123'
});
```

---

### 4. Recommendations Commands

#### `get_recommendations`

Fetch stored recommendations.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_recommendations(
    filters: Option<RecommendationFilters>
) -> Result<Vec<Recommendation>, String>
```

**TypeScript Usage**:
```typescript
interface RecommendationFilters {
  category?: 'cost' | 'security' | 'performance' | 'reliability';
  status?: 'pending' | 'applied' | 'dismissed';
  priority?: 'high' | 'medium' | 'low';
}

interface Recommendation {
  id: string;
  title: string;
  description: string;
  category: string;
  priority: string;
  potentialSavings?: number;
  resourceIds: string[];
  status: string;
  createdAt: string;
}

const recommendations = await invoke<Recommendation[]>('get_recommendations', {
  filters: { category: 'cost', status: 'pending' }
});
```

---

#### `update_recommendation_status`

Update the status of a recommendation.

**Rust Signature**:
```rust
#[tauri::command]
async fn update_recommendation_status(
    recommendation_id: String,
    status: String,
    notes: Option<String>
) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('update_recommendation_status', {
  recommendationId: 'rec_123',
  status: 'applied',
  notes: 'Applied via playbook execution'
});
```

---

### 5. Alerts Commands

#### `get_alerts`

Fetch stored alerts.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_alerts(
    filters: Option<AlertFilters>
) -> Result<Vec<Alert>, String>
```

**TypeScript Usage**:
```typescript
interface AlertFilters {
  severity?: 'critical' | 'high' | 'medium' | 'low';
  status?: 'active' | 'acknowledged' | 'resolved';
  category?: string;
}

interface Alert {
  id: string;
  title: string;
  description: string;
  severity: string;
  category: string;
  status: string;
  resourceIds: string[];
  triggeredAt: string;
  acknowledgedAt?: string;
  resolvedAt?: string;
}

const alerts = await invoke<Alert[]>('get_alerts', {
  filters: { severity: 'critical', status: 'active' }
});
```

---

#### `acknowledge_alert`

Mark an alert as acknowledged.

**Rust Signature**:
```rust
#[tauri::command]
async fn acknowledge_alert(alert_id: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('acknowledge_alert', {
  alertId: 'alert_123'
});
```

---

### 6. Application State Commands

#### `save_app_state`

Save application UI state (window position, preferences, etc.).

**Rust Signature**:
```rust
#[tauri::command]
async fn save_app_state(
    key: String,
    value: serde_json::Value
) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('save_app_state', {
  key: 'window_position',
  value: { x: 100, y: 200, width: 1200, height: 800 }
});
```

---

#### `get_app_state`

Retrieve application state by key.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_app_state(key: String) -> Result<serde_json::Value, String>
```

**TypeScript Usage**:
```typescript
const windowPos = await invoke<any>('get_app_state', {
  key: 'window_position'
});
```

---

### 7. Backup & Maintenance Commands

#### `backup_database`

Create a backup of the local database.

**Rust Signature**:
```rust
#[tauri::command]
async fn backup_database(backup_path: Option<String>) -> Result<String, String>
```

**TypeScript Usage**:
```typescript
// Auto-generated backup path
const backupPath = await invoke<string>('backup_database', {
  backupPath: null
});

// Custom backup path
const backupPath = await invoke<string>('backup_database', {
  backupPath: '/path/to/backup.db'
});
```

**Returns**: Path to the backup file

---

#### `get_storage_stats`

Get database storage statistics.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_storage_stats() -> Result<StorageStats, String>
```

**TypeScript Usage**:
```typescript
interface StorageStats {
  databaseSizeBytes: number;
  totalResources: number;
  totalMessages: number;
  totalPolicies: number;
  totalRecommendations: number;
  totalAlerts: number;
  lastBackup?: string;
}

const stats = await invoke<StorageStats>('get_storage_stats');
```

---

## TypeScript Service Wrapper

**Recommended Pattern**: Wrap all Tauri commands in a TypeScript service class.

**File**: `src/services/tauri/StorageService.ts`

```typescript
import { invoke } from '@tauri-apps/api/tauri';

export class StorageService {
  // Estate Data
  async searchResources(query: string, filters?: ResourceFilters): Promise<AWSResource[]> {
    return invoke('search_resources', { query, filters });
  }

  async getEstateSummary(): Promise<EstateSummary> {
    return invoke('get_estate_summary');
  }

  async getResourceById(resourceId: string): Promise<AWSResource> {
    return invoke('get_resource_by_id', { resourceId });
  }

  // Chat History
  async saveChatMessage(conversationId: string, message: ChatMessage): Promise<string> {
    return invoke('save_chat_message', { conversationId, message });
  }

  async getConversationHistory(conversationId: string, limit?: number): Promise<ChatMessage[]> {
    return invoke('get_conversation_history', { conversationId, limit });
  }

  async listConversations(): Promise<Conversation[]> {
    return invoke('list_conversations');
  }

  async deleteConversation(conversationId: string): Promise<boolean> {
    return invoke('delete_conversation', { conversationId });
  }

  // Policies
  async savePolicy(policy: AgentPolicy): Promise<string> {
    return invoke('save_policy', { policy });
  }

  async getAllPolicies(): Promise<AgentPolicy[]> {
    return invoke('get_all_policies');
  }

  async deletePolicy(policyId: string): Promise<boolean> {
    return invoke('delete_policy', { policyId });
  }

  // Recommendations
  async getRecommendations(filters?: RecommendationFilters): Promise<Recommendation[]> {
    return invoke('get_recommendations', { filters });
  }

  async updateRecommendationStatus(
    recommendationId: string,
    status: string,
    notes?: string
  ): Promise<boolean> {
    return invoke('update_recommendation_status', { recommendationId, status, notes });
  }

  // Alerts
  async getAlerts(filters?: AlertFilters): Promise<Alert[]> {
    return invoke('get_alerts', { filters });
  }

  async acknowledgeAlert(alertId: string): Promise<boolean> {
    return invoke('acknowledge_alert', { alertId });
  }

  // App State
  async saveAppState(key: string, value: any): Promise<boolean> {
    return invoke('save_app_state', { key, value });
  }

  async getAppState(key: string): Promise<any> {
    return invoke('get_app_state', { key });
  }

  // Maintenance
  async backupDatabase(backupPath?: string): Promise<string> {
    return invoke('backup_database', { backupPath });
  }

  async getStorageStats(): Promise<StorageStats> {
    return invoke('get_storage_stats');
  }
}

// Singleton instance
export const storageService = new StorageService();
```

---

## Usage in Controllers

**Example**: ChatController using StorageService

```typescript
import { storageService } from '@/services/tauri/StorageService';

export class ChatController {
  async loadConversationHistory(conversationId: string): Promise<ChatMessage[]> {
    try {
      const history = await storageService.getConversationHistory(conversationId, 50);
      return history;
    } catch (error) {
      console.error('Failed to load conversation history:', error);
      throw new Error('Could not load chat history');
    }
  }

  async saveMessage(conversationId: string, message: ChatMessage): Promise<void> {
    try {
      await storageService.saveChatMessage(conversationId, message);
    } catch (error) {
      console.error('Failed to save message:', error);
      throw new Error('Could not save message');
    }
  }
}
```

---

## Error Handling Best Practices

1. **Always wrap invoke calls in try-catch**:
```typescript
try {
  const result = await invoke('command_name', { params });
} catch (error) {
  // error is a string from Rust
  console.error('Command failed:', error);
  showUserFriendlyError(error);
}
```

2. **Map Rust errors to user-friendly messages**:
```typescript
function mapStorageError(error: string): string {
  if (error.includes('not found')) {
    return 'Resource not found. It may have been deleted.';
  }
  if (error.includes('Database')) {
    return 'Storage error. Please try again.';
  }
  return 'An unexpected error occurred.';
}
```

3. **Use TypeScript types for all interfaces** to catch errors at compile time.

---

## Performance Notes

- **Pagination**: Always use pagination for large result sets (resources, messages)
- **Caching**: Consider caching estate summary and frequently accessed data in Zustand
- **Batching**: Group multiple saves into transactions when possible (handled by Rust layer)
- **Indexing**: SQLite indexes are pre-configured for common queries (account_id, resource_type, conversation_id)

---

## Next Steps

- [Execution Engine Commands →](./commands-execution.md)
- [Estate Scanner Commands →](./commands-estate-scanner.md)
- [Authentication Commands →](./commands-auth.md)