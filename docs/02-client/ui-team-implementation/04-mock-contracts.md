# UI Team - Mock Contracts

TypeScript interfaces and mock implementations for Platform and Server APIs.

**Purpose**: Define contracts so UI can be built independently with mocks, then swap to real implementations later.

---

## Table of Contents

1. [Tauri Platform Contracts](#tauri-platform-contracts)
2. [WebSocket Server Contracts](#websocket-server-contracts)
3. [HTTP API Contracts](#http-api-contracts)
4. [Mock Data Generators](#mock-data-generators)

---

## Tauri Platform Contracts

### Storage Commands

```typescript
// src/services/tauri/types.ts

// Chat history
export interface GetChatHistoryParams {
  contextId: string;
  limit?: number;
}

export interface SaveMessageParams {
  contextId: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  metadata?: Record<string, any>;
}

// Estate operations
export interface GetEstateResourcesParams {
  filters?: {
    resourceType?: string;
    region?: string;
    accountId?: string;
    state?: string;
    tags?: Record<string, string>;
  };
  limit?: number;
  offset?: number;
}

export interface SemanticSearchParams {
  query: string;
  filters?: GetEstateResourcesParams['filters'];
  limit?: number;
}

// Policies
export interface AgentPolicy {
  allowedOperations: Record<string, string[]>;
  blockedOperations: Record<string, string[]>;
  resourceConstraints: {
    tags?: Record<string, string>;
    instanceTypes?: string[];
    costLimits?: number;
  };
  approvalRequired: Record<string, boolean>;
}
```

### Scanner Commands

```typescript
export interface ScanConfig {
  accounts: Array<{
    id: string;
    name: string;
    role?: string;
  }>;
  regions: string[];
  services: string[];
  filters?: {
    includeTags?: Record<string, string>;
    excludeTags?: Record<string, string>;
  };
}

export interface ScanProgress {
  scanId: string;
  status: 'running' | 'paused' | 'completed' | 'failed' | 'cancelled';
  progress: {
    percentComplete: number;
    currentService: string;
    servicesCompleted: number;
    servicesTotal: number;
    resourcesDiscovered: number;
  };
  currentTask?: string;
  elapsedSeconds: number;
}

export interface ScanResults {
  scanId: string;
  totalResources: number;
  resourcesByService: Record<string, number>;
  resourcesByRegion: Record<string, number>;
  errors?: Array<{
    service: string;
    error: string;
  }>;
}
```

### Execution Commands

```typescript
export interface ExecuteCommandParams {
  command: string;
  args?: string[];
  workingDir?: string;
  env?: Record<string, string>;
}

export interface ExecutePlaybookParams {
  playbook: Playbook;
  context?: {
    resources?: string[];
    parameters?: Record<string, any>;
  };
}

export interface ExecutionStatus {
  executionId: string;
  status: 'pending' | 'running' | 'paused' | 'completed' | 'failed' | 'cancelled';
  currentStep?: number;
  totalSteps?: number;
  output: string;
  error?: string;
  startedAt: number;
  completedAt?: number;
}
```

### Mock Tauri Service

```typescript
// src/services/tauri/MockTauriService.ts
import { invoke } from '@tauri-apps/api';
import type * as Types from './types';

export class MockTauriService {
  // ========== CHAT COMMANDS ==========

  async getChatHistory(params: Types.GetChatHistoryParams): Promise<Message[]> {
    await this.delay(100);
    return [
      {
        id: '1',
        role: 'user',
        content: 'Stop RDS instance db-prod-01',
        timestamp: Date.now() - 120000
      },
      {
        id: '2',
        role: 'assistant',
        content: 'I will stop the RDS instance db-prod-01 for you.',
        timestamp: Date.now() - 90000
      }
    ];
  }

  async saveMessage(params: Types.SaveMessageParams): Promise<void> {
    await this.delay(50);
    console.log('[Mock] Saved message:', params);
  }

  async deleteConversation(contextId: string): Promise<void> {
    await this.delay(50);
    console.log('[Mock] Deleted conversation:', contextId);
  }

  // ========== ESTATE COMMANDS ==========

  async getEstateResources(
    params: Types.GetEstateResourcesParams
  ): Promise<Resource[]> {
    await this.delay(200);
    return generateMockResources(params.limit || 10);
  }

  async semanticSearch(params: Types.SemanticSearchParams): Promise<Resource[]> {
    await this.delay(300);
    return generateMockResources(params.limit || 5);
  }

  async getResourceDetails(resourceId: string): Promise<Resource> {
    await this.delay(100);
    return generateMockResources(1)[0];
  }

  // ========== POLICY COMMANDS ==========

  async getPolicy(): Promise<Types.AgentPolicy> {
    await this.delay(100);
    return {
      allowedOperations: {
        ec2: ['stop', 'start', 'describe'],
        rds: ['stop', 'start', 'describe']
      },
      blockedOperations: {
        ec2: ['terminate'],
        rds: ['delete']
      },
      resourceConstraints: {
        tags: { env: 'production' },
        costLimits: 1000
      },
      approvalRequired: {
        stop: false,
        delete: true
      }
    };
  }

  async updatePolicy(policy: Types.AgentPolicy): Promise<void> {
    await this.delay(200);
    console.log('[Mock] Updated policy:', policy);
  }

  // ========== SCANNER COMMANDS ==========

  async startScan(config: Types.ScanConfig): Promise<string> {
    const scanId = `scan-${Date.now()}`;

    // Simulate progress events
    this.simulateScanProgress(scanId);

    return scanId;
  }

  async cancelScan(scanId: string): Promise<void> {
    await this.delay(50);
    console.log('[Mock] Cancelled scan:', scanId);
    this.emitEvent('scan_cancelled', { scanId });
  }

  async getScanResults(scanId: string): Promise<Types.ScanResults> {
    await this.delay(100);
    return {
      scanId,
      totalResources: 42,
      resourcesByService: {
        ec2: 15,
        rds: 10,
        s3: 8,
        lambda: 9
      },
      resourcesByRegion: {
        'us-east-1': 20,
        'us-west-2': 22
      }
    };
  }

  // ========== EXECUTION COMMANDS ==========

  async executeCommand(params: Types.ExecuteCommandParams): Promise<string> {
    const executionId = `exec-${Date.now()}`;

    // Simulate execution output
    this.simulateExecution(executionId, params.command);

    return executionId;
  }

  async executePlaybook(params: Types.ExecutePlaybookParams): Promise<string> {
    const executionId = `pb-${Date.now()}`;

    // Simulate playbook execution
    this.simulatePlaybookExecution(executionId, params.playbook);

    return executionId;
  }

  async cancelExecution(executionId: string): Promise<void> {
    await this.delay(50);
    console.log('[Mock] Cancelled execution:', executionId);
    this.emitEvent('execution_cancelled', { executionId });
  }

  async getExecutionStatus(executionId: string): Promise<Types.ExecutionStatus> {
    await this.delay(50);
    return {
      executionId,
      status: 'completed',
      currentStep: 3,
      totalSteps: 3,
      output: 'Command executed successfully',
      startedAt: Date.now() - 5000,
      completedAt: Date.now()
    };
  }

  // ========== AUTH COMMANDS ==========

  async storeTokens(accessToken: string, refreshToken: string): Promise<void> {
    await this.delay(50);
    console.log('[Mock] Stored tokens');
  }

  async getAccessToken(): Promise<string | null> {
    await this.delay(50);
    return 'mock-access-token';
  }

  async refreshToken(): Promise<string> {
    await this.delay(200);
    return 'mock-new-access-token';
  }

  async clearTokens(): Promise<void> {
    await this.delay(50);
    console.log('[Mock] Cleared tokens');
  }

  // ========== EVENT LISTENERS ==========

  onScanProgress(callback: (data: Types.ScanProgress) => void): () => void {
    return this.addEventListener('scan_progress', callback);
  }

  onScanComplete(callback: (data: any) => void): () => void {
    return this.addEventListener('scan_complete', callback);
  }

  onExecutionOutput(callback: (data: any) => void): () => void {
    return this.addEventListener('execution_output', callback);
  }

  onExecutionComplete(callback: (data: any) => void): () => void {
    return this.addEventListener('execution_complete', callback);
  }

  // ========== HELPERS ==========

  private async delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  private emitEvent(eventName: string, data: any) {
    window.dispatchEvent(new CustomEvent(eventName, { detail: data }));
  }

  private addEventListener(eventName: string, callback: Function): () => void {
    const handler = (e: any) => callback(e.detail);
    window.addEventListener(eventName, handler);
    return () => window.removeEventListener(eventName, handler);
  }

  private simulateScanProgress(scanId: string) {
    const steps = [
      { percent: 10, service: 'EC2' },
      { percent: 30, service: 'RDS' },
      { percent: 50, service: 'S3' },
      { percent: 70, service: 'Lambda' },
      { percent: 90, service: 'IAM' },
      { percent: 100, service: 'Complete' }
    ];

    steps.forEach((step, i) => {
      setTimeout(() => {
        this.emitEvent('scan_progress', {
          scanId,
          status: 'running',
          progress: {
            percentComplete: step.percent,
            currentService: step.service,
            servicesCompleted: i + 1,
            servicesTotal: 6,
            resourcesDiscovered: step.percent * 5
          },
          currentTask: `Scanning ${step.service}...`,
          elapsedSeconds: (i + 1) * 2
        });
      }, i * 1000);
    });

    setTimeout(() => {
      this.emitEvent('scan_complete', {
        scanId,
        resourcesFound: 500
      });
    }, steps.length * 1000);
  }

  private simulateExecution(executionId: string, command: string) {
    const outputs = [
      'Starting execution...',
      `Running command: ${command}`,
      'Processing...',
      'Command completed successfully'
    ];

    outputs.forEach((output, i) => {
      setTimeout(() => {
        this.emitEvent('execution_output', {
          executionId,
          output: output + '\n',
          stream: 'stdout',
          timestamp: new Date().toISOString()
        });
      }, i * 500);
    });

    setTimeout(() => {
      this.emitEvent('execution_complete', {
        executionId,
        exitCode: 0,
        duration: outputs.length * 500
      });
    }, outputs.length * 500 + 100);
  }

  private simulatePlaybookExecution(executionId: string, playbook: Playbook) {
    playbook.steps.forEach((step, i) => {
      setTimeout(() => {
        this.emitEvent('playbook_step_start', {
          executionId,
          stepIndex: i,
          stepDescription: step.description
        });

        setTimeout(() => {
          this.emitEvent('execution_output', {
            executionId,
            output: `Executing: ${step.description}\n`,
            stream: 'stdout',
            timestamp: new Date().toISOString()
          });

          setTimeout(() => {
            this.emitEvent('playbook_step_complete', {
              executionId,
              stepIndex: i,
              success: true
            });
          }, 1000);
        }, 200);
      }, i * 1500);
    });

    setTimeout(() => {
      this.emitEvent('execution_complete', {
        executionId,
        exitCode: 0,
        duration: playbook.steps.length * 1500
      });
    }, playbook.steps.length * 1500 + 200);
  }
}

// Helper function to generate mock resources
function generateMockResources(count: number): Resource[] {
  const resources: Resource[] = [];
  const types = ['ec2_instance', 'rds_instance', 's3_bucket', 'lambda_function'];
  const regions = ['us-east-1', 'us-west-2', 'eu-west-1'];
  const states = ['running', 'stopped', 'available'];

  for (let i = 0; i < count; i++) {
    resources.push({
      id: `resource-${i + 1}`,
      type: types[i % types.length],
      name: `resource-name-${i + 1}`,
      region: regions[i % regions.length],
      accountId: '123456789012',
      state: states[i % states.length],
      tags: {
        env: i % 2 === 0 ? 'production' : 'development',
        app: 'escher'
      },
      metadata: {
        createdAt: Date.now() - Math.random() * 10000000000,
        lastModified: Date.now() - Math.random() * 1000000000
      }
    });
  }

  return resources;
}
```

---

## WebSocket Server Contracts

### Message Types

```typescript
// src/services/websocket/types.ts

// Client → Server messages
export type ClientMessage =
  | ChatMessageRequest
  | PlaybookActionRequest
  | SystemMessage;

export interface ChatMessageRequest {
  type: 'chat_message';
  payload: {
    text: string;
    contextId: string;
    history: MessageHistory[];
  };
}

export interface PlaybookActionRequest {
  type: 'playbook_action';
  payload: {
    action: 'apply' | 'explain' | 'cancel';
    playbookId: string;
  };
}

export interface SystemMessage {
  type: 'ping' | 'subscribe' | 'unsubscribe';
  payload?: any;
}

// Server → Client messages
export type ServerMessage =
  | TextChunkMessage
  | EnhancementMessage
  | PlaybookMessage
  | AlertMessage
  | ErrorMessage;

export interface TextChunkMessage {
  type: 'text_chunk';
  payload: {
    text: string;
    messageId: string;
  };
}

export interface EnhancementMessage {
  type: 'enhancement';
  payload: {
    messageId: string;
    ui: UIEnhancement;
    history: MessageHistory;
  };
}

export interface UIEnhancement {
  type: 'playbook' | 'chart' | 'table' | 'card' | 'alert';
  data: any;
}

export interface PlaybookMessage {
  type: 'playbook';
  payload: Playbook;
}

export interface AlertMessage {
  type: 'alert';
  payload: Alert;
}

export interface ErrorMessage {
  type: 'error';
  payload: {
    code: string;
    message: string;
  };
}
```

### Mock WebSocket Service

```typescript
// src/services/websocket/MockWebSocketService.ts

export class MockWebSocketService {
  private handlers = new Map<string, Function[]>();
  private connected = false;

  connect(url: string) {
    console.log('[Mock WS] Connecting to', url);
    this.connected = true;
    setTimeout(() => {
      this.emit('connected', {});
    }, 100);
  }

  disconnect() {
    console.log('[Mock WS] Disconnecting');
    this.connected = false;
    this.emit('disconnected', {});
  }

  send(message: ClientMessage) {
    if (!this.connected) {
      console.error('[Mock WS] Not connected');
      return;
    }

    console.log('[Mock WS] Sending:', message);

    if (message.type === 'chat_message') {
      this.simulateChatResponse(message.payload.text);
    }
  }

  on(event: string, callback: Function) {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    this.handlers.get(event)!.push(callback);
  }

  off(event: string, callback: Function) {
    const handlers = this.handlers.get(event);
    if (handlers) {
      const index = handlers.indexOf(callback);
      if (index > -1) {
        handlers.splice(index, 1);
      }
    }
  }

  private emit(event: string, data: any) {
    const handlers = this.handlers.get(event) || [];
    handlers.forEach(callback => callback(data));
  }

  private simulateChatResponse(userMessage: string) {
    const messageId = `msg-${Date.now()}`;

    // Phase 1: Stream text response
    const response = `I'll help you with: ${userMessage}. Let me process that...`;
    const words = response.split(' ');

    words.forEach((word, i) => {
      setTimeout(() => {
        this.emit('text_chunk', {
          text: word + ' ',
          messageId
        });
      }, i * 50);
    });

    // Phase 2: Send enhancement
    setTimeout(() => {
      // Detect if message needs a playbook
      const needsPlaybook = userMessage.toLowerCase().includes('stop') ||
                           userMessage.toLowerCase().includes('start') ||
                           userMessage.toLowerCase().includes('create');

      if (needsPlaybook) {
        this.emit('enhancement', {
          messageId,
          ui: {
            type: 'playbook',
            data: {
              id: `pb-${Date.now()}`,
              title: `Execute: ${userMessage}`,
              summary: 'This playbook will perform the requested action.',
              riskLevel: 'medium',
              steps: [
                {
                  description: 'Verify resource state',
                  command: 'aws ec2 describe-instances',
                  estimatedTime: '5s'
                },
                {
                  description: 'Perform action',
                  command: `aws ${userMessage.toLowerCase()}`,
                  estimatedTime: '10s'
                },
                {
                  description: 'Confirm completion',
                  command: 'aws ec2 describe-instances',
                  estimatedTime: '5s'
                }
              ]
            }
          },
          history: {
            role: 'assistant',
            content: response
          }
        });
      } else {
        this.emit('enhancement', {
          messageId,
          ui: {
            type: 'card',
            data: {
              title: 'Response',
              content: 'Here is the information you requested.'
            }
          },
          history: {
            role: 'assistant',
            content: response
          }
        });
      }
    }, words.length * 50 + 500);
  }
}
```

---

## HTTP API Contracts

### Endpoints

```typescript
// src/services/api/endpoints.ts

export const API_ENDPOINTS = {
  // Recommendations
  GET_RECOMMENDATIONS: '/api/recommendations',
  GET_RECOMMENDATION: '/api/recommendations/:id',
  APPLY_RECOMMENDATION: '/api/recommendations/:id/apply',
  POSTPONE_RECOMMENDATION: '/api/recommendations/:id/postpone',
  DISMISS_RECOMMENDATION: '/api/recommendations/:id/dismiss',

  // Alerts
  GET_ALERTS: '/api/alerts',
  GET_ALERT: '/api/alerts/:id',
  ACKNOWLEDGE_ALERT: '/api/alerts/:id/acknowledge',
  RESOLVE_ALERT: '/api/alerts/:id/resolve',

  // Reports
  GET_REPORT: '/api/reports/:id',
  GENERATE_REPORT: '/api/reports/generate',
  EXPORT_REPORT: '/api/reports/:id/export',

  // Health
  HEALTH: '/api/health'
} as const;
```

### Mock API Service

```typescript
// src/services/api/MockAPIService.ts

export class MockAPIService {
  private baseUrl = 'http://localhost:3000';

  // ========== RECOMMENDATIONS ==========

  async getRecommendations(filters?: {
    category?: string;
    impact?: string;
    status?: string;
  }): Promise<Recommendation[]> {
    await this.delay(500);
    return [
      {
        id: 'rec-1',
        title: 'Resize overprovisioned EC2 instances',
        description: 'You have 5 EC2 instances that are underutilized',
        category: 'cost',
        impact: 'high',
        estimatedSavings: 500,
        status: 'pending',
        createdAt: Date.now() - 86400000,
        resources: ['i-12345', 'i-67890']
      },
      {
        id: 'rec-2',
        title: 'Enable encryption on S3 buckets',
        description: '3 S3 buckets are not encrypted at rest',
        category: 'security',
        impact: 'high',
        status: 'pending',
        createdAt: Date.now() - 172800000,
        resources: ['bucket-1', 'bucket-2', 'bucket-3']
      },
      {
        id: 'rec-3',
        title: 'Update outdated Lambda runtimes',
        description: '2 Lambda functions using deprecated Node.js 12',
        category: 'maintenance',
        impact: 'medium',
        status: 'pending',
        createdAt: Date.now() - 259200000,
        resources: ['function-1', 'function-2']
      }
    ];
  }

  async applyRecommendation(id: string): Promise<{ playbookId: string }> {
    await this.delay(300);
    console.log('[Mock API] Applying recommendation:', id);
    return { playbookId: `pb-${Date.now()}` };
  }

  async postponeRecommendation(id: string, until: number): Promise<void> {
    await this.delay(200);
    console.log('[Mock API] Postponed recommendation:', id, 'until', new Date(until));
  }

  async dismissRecommendation(id: string, reason?: string): Promise<void> {
    await this.delay(200);
    console.log('[Mock API] Dismissed recommendation:', id, 'reason:', reason);
  }

  // ========== ALERTS ==========

  async getAlerts(filters?: {
    severity?: string;
    status?: string;
    category?: string;
  }): Promise<Alert[]> {
    await this.delay(400);
    return [
      {
        id: 'alert-1',
        title: 'Cost spike detected',
        description: 'Your AWS spend increased 30% this week',
        severity: 'warning',
        category: 'cost',
        status: 'unread',
        timestamp: Date.now() - 3600000,
        resources: []
      },
      {
        id: 'alert-2',
        title: 'Unauthorized access attempt',
        description: 'Failed login attempts from unknown IP',
        severity: 'critical',
        category: 'security',
        status: 'unread',
        timestamp: Date.now() - 7200000,
        resources: []
      },
      {
        id: 'alert-3',
        title: 'RDS backup failed',
        description: 'Automated backup failed for db-prod-01',
        severity: 'high',
        category: 'availability',
        status: 'read',
        timestamp: Date.now() - 86400000,
        resources: ['db-prod-01']
      }
    ];
  }

  async acknowledgeAlert(id: string): Promise<void> {
    await this.delay(200);
    console.log('[Mock API] Acknowledged alert:', id);
  }

  async resolveAlert(id: string, resolution?: string): Promise<void> {
    await this.delay(200);
    console.log('[Mock API] Resolved alert:', id, 'resolution:', resolution);
  }

  // ========== REPORTS ==========

  async generateReport(params: {
    type: string;
    period: string;
    filters?: Record<string, any>;
  }): Promise<Report> {
    await this.delay(1000);
    return {
      id: `report-${Date.now()}`,
      type: params.type,
      period: params.period,
      generatedAt: Date.now(),
      data: {
        summary: {
          totalCost: 1234.56,
          resourceCount: 42,
          alerts: 3
        },
        charts: [
          {
            type: 'line',
            title: 'Daily Cost Trend',
            data: Array.from({ length: 30 }, (_, i) => ({
              date: new Date(Date.now() - (29 - i) * 86400000).toISOString().split('T')[0],
              cost: Math.random() * 100 + 20
            }))
          }
        ],
        tables: [
          {
            title: 'Top Resources by Cost',
            columns: ['Resource', 'Type', 'Cost'],
            rows: [
              ['i-12345', 'EC2 Instance', '$45.20'],
              ['db-prod-01', 'RDS Instance', '$123.45'],
              ['bucket-logs', 'S3 Bucket', '$12.34']
            ]
          }
        ]
      }
    };
  }

  async exportReport(id: string, format: 'pdf' | 'csv'): Promise<Blob> {
    await this.delay(800);
    console.log('[Mock API] Exporting report:', id, 'format:', format);
    return new Blob(['Mock report content'], { type: 'application/octet-stream' });
  }

  // ========== HELPERS ==========

  private async delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## Mock Data Generators

```typescript
// src/utils/mockDataGenerators.ts

export function generateMockMessages(count: number): Message[] {
  const messages: Message[] = [];
  const roles: Array<'user' | 'assistant'> = ['user', 'assistant'];

  for (let i = 0; i < count; i++) {
    messages.push({
      id: `msg-${i + 1}`,
      role: roles[i % 2],
      content: i % 2 === 0
        ? `User message ${Math.floor(i / 2) + 1}`
        : `Assistant response ${Math.floor(i / 2) + 1}`,
      timestamp: Date.now() - (count - i) * 60000
    });
  }

  return messages;
}

export function generateMockPlaybook(title: string): Playbook {
  return {
    id: `pb-${Date.now()}`,
    title,
    summary: `This playbook will execute ${title.toLowerCase()}`,
    riskLevel: 'medium',
    estimatedDuration: 180,
    steps: [
      {
        description: 'Verify current state',
        command: 'aws ec2 describe-instances --instance-ids i-12345',
        estimatedTime: '5s'
      },
      {
        description: 'Execute main action',
        command: `aws ec2 ${title.toLowerCase()}-instances --instance-ids i-12345`,
        estimatedTime: '30s'
      },
      {
        description: 'Confirm completion',
        command: 'aws ec2 describe-instances --instance-ids i-12345',
        estimatedTime: '5s'
      }
    ],
    createdAt: Date.now()
  };
}

export function generateMockResources(count: number, type?: string): Resource[] {
  const types = type ? [type] : ['ec2_instance', 'rds_instance', 's3_bucket', 'lambda_function'];
  const regions = ['us-east-1', 'us-west-2', 'eu-west-1'];
  const states = ['running', 'stopped', 'available', 'active'];

  return Array.from({ length: count }, (_, i) => ({
    id: `resource-${i + 1}`,
    type: types[i % types.length],
    name: `resource-name-${i + 1}`,
    region: regions[i % regions.length],
    accountId: '123456789012',
    state: states[i % states.length],
    tags: {
      env: i % 2 === 0 ? 'production' : 'development',
      app: 'escher',
      owner: `team-${i % 3 + 1}`
    },
    metadata: {
      createdAt: Date.now() - Math.random() * 10000000000,
      lastModified: Date.now() - Math.random() * 1000000000
    }
  }));
}
```

---

## Usage Example

```typescript
// src/App.tsx
import { MockTauriService } from '@/services/tauri/MockTauriService';
import { MockWebSocketService } from '@/services/websocket/MockWebSocketService';
import { MockAPIService } from '@/services/api/MockAPIService';

// Use mocks during development
const tauri = new MockTauriService();
const ws = new MockWebSocketService();
const api = new MockAPIService();

// Later, swap to real implementations
// const tauri = new TauriService();
// const ws = new WebSocketService('ws://server-url');
// const api = new APIService('https://api-url');
```

---

## Contract Agreement Checklist

Before integration, ensure:

- [ ] All TypeScript interfaces match between UI, Platform, and Server teams
- [ ] Mock implementations return data in the correct format
- [ ] Event payloads match expected structure
- [ ] Error handling patterns are consistent
- [ ] All 70+ Tauri commands have TypeScript signatures
- [ ] WebSocket message types are documented
- [ ] HTTP API endpoints are versioned

**These contracts enable fully independent development!** 🎉