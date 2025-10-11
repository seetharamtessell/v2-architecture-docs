# Execution Engine Commands

**Module**: `cloudops-execution-engine`
**Purpose**: Execute AWS CLI commands and manage command execution lifecycle

---

## Overview

The Execution Engine handles all AWS command execution. It supports:
- AWS CLI commands
- Custom scripts
- Execution tracking and history
- Real-time output streaming via events
- Cancellation and timeout management

**Key Feature**: All output streams via Tauri events, not command returns

---

## Commands

### 1. Command Execution

#### `execute_command`

Execute an AWS CLI command or script.

**Rust Signature**:
```rust
#[tauri::command]
async fn execute_command(
    command: String,
    command_type: CommandType,
    options: Option<ExecutionOptions>
) -> Result<ExecutionHandle, String>
```

**TypeScript Usage**:
```typescript
type CommandType = 'aws_cli' | 'script' | 'playbook';

interface ExecutionOptions {
  workingDir?: string;
  env?: Record<string, string>;
  timeout?: number;  // seconds
  awsProfile?: string;
  awsRegion?: string;
}

interface ExecutionHandle {
  executionId: string;
  startedAt: string;
  status: 'running' | 'completed' | 'failed' | 'cancelled';
}

// Execute AWS CLI command
const handle = await invoke<ExecutionHandle>('execute_command', {
  command: 'aws ec2 describe-instances --region us-east-1',
  commandType: 'aws_cli',
  options: {
    timeout: 300,
    awsProfile: 'production',
    awsRegion: 'us-east-1'
  }
});

// Listen for output (see events-execution.md)
await listen('execution_output', (event) => {
  console.log(event.payload.output);
});
```

**Returns**: ExecutionHandle immediately (non-blocking)
**Output**: Streamed via `execution_output` event
**Completion**: Signaled via `execution_complete` event

**Errors**:
- `"Invalid command format"` - Command validation failed
- `"AWS CLI not found"` - AWS CLI not installed
- `"Profile not found: {profile}"` - Invalid AWS profile
- `"Failed to start execution: {error}"` - Process spawn failed

---

#### `execute_playbook`

Execute a server-provided playbook with multiple steps.

**Rust Signature**:
```rust
#[tauri::command]
async fn execute_playbook(
    playbook: Playbook,
    options: Option<ExecutionOptions>
) -> Result<ExecutionHandle, String>
```

**TypeScript Usage**:
```typescript
interface Playbook {
  id: string;
  name: string;
  steps: PlaybookStep[];
}

interface PlaybookStep {
  id: string;
  name: string;
  command: string;
  commandType: CommandType;
  continueOnError: boolean;
}

const handle = await invoke<ExecutionHandle>('execute_playbook', {
  playbook: {
    id: 'playbook_123',
    name: 'Stop Dev EC2 Instances',
    steps: [
      {
        id: 'step_1',
        name: 'List instances',
        command: 'aws ec2 describe-instances --filters "Name=tag:Environment,Values=dev"',
        commandType: 'aws_cli',
        continueOnError: false
      },
      {
        id: 'step_2',
        name: 'Stop instances',
        command: 'aws ec2 stop-instances --instance-ids i-abc123',
        commandType: 'aws_cli',
        continueOnError: false
      }
    ]
  },
  options: {
    timeout: 600,
    awsProfile: 'production'
  }
});

// Listen for step progress
await listen('playbook_step_complete', (event) => {
  console.log('Step completed:', event.payload);
});
```

**Events Emitted**:
- `playbook_step_start` - When each step begins
- `playbook_step_complete` - When each step finishes
- `execution_output` - Output from each step
- `execution_complete` - When entire playbook finishes

---

### 2. Execution Management

#### `cancel_execution`

Cancel a running execution.

**Rust Signature**:
```rust
#[tauri::command]
async fn cancel_execution(execution_id: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('cancel_execution', {
  executionId: 'exec_123'
});

// Listen for cancellation confirmation
await listen('execution_cancelled', (event) => {
  console.log('Execution cancelled:', event.payload.executionId);
});
```

**Errors**:
- `"Execution not found: {id}"` - Invalid execution ID
- `"Execution already completed"` - Cannot cancel finished execution
- `"Failed to cancel: {error}"` - Cancellation failed

---

#### `get_execution_status`

Get the current status of an execution.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_execution_status(execution_id: String) -> Result<ExecutionStatus, String>
```

**TypeScript Usage**:
```typescript
interface ExecutionStatus {
  executionId: string;
  status: 'running' | 'completed' | 'failed' | 'cancelled';
  startedAt: string;
  completedAt?: string;
  exitCode?: number;
  error?: string;
  progress?: {
    currentStep?: number;
    totalSteps?: number;
  };
}

const status = await invoke<ExecutionStatus>('get_execution_status', {
  executionId: 'exec_123'
});
```

---

#### `get_active_executions`

List all currently running executions.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_active_executions() -> Result<Vec<ExecutionStatus>, String>
```

**TypeScript Usage**:
```typescript
const activeExecutions = await invoke<ExecutionStatus[]>('get_active_executions');

// Display in UI
activeExecutions.forEach(exec => {
  console.log(`${exec.executionId}: ${exec.status}`);
});
```

---

### 3. Execution History

#### `get_execution_history`

Retrieve execution history with optional filters.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_execution_history(
    filters: Option<ExecutionFilters>,
    pagination: Option<Pagination>
) -> Result<PaginatedExecutions, String>
```

**TypeScript Usage**:
```typescript
interface ExecutionFilters {
  commandType?: CommandType;
  status?: string;
  startDate?: string;
  endDate?: string;
}

interface ExecutionRecord {
  executionId: string;
  command: string;
  commandType: CommandType;
  status: string;
  startedAt: string;
  completedAt?: string;
  duration?: number;  // seconds
  exitCode?: number;
  error?: string;
}

interface PaginatedExecutions {
  executions: ExecutionRecord[];
  totalCount: number;
  page: number;
  pageSize: number;
}

const history = await invoke<PaginatedExecutions>('get_execution_history', {
  filters: {
    commandType: 'aws_cli',
    status: 'completed'
  },
  pagination: {
    page: 1,
    pageSize: 50
  }
});
```

---

#### `get_execution_output`

Retrieve stored output from a completed execution.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_execution_output(execution_id: String) -> Result<ExecutionOutput, String>
```

**TypeScript Usage**:
```typescript
interface ExecutionOutput {
  executionId: string;
  stdout: string;
  stderr: string;
  exitCode: number;
  duration: number;
}

const output = await invoke<ExecutionOutput>('get_execution_output', {
  executionId: 'exec_123'
});

console.log('STDOUT:', output.stdout);
console.log('STDERR:', output.stderr);
```

**Note**: Use this for completed executions. For real-time output, listen to `execution_output` event.

---

#### `delete_execution_history`

Delete execution history records.

**Rust Signature**:
```rust
#[tauri::command]
async fn delete_execution_history(
    execution_ids: Vec<String>
) -> Result<u32, String>
```

**TypeScript Usage**:
```typescript
const deletedCount = await invoke<number>('delete_execution_history', {
  executionIds: ['exec_123', 'exec_456']
});

console.log(`Deleted ${deletedCount} execution records`);
```

---

### 4. AWS Configuration

#### `list_aws_profiles`

List available AWS CLI profiles.

**Rust Signature**:
```rust
#[tauri::command]
async fn list_aws_profiles() -> Result<Vec<AWSProfile>, String>
```

**TypeScript Usage**:
```typescript
interface AWSProfile {
  name: string;
  region?: string;
  accountId?: string;
}

const profiles = await invoke<AWSProfile[]>('list_aws_profiles');

// Display in profile selector
profiles.forEach(profile => {
  console.log(`${profile.name} (${profile.region})`);
});
```

---

#### `get_default_aws_profile`

Get the default AWS profile.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_default_aws_profile() -> Result<String, String>
```

**TypeScript Usage**:
```typescript
const defaultProfile = await invoke<string>('get_default_aws_profile');
```

---

#### `validate_aws_credentials`

Validate AWS credentials for a profile.

**Rust Signature**:
```rust
#[tauri::command]
async fn validate_aws_credentials(profile: String) -> Result<CredentialStatus, String>
```

**TypeScript Usage**:
```typescript
interface CredentialStatus {
  valid: boolean;
  accountId?: string;
  error?: string;
  expiresAt?: string;  // For temporary credentials
}

const status = await invoke<CredentialStatus>('validate_aws_credentials', {
  profile: 'production'
});

if (!status.valid) {
  console.error('Invalid credentials:', status.error);
}
```

---

### 5. Command Templates

#### `get_command_templates`

Get pre-defined command templates for common operations.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_command_templates(
    category: Option<String>
) -> Result<Vec<CommandTemplate>, String>
```

**TypeScript Usage**:
```typescript
interface CommandTemplate {
  id: string;
  name: string;
  description: string;
  category: string;
  command: string;
  parameters: TemplateParameter[];
}

interface TemplateParameter {
  name: string;
  description: string;
  type: 'string' | 'number' | 'boolean' | 'select';
  required: boolean;
  defaultValue?: any;
  options?: string[];  // For select type
}

const templates = await invoke<CommandTemplate[]>('get_command_templates', {
  category: 'ec2'
});

// Example template
// {
//   id: 'stop_instance',
//   name: 'Stop EC2 Instance',
//   command: 'aws ec2 stop-instances --instance-ids {{instance_id}} --region {{region}}',
//   parameters: [
//     { name: 'instance_id', type: 'string', required: true },
//     { name: 'region', type: 'select', options: ['us-east-1', 'us-west-2'], required: true }
//   ]
// }
```

---

#### `execute_template`

Execute a command template with parameters.

**Rust Signature**:
```rust
#[tauri::command]
async fn execute_template(
    template_id: String,
    parameters: HashMap<String, String>,
    options: Option<ExecutionOptions>
) -> Result<ExecutionHandle, String>
```

**TypeScript Usage**:
```typescript
const handle = await invoke<ExecutionHandle>('execute_template', {
  templateId: 'stop_instance',
  parameters: {
    instance_id: 'i-abc123',
    region: 'us-east-1'
  },
  options: {
    awsProfile: 'production'
  }
});
```

---

## TypeScript Service Wrapper

**File**: `src/services/tauri/ExecutionService.ts`

```typescript
import { invoke } from '@tauri-apps/api/tauri';
import { listen, UnlistenFn } from '@tauri-apps/api/event';

export class ExecutionService {
  private outputListeners: Map<string, UnlistenFn> = new Map();

  // Command Execution
  async executeCommand(
    command: string,
    commandType: CommandType,
    options?: ExecutionOptions
  ): Promise<ExecutionHandle> {
    return invoke('execute_command', { command, commandType, options });
  }

  async executePlaybook(playbook: Playbook, options?: ExecutionOptions): Promise<ExecutionHandle> {
    return invoke('execute_playbook', { playbook, options });
  }

  // Execution Management
  async cancelExecution(executionId: string): Promise<boolean> {
    return invoke('cancel_execution', { executionId });
  }

  async getExecutionStatus(executionId: string): Promise<ExecutionStatus> {
    return invoke('get_execution_status', { executionId });
  }

  async getActiveExecutions(): Promise<ExecutionStatus[]> {
    return invoke('get_active_executions');
  }

  // Execution History
  async getExecutionHistory(
    filters?: ExecutionFilters,
    pagination?: Pagination
  ): Promise<PaginatedExecutions> {
    return invoke('get_execution_history', { filters, pagination });
  }

  async getExecutionOutput(executionId: string): Promise<ExecutionOutput> {
    return invoke('get_execution_output', { executionId });
  }

  async deleteExecutionHistory(executionIds: string[]): Promise<number> {
    return invoke('delete_execution_history', { executionIds });
  }

  // AWS Configuration
  async listAWSProfiles(): Promise<AWSProfile[]> {
    return invoke('list_aws_profiles');
  }

  async getDefaultAWSProfile(): Promise<string> {
    return invoke('get_default_aws_profile');
  }

  async validateAWSCredentials(profile: string): Promise<CredentialStatus> {
    return invoke('validate_aws_credentials', { profile });
  }

  // Command Templates
  async getCommandTemplates(category?: string): Promise<CommandTemplate[]> {
    return invoke('get_command_templates', { category });
  }

  async executeTemplate(
    templateId: string,
    parameters: Record<string, string>,
    options?: ExecutionOptions
  ): Promise<ExecutionHandle> {
    return invoke('execute_template', { templateId, parameters, options });
  }

  // Event Listeners
  async listenToExecutionOutput(
    executionId: string,
    callback: (output: string) => void
  ): Promise<void> {
    const unlisten = await listen<{ executionId: string; output: string }>(
      'execution_output',
      (event) => {
        if (event.payload.executionId === executionId) {
          callback(event.payload.output);
        }
      }
    );
    this.outputListeners.set(executionId, unlisten);
  }

  stopListeningToExecution(executionId: string): void {
    const unlisten = this.outputListeners.get(executionId);
    if (unlisten) {
      unlisten();
      this.outputListeners.delete(executionId);
    }
  }

  async listenToExecutionComplete(
    executionId: string,
    callback: (status: ExecutionStatus) => void
  ): Promise<UnlistenFn> {
    return listen<ExecutionStatus>('execution_complete', (event) => {
      if (event.payload.executionId === executionId) {
        callback(event.payload);
        this.stopListeningToExecution(executionId);
      }
    });
  }
}

export const executionService = new ExecutionService();
```

---

## Usage in Controllers

**Example**: PlaybookController using ExecutionService

```typescript
import { executionService } from '@/services/tauri/ExecutionService';

export class PlaybookController {
  async executePlaybook(playbook: Playbook): Promise<string> {
    try {
      const handle = await executionService.executePlaybook(playbook, {
        timeout: 600,
        awsProfile: 'production'
      });

      // Listen for output
      await executionService.listenToExecutionOutput(handle.executionId, (output) => {
        this.handleExecutionOutput(output);
      });

      // Listen for completion
      await executionService.listenToExecutionComplete(handle.executionId, (status) => {
        this.handleExecutionComplete(status);
      });

      return handle.executionId;
    } catch (error) {
      console.error('Failed to execute playbook:', error);
      throw new Error('Playbook execution failed');
    }
  }

  private handleExecutionOutput(output: string): void {
    // Update UI with streaming output
    console.log(output);
  }

  private handleExecutionComplete(status: ExecutionStatus): void {
    // Update UI with final status
    console.log('Execution completed:', status);
  }
}
```

---

## Error Handling

1. **Command Validation Errors**:
```typescript
try {
  await executionService.executeCommand(command, 'aws_cli');
} catch (error) {
  if (error.includes('Invalid command')) {
    showError('Please check your command syntax');
  }
}
```

2. **AWS Credential Errors**:
```typescript
const status = await executionService.validateAWSCredentials(profile);
if (!status.valid) {
  showError(`Invalid credentials for profile ${profile}: ${status.error}`);
}
```

3. **Execution Failures**:
```typescript
await executionService.listenToExecutionComplete(executionId, (status) => {
  if (status.status === 'failed') {
    showError(`Execution failed: ${status.error}`);
  }
});
```

---

## Performance Notes

- **Non-blocking**: All execute commands return immediately with ExecutionHandle
- **Streaming**: Output streams in real-time via events (no buffering)
- **Cancellation**: Always cleanup event listeners when cancelling or on completion
- **History**: Limit history queries with pagination to avoid loading thousands of records

---

## Security Notes

- **AWS Credentials**: Never stored in frontend, always read from AWS CLI configuration
- **Command Validation**: All commands validated before execution
- **Sandboxing**: Execution engine runs in separate process with limited permissions
- **Logging**: All commands logged for audit trail (stored in Storage Service)

---

## Next Steps

- [Estate Scanner Commands →](./commands-estate-scanner.md)
- [Execution Events →](./events-execution.md)
- [Request Builder Commands →](./commands-request-builder.md)