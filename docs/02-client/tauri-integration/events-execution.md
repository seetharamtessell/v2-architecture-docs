# Execution Events

**Module**: `cloudops-execution-engine`
**Purpose**: Real-time streaming of command execution output and status

---

## Overview

Execution events stream real-time output from running commands (AWS CLI, scripts, playbooks) to the frontend. These events enable:
- Live terminal-style output display
- Real-time command progress
- Playbook step-by-step execution tracking
- Error capture and display
- Execution completion notifications

**Key Pattern**: Output streams continuously via events. Frontend listens and displays output in real-time.

---

## Event List

### 1. `execution_started`

Emitted when a command or playbook execution begins.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ExecutionStartedPayload {
    execution_id: String,
    command: String,
    command_type: String,  // "aws_cli", "script", "playbook"
    started_at: String,
    options: ExecutionOptions,
}

#[derive(Clone, serde::Serialize)]
struct ExecutionOptions {
    working_dir: Option<String>,
    timeout: Option<u64>,
    aws_profile: Option<String>,
    aws_region: Option<String>,
}

app_handle.emit_all("execution_started", ExecutionStartedPayload {
    execution_id: execution.id.clone(),
    command: execution.command.clone(),
    command_type: "aws_cli".to_string(),
    started_at: Utc::now().to_rfc3339(),
    options: execution.options.clone(),
})?;
```

**TypeScript Listener**:
```typescript
interface ExecutionOptions {
  workingDir?: string;
  timeout?: number;
  awsProfile?: string;
  awsRegion?: string;
}

interface ExecutionStartedPayload {
  executionId: string;
  command: string;
  commandType: 'aws_cli' | 'script' | 'playbook';
  startedAt: string;
  options: ExecutionOptions;
}

await listen<ExecutionStartedPayload>('execution_started', (event) => {
  const { executionId, command, commandType } = event.payload;

  console.log(`Execution ${executionId} started`);
  console.log(`Command: ${command}`);

  // Initialize output display
  initializeExecutionOutput(executionId, {
    command,
    type: commandType,
    startedAt: event.payload.startedAt
  });
});
```

---

### 2. `execution_output`

Emitted continuously as the command produces output (stdout/stderr).

**Frequency**: Real-time as output is produced (typically every 10-100ms)

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ExecutionOutputPayload {
    execution_id: String,
    output: String,
    stream: String,  // "stdout" or "stderr"
    timestamp: String,
}

// Stream stdout
app_handle.emit_all("execution_output", ExecutionOutputPayload {
    execution_id: execution.id.clone(),
    output: "Listing EC2 instances...\n".to_string(),
    stream: "stdout".to_string(),
    timestamp: Utc::now().to_rfc3339(),
})?;

// Stream stderr
app_handle.emit_all("execution_output", ExecutionOutputPayload {
    execution_id: execution.id.clone(),
    output: "Warning: Deprecated API version\n".to_string(),
    stream: "stderr".to_string(),
    timestamp: Utc::now().to_rfc3339(),
})?;
```

**TypeScript Listener**:
```typescript
interface ExecutionOutputPayload {
  executionId: string;
  output: string;
  stream: 'stdout' | 'stderr';
  timestamp: string;
}

await listen<ExecutionOutputPayload>('execution_output', (event) => {
  const { executionId, output, stream } = event.payload;

  // Append output to terminal display
  appendExecutionOutput(executionId, {
    text: output,
    type: stream,
    timestamp: event.payload.timestamp
  });

  // Apply styling based on stream
  if (stream === 'stderr') {
    // Display in red/warning color
    styleAsError(output);
  }

  console.log(`[${stream}] ${output}`);
});
```

**UI Pattern**: Terminal-style display
```typescript
// React component example
const [outputLines, setOutputLines] = useState<OutputLine[]>([]);

useEffect(() => {
  const setupListener = async () => {
    await listen<ExecutionOutputPayload>('execution_output', (event) => {
      if (event.payload.executionId === currentExecutionId) {
        setOutputLines(prev => [
          ...prev,
          {
            text: event.payload.output,
            type: event.payload.stream,
            timestamp: event.payload.timestamp
          }
        ]);
      }
    });
  };

  setupListener();
}, [currentExecutionId]);

// Render
return (
  <Terminal>
    {outputLines.map((line, idx) => (
      <TerminalLine key={idx} type={line.type}>
        {line.text}
      </TerminalLine>
    ))}
  </Terminal>
);
```

---

### 3. `execution_complete`

Emitted when command execution finishes (success or failure).

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ExecutionCompletePayload {
    execution_id: String,
    status: String,  // "completed", "failed", "timeout", "cancelled"
    started_at: String,
    completed_at: String,
    duration_ms: u64,
    exit_code: Option<i32>,
    error: Option<String>,
}

app_handle.emit_all("execution_complete", ExecutionCompletePayload {
    execution_id: execution.id.clone(),
    status: "completed".to_string(),
    started_at: execution.started_at.to_rfc3339(),
    completed_at: Utc::now().to_rfc3339(),
    duration_ms: execution.duration_ms(),
    exit_code: Some(0),
    error: None,
})?;
```

**TypeScript Listener**:
```typescript
interface ExecutionCompletePayload {
  executionId: string;
  status: 'completed' | 'failed' | 'timeout' | 'cancelled';
  startedAt: string;
  completedAt: string;
  durationMs: number;
  exitCode?: number;
  error?: string;
}

await listen<ExecutionCompletePayload>('execution_complete', (event) => {
  const { executionId, status, exitCode, durationMs, error } = event.payload;

  console.log(`Execution ${executionId} ${status} (${durationMs}ms)`);

  if (status === 'completed' && exitCode === 0) {
    // Success
    showSuccessNotification({
      title: 'Command Completed',
      message: `Execution finished in ${(durationMs / 1000).toFixed(2)}s`
    });

    // Mark execution as complete in UI
    markExecutionComplete(executionId, 'success');
  } else if (status === 'failed' || exitCode !== 0) {
    // Failure
    showErrorNotification({
      title: 'Command Failed',
      message: error || `Exit code: ${exitCode}`
    });

    markExecutionComplete(executionId, 'error');
  } else if (status === 'timeout') {
    showWarningNotification({
      title: 'Command Timeout',
      message: 'Execution exceeded timeout limit'
    });

    markExecutionComplete(executionId, 'timeout');
  }

  // Cleanup: stop listening for this execution
  cleanupExecutionListeners(executionId);
});
```

---

### 4. `execution_cancelled`

Emitted when an execution is cancelled by the user.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ExecutionCancelledPayload {
    execution_id: String,
    cancelled_at: String,
    reason: String,
    partial_output: String,
}

app_handle.emit_all("execution_cancelled", ExecutionCancelledPayload {
    execution_id: execution.id.clone(),
    cancelled_at: Utc::now().to_rfc3339(),
    reason: "User requested cancellation".to_string(),
    partial_output: execution.get_output_so_far(),
})?;
```

**TypeScript Listener**:
```typescript
interface ExecutionCancelledPayload {
  executionId: string;
  cancelledAt: string;
  reason: string;
  partialOutput: string;
}

await listen<ExecutionCancelledPayload>('execution_cancelled', (event) => {
  const { executionId, reason } = event.payload;

  console.log(`Execution ${executionId} cancelled: ${reason}`);

  // Update UI
  showCancelledState(executionId, reason);

  // Show partial output if available
  if (event.payload.partialOutput) {
    displayPartialOutput(executionId, event.payload.partialOutput);
  }
});
```

---

### 5. `playbook_step_start`

Emitted when a playbook step begins (playbook executions only).

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct PlaybookStepStartPayload {
    execution_id: String,
    step_id: String,
    step_number: usize,
    total_steps: usize,
    step_name: String,
    command: String,
}

app_handle.emit_all("playbook_step_start", PlaybookStepStartPayload {
    execution_id: execution.id.clone(),
    step_id: "step_1".to_string(),
    step_number: 1,
    total_steps: 3,
    step_name: "List EC2 Instances".to_string(),
    command: "aws ec2 describe-instances".to_string(),
})?;
```

**TypeScript Listener**:
```typescript
interface PlaybookStepStartPayload {
  executionId: string;
  stepId: string;
  stepNumber: number;
  totalSteps: number;
  stepName: string;
  command: string;
}

await listen<PlaybookStepStartPayload>('playbook_step_start', (event) => {
  const { stepNumber, totalSteps, stepName, command } = event.payload;

  console.log(`Step ${stepNumber}/${totalSteps}: ${stepName}`);
  console.log(`Command: ${command}`);

  // Update playbook UI
  updatePlaybookProgress({
    currentStep: stepNumber,
    totalSteps: totalSteps,
    stepName: stepName
  });

  // Highlight current step
  highlightCurrentStep(event.payload.stepId);
});
```

---

### 6. `playbook_step_complete`

Emitted when a playbook step finishes.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct PlaybookStepCompletePayload {
    execution_id: String,
    step_id: String,
    step_number: usize,
    step_name: String,
    success: bool,
    exit_code: Option<i32>,
    duration_ms: u64,
    error: Option<String>,
    continue_on_error: bool,
}

app_handle.emit_all("playbook_step_complete", PlaybookStepCompletePayload {
    execution_id: execution.id.clone(),
    step_id: "step_1".to_string(),
    step_number: 1,
    step_name: "List EC2 Instances".to_string(),
    success: true,
    exit_code: Some(0),
    duration_ms: 1250,
    error: None,
    continue_on_error: false,
})?;
```

**TypeScript Listener**:
```typescript
interface PlaybookStepCompletePayload {
  executionId: string;
  stepId: string;
  stepNumber: number;
  stepName: string;
  success: boolean;
  exitCode?: number;
  durationMs: number;
  error?: string;
  continueOnError: boolean;
}

await listen<PlaybookStepCompletePayload>('playbook_step_complete', (event) => {
  const { stepId, stepName, success, durationMs, error } = event.payload;

  if (success) {
    console.log(`✓ Step ${event.payload.stepNumber} completed (${durationMs}ms)`);

    // Mark step as complete with success
    markStepComplete(stepId, 'success');
  } else {
    console.error(`✗ Step ${event.payload.stepNumber} failed: ${error}`);

    // Mark step as failed
    markStepComplete(stepId, 'error');

    if (!event.payload.continueOnError) {
      showErrorNotification({
        title: `Step Failed: ${stepName}`,
        message: error || 'Unknown error'
      });
    }
  }
});
```

---

## Usage Patterns

### Pattern 1: Terminal Output Display

Display command output in terminal-style interface.

```typescript
import { listen, UnlistenFn } from '@tauri-apps/api/event';

interface TerminalLine {
  text: string;
  type: 'stdout' | 'stderr';
  timestamp: string;
}

function ExecutionTerminal({ executionId }: { executionId: string }) {
  const [lines, setLines] = useState<TerminalLine[]>([]);
  const [isRunning, setIsRunning] = useState(true);
  const terminalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    let unlistenOutput: UnlistenFn;
    let unlistenComplete: UnlistenFn;

    const setupListeners = async () => {
      // Listen for output
      unlistenOutput = await listen<ExecutionOutputPayload>(
        'execution_output',
        (event) => {
          if (event.payload.executionId === executionId) {
            setLines(prev => [
              ...prev,
              {
                text: event.payload.output,
                type: event.payload.stream,
                timestamp: event.payload.timestamp
              }
            ]);

            // Auto-scroll to bottom
            setTimeout(() => {
              terminalRef.current?.scrollTo({
                top: terminalRef.current.scrollHeight,
                behavior: 'smooth'
              });
            }, 50);
          }
        }
      );

      // Listen for completion
      unlistenComplete = await listen<ExecutionCompletePayload>(
        'execution_complete',
        (event) => {
          if (event.payload.executionId === executionId) {
            setIsRunning(false);
          }
        }
      );
    };

    setupListeners();

    return () => {
      unlistenOutput?.();
      unlistenComplete?.();
    };
  }, [executionId]);

  return (
    <div ref={terminalRef} className="terminal">
      {lines.map((line, idx) => (
        <div
          key={idx}
          className={line.type === 'stderr' ? 'terminal-line-error' : 'terminal-line'}
        >
          {line.text}
        </div>
      ))}
      {isRunning && <div className="terminal-cursor">_</div>}
    </div>
  );
}
```

---

### Pattern 2: Playbook Step Progress

Track and display playbook execution step-by-step.

```typescript
interface PlaybookStep {
  id: string;
  name: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  duration?: number;
  error?: string;
}

function PlaybookExecution({ executionId, steps }: Props) {
  const [stepStates, setStepStates] = useState<Map<string, PlaybookStep>>(
    new Map(steps.map(s => [s.id, { ...s, status: 'pending' }]))
  );

  useEffect(() => {
    const setupListeners = async () => {
      // Step started
      await listen<PlaybookStepStartPayload>('playbook_step_start', (event) => {
        if (event.payload.executionId === executionId) {
          setStepStates(prev => {
            const next = new Map(prev);
            const step = next.get(event.payload.stepId);
            if (step) {
              step.status = 'running';
              next.set(event.payload.stepId, step);
            }
            return next;
          });
        }
      });

      // Step completed
      await listen<PlaybookStepCompletePayload>('playbook_step_complete', (event) => {
        if (event.payload.executionId === executionId) {
          setStepStates(prev => {
            const next = new Map(prev);
            const step = next.get(event.payload.stepId);
            if (step) {
              step.status = event.payload.success ? 'completed' : 'failed';
              step.duration = event.payload.durationMs;
              step.error = event.payload.error;
              next.set(event.payload.stepId, step);
            }
            return next;
          });
        }
      });
    };

    setupListeners();
  }, [executionId]);

  return (
    <div className="playbook-steps">
      {Array.from(stepStates.values()).map((step, idx) => (
        <PlaybookStepItem
          key={step.id}
          stepNumber={idx + 1}
          step={step}
        />
      ))}
    </div>
  );
}
```

---

### Pattern 3: Execution Manager Service

Centralized service to manage multiple executions.

```typescript
import { listen, UnlistenFn } from '@tauri-apps/api/event';

class ExecutionManager {
  private executions = new Map<string, ExecutionState>();
  private listeners = new Map<string, UnlistenFn[]>();

  async startTracking(executionId: string): Promise<void> {
    // Initialize state
    this.executions.set(executionId, {
      id: executionId,
      status: 'running',
      output: [],
      startedAt: new Date().toISOString()
    });

    // Setup listeners
    const unlistenOutput = await listen<ExecutionOutputPayload>(
      'execution_output',
      (event) => {
        if (event.payload.executionId === executionId) {
          this.handleOutput(event.payload);
        }
      }
    );

    const unlistenComplete = await listen<ExecutionCompletePayload>(
      'execution_complete',
      (event) => {
        if (event.payload.executionId === executionId) {
          this.handleComplete(event.payload);
        }
      }
    );

    this.listeners.set(executionId, [unlistenOutput, unlistenComplete]);
  }

  private handleOutput(payload: ExecutionOutputPayload): void {
    const execution = this.executions.get(payload.executionId);
    if (execution) {
      execution.output.push({
        text: payload.output,
        type: payload.stream,
        timestamp: payload.timestamp
      });
    }
  }

  private handleComplete(payload: ExecutionCompletePayload): void {
    const execution = this.executions.get(payload.executionId);
    if (execution) {
      execution.status = payload.status;
      execution.exitCode = payload.exitCode;
      execution.completedAt = payload.completedAt;
    }

    // Cleanup listeners
    this.stopTracking(payload.executionId);
  }

  stopTracking(executionId: string): void {
    const listeners = this.listeners.get(executionId);
    if (listeners) {
      listeners.forEach(unlisten => unlisten());
      this.listeners.delete(executionId);
    }
  }

  getExecution(executionId: string): ExecutionState | undefined {
    return this.executions.get(executionId);
  }

  getAllExecutions(): ExecutionState[] {
    return Array.from(this.executions.values());
  }
}

export const executionManager = new ExecutionManager();
```

---

## Best Practices

### 1. Filter by Execution ID

```typescript
// Always filter events by execution ID
await listen<ExecutionOutputPayload>('execution_output', (event) => {
  if (event.payload.executionId === currentExecutionId) {
    // Handle only events for this execution
    handleOutput(event.payload);
  }
});
```

### 2. Buffer Output for Performance

```typescript
// Buffer output updates to avoid excessive renders
const outputBuffer: string[] = [];
let flushTimeout: NodeJS.Timeout;

await listen<ExecutionOutputPayload>('execution_output', (event) => {
  outputBuffer.push(event.payload.output);

  clearTimeout(flushTimeout);
  flushTimeout = setTimeout(() => {
    // Flush buffer to UI
    appendToTerminal(outputBuffer.join(''));
    outputBuffer.length = 0;
  }, 100); // Flush every 100ms
});
```

### 3. Limit Output History

```typescript
// Prevent memory issues with long-running commands
const MAX_OUTPUT_LINES = 10000;

setOutputLines(prev => {
  const newLines = [...prev, newLine];
  if (newLines.length > MAX_OUTPUT_LINES) {
    // Keep only last MAX_OUTPUT_LINES
    return newLines.slice(-MAX_OUTPUT_LINES);
  }
  return newLines;
});
```

### 4. Always Cleanup on Completion

```typescript
await listen<ExecutionCompletePayload>('execution_complete', (event) => {
  // Handle completion
  handleExecutionComplete(event.payload);

  // IMPORTANT: Cleanup all listeners for this execution
  cleanupListeners(event.payload.executionId);
});
```

---

## Event Frequency

| Event | Frequency | Notes |
|-------|-----------|-------|
| `execution_started` | Once per execution | First event |
| `execution_output` | 10-100 per second | During active execution |
| `execution_complete` | Once per execution | Terminal event |
| `execution_cancelled` | Once per execution | Terminal event (if cancelled) |
| `playbook_step_start` | Once per step | Playbooks only |
| `playbook_step_complete` | Once per step | Playbooks only |

---

## Performance Notes

- **Buffering**: Output events can be very frequent (100/sec). Buffer updates to avoid excessive renders.
- **Line Limits**: Limit terminal display to last 10,000 lines to prevent memory issues.
- **Auto-scroll**: Debounce auto-scroll to avoid janky scrolling on rapid output.

---

## Next Steps

- [System Events →](./events-system.md)
- [Scan Events →](./events-scan.md)
- [Execution Commands →](./commands-execution.md)