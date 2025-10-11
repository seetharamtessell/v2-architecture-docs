# Request Builder Commands

**Module**: `cloudops-request-builder`
**Status**: ⚠️ To be designed
**Purpose**: Context enrichment and server request preparation

---

## Overview

The Request Builder module will handle:
- Enriching user queries with context from estate data
- Preparing requests for server communication
- Formatting chat messages with relevant metadata
- Adding policy constraints to requests

**Note**: This module is pending design. The commands below are preliminary and subject to change.

---

## Preliminary Commands

### 1. Context Enrichment

#### `enrich_user_query` (Planned)

Enrich a user query with relevant context from local estate data.

**Rust Signature** (Preliminary):
```rust
#[tauri::command]
async fn enrich_user_query(
    query: String,
    conversation_history: Vec<ChatMessage>
) -> Result<EnrichedQuery, String>
```

**TypeScript Usage** (Preliminary):
```typescript
interface EnrichedQuery {
  originalQuery: string;
  enrichedQuery: string;
  context: {
    relevantResources?: AWSResource[];
    recentExecutions?: ExecutionRecord[];
    activePolicies?: AgentPolicy[];
    estateSummary?: EstateSummary;
  };
  metadata: {
    entities: string[];
    intent: string;
    confidence: number;
  };
}

// Enrich query before sending to server
const enriched = await invoke<EnrichedQuery>('enrich_user_query', {
  query: 'Stop my production database',
  conversationHistory: recentMessages
});

// Send enriched query to server
await sendToServer(enriched);
```

**Purpose**: Add context so server can make better decisions

---

### 2. Request Preparation

#### `prepare_server_request` (Planned)

Format a request for the server with all necessary context.

**Rust Signature** (Preliminary):
```rust
#[tauri::command]
async fn prepare_server_request(
    message: String,
    request_type: RequestType,
    options: Option<RequestOptions>
) -> Result<ServerRequest, String>
```

**TypeScript Usage** (Preliminary):
```typescript
type RequestType = 'chat' | 'playbook_execution' | 'recommendation_request';

interface RequestOptions {
  includeEstate?: boolean;
  includePolicies?: boolean;
  includeHistory?: boolean;
  historyLimit?: number;
}

interface ServerRequest {
  id: string;
  type: RequestType;
  message: string;
  context: {
    history?: ChatMessage[];
    estateSummary?: EstateSummary;
    policies?: AgentPolicy[];
    userInfo?: TokenPayload;
  };
  metadata: {
    timestamp: string;
    clientVersion: string;
  };
}

const request = await invoke<ServerRequest>('prepare_server_request', {
  message: 'List my EC2 instances',
  requestType: 'chat',
  options: {
    includeEstate: true,
    includePolicies: true,
    includeHistory: true,
    historyLimit: 20
  }
});

// Send to server via WebSocket
websocket.send(JSON.stringify(request));
```

---

### 3. Policy Validation

#### `validate_request_against_policies` (Planned)

Check if a request complies with configured policies.

**Rust Signature** (Preliminary):
```rust
#[tauri::command]
async fn validate_request_against_policies(
    request: String,
    action: String
) -> Result<PolicyValidation, String>
```

**TypeScript Usage** (Preliminary):
```typescript
interface PolicyValidation {
  allowed: boolean;
  requiresConfirmation: boolean;
  matchedPolicies: string[];  // Policy IDs
  violations?: PolicyViolation[];
  message?: string;
}

interface PolicyViolation {
  policyId: string;
  policyName: string;
  reason: string;
}

const validation = await invoke<PolicyValidation>('validate_request_against_policies', {
  request: 'Stop RDS instance db-prod-01',
  action: 'rds:StopDBInstance'
});

if (!validation.allowed) {
  showError(`Action blocked: ${validation.message}`);
  return;
}

if (validation.requiresConfirmation) {
  const confirmed = await showConfirmation('Are you sure?');
  if (!confirmed) return;
}

// Proceed with request
```

---

### 4. Resource Resolution

#### `resolve_resource_references` (Planned)

Resolve natural language resource references to actual resource IDs.

**Rust Signature** (Preliminary):
```rust
#[tauri::command]
async fn resolve_resource_references(
    query: String
) -> Result<ResourceResolution, String>
```

**TypeScript Usage** (Preliminary):
```typescript
interface ResourceResolution {
  originalQuery: string;
  resolvedResources: ResolvedResource[];
  ambiguous: AmbiguousReference[];
}

interface ResolvedResource {
  reference: string;           // e.g., "production database"
  resourceId: string;          // e.g., "db-prod-01"
  resourceArn: string;
  resourceType: string;
  confidence: number;
}

interface AmbiguousReference {
  reference: string;
  candidates: AWSResource[];   // Multiple matches
  reason: string;
}

const resolution = await invoke<ResourceResolution>('resolve_resource_references', {
  query: 'Stop my production database'
});

if (resolution.ambiguous.length > 0) {
  // Ask user to clarify
  const selected = await showResourceSelector(resolution.ambiguous[0].candidates);
  // Retry with specific resource
}

// Use resolved resources in request
console.log(`Resolved to: ${resolution.resolvedResources[0].resourceId}`);
```

---

## Design Notes

The Request Builder module needs to address:

1. **Context Efficiency**: How much estate data to include in each request?
   - Option A: Include full estate summary
   - Option B: Include only relevant resources (filtered by query)
   - Option C: Server fetches from database directly

2. **Policy Enforcement Point**: Where should policies be enforced?
   - Client-side (before sending) - fast feedback
   - Server-side (authoritative) - more secure
   - Both (client validates, server enforces) - best UX + security

3. **Resource Resolution**: Should this happen client-side or server-side?
   - Client-side: Faster, uses local estate data
   - Server-side: More accurate, can access full context
   - Hybrid: Client suggests, server validates

4. **History Management**: How much conversation history to send?
   - Last N messages (e.g., 20)
   - Last N tokens (e.g., 10,000)
   - Relevant messages only (filtered by similarity)
   - Server-managed (client sends conversation_id, server fetches)

---

## Recommended Design Direction

Based on the architecture documents, here's the recommended approach:

### Option 1: Lightweight Request Builder (Recommended)

**Philosophy**: Keep client thin, server handles heavy lifting

**Client Responsibilities**:
- Format requests with message + conversation_id
- Include minimal metadata (timestamp, version)
- Basic policy pre-checks for UX (non-authoritative)

**Server Responsibilities**:
- Fetch conversation history from database
- Load relevant estate data based on query
- Enrich context with policies
- Resolve resource references

**Advantages**:
- Simpler client code
- Server has full context
- Easier to update enrichment logic
- Consistent behavior across clients

**Commands Needed**:
```typescript
// Minimal client-side preparation
prepare_chat_message(message, conversationId)
validate_basic_policies(message, action)  // Quick UX check only
```

---

### Option 2: Rich Request Builder (Alternative)

**Philosophy**: Client enriches fully, server just processes

**Client Responsibilities**:
- Enrich with full context (estate, history, policies)
- Resolve resource references locally
- Validate policies authoritatively

**Server Responsibilities**:
- Process enriched request
- Execute actions
- Return enhanced UI

**Advantages**:
- Server stays stateless
- Client has full control
- Lower server load

**Disadvantages**:
- Larger messages (estate data + history)
- Harder to keep logic consistent
- More complex client code

---

## Implementation Priority

**Phase 1** (MVP): Lightweight approach
- `prepare_chat_message` - Basic message formatting
- `get_conversation_context` - Fetch last N messages for sending

**Phase 2** (Enhancement): Add client-side optimization
- `resolve_resource_references` - Improve UX with suggestions
- `validate_basic_policies` - Fast feedback before server call

**Phase 3** (Advanced): Smart context
- `enrich_user_query` - Add relevant estate context
- `calculate_context_relevance` - Filter history by relevance

---

## TypeScript Service Wrapper (Preliminary)

**File**: `src/services/tauri/RequestBuilderService.ts`

```typescript
import { invoke } from '@tauri-apps/api/tauri';

export class RequestBuilderService {
  // Phase 1: MVP
  async prepareChatMessage(
    message: string,
    conversationId: string
  ): Promise<ServerRequest> {
    // To be implemented
    return invoke('prepare_chat_message', { message, conversationId });
  }

  async getConversationContext(
    conversationId: string,
    limit: number
  ): Promise<ChatMessage[]> {
    // Wrapper around storage service
    return invoke('get_conversation_history', { conversationId, limit });
  }

  // Phase 2: Enhancement
  async resolveResourceReferences(query: string): Promise<ResourceResolution> {
    return invoke('resolve_resource_references', { query });
  }

  async validateBasicPolicies(message: string, action: string): Promise<PolicyValidation> {
    return invoke('validate_basic_policies', { message, action });
  }

  // Phase 3: Advanced
  async enrichUserQuery(
    query: string,
    conversationHistory: ChatMessage[]
  ): Promise<EnrichedQuery> {
    return invoke('enrich_user_query', { query, conversationHistory });
  }
}

export const requestBuilderService = new RequestBuilderService();
```

---

## Next Steps

1. **Design Decision**: Choose between Lightweight vs Rich approach
2. **Server Integration**: Define server API contract
3. **Implementation**: Build Phase 1 commands
4. **Testing**: Validate with real queries and estate data
5. **Optimization**: Add Phase 2/3 based on performance needs

---

## Related Documentation

- [Frontend Data Flow →](../frontend/data-flow.md) (To be created)
- [Server Integration Protocol →](../frontend/integration-patterns.md) (To be created)
- [WebSocket Communication →](./typescript-services.md) (To be created)

---

## Questions to Resolve

1. Should Request Builder call Storage Service directly or go through controllers?
2. How to handle large estate contexts (>100k resources)?
3. Should policies be evaluated client-side at all?
4. What's the server's preferred request format?
5. How does this integrate with UI Agent (server-side)?

---

**Status**: This document is a preliminary design. Implementation pending architecture decisions.