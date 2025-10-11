# Estate Scan Events

**Module**: `cloudops-estate-scanner`
**Purpose**: Real-time progress updates during estate scanning operations

---

## Overview

Estate scan events stream real-time progress from the Rust Estate Scanner to the frontend. These events enable:
- Live progress bars
- Current scan status display
- Resource discovery counts
- Error reporting during scans
- Scan completion notifications

**Key Pattern**: Events are emitted continuously during scan operations. Frontend listens with `listen()` and unlistens when done.

---

## Event List

### 1. `scan_started`

Emitted when an estate scan begins.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ScanStartedPayload {
    scan_id: String,
    scan_type: String,  // "full", "incremental", "quick"
    accounts: Vec<String>,
    regions: Vec<String>,
    services: Vec<String>,
    started_at: String,
}

app_handle.emit_all("scan_started", ScanStartedPayload {
    scan_id: scan.id.clone(),
    scan_type: scan.scan_type.to_string(),
    accounts: scan.accounts.clone(),
    regions: scan.regions.clone(),
    services: scan.services.clone(),
    started_at: Utc::now().to_rfc3339(),
})?;
```

**TypeScript Listener**:
```typescript
interface ScanStartedPayload {
  scanId: string;
  scanType: 'full' | 'incremental' | 'quick';
  accounts: string[];
  regions: string[];
  services: string[];
  startedAt: string;
}

await listen<ScanStartedPayload>('scan_started', (event) => {
  console.log(`Scan ${event.payload.scanId} started`);
  console.log(`Type: ${event.payload.scanType}`);
  console.log(`Scanning ${event.payload.services.length} services`);

  // Update UI
  showScanStatus({
    status: 'running',
    scanId: event.payload.scanId,
    startedAt: event.payload.startedAt
  });
});
```

---

### 2. `scan_progress`

Emitted periodically during scan execution with progress updates.

**Frequency**: Every 1-2 seconds during active scanning

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ScanProgressPayload {
    scan_id: String,
    status: String,
    progress: ScanProgress,
    current_task: Option<String>,
    elapsed_seconds: u64,
}

#[derive(Clone, serde::Serialize)]
struct ScanProgress {
    total_services: usize,
    completed_services: usize,
    current_service: Option<String>,
    current_region: Option<String>,
    resources_found: usize,
    percent_complete: f32,
}

app_handle.emit_all("scan_progress", ScanProgressPayload {
    scan_id: scan.id.clone(),
    status: "running".to_string(),
    progress: ScanProgress {
        total_services: scan.total_services,
        completed_services: scan.completed_services,
        current_service: Some("ec2".to_string()),
        current_region: Some("us-east-1".to_string()),
        resources_found: scan.resources_found,
        percent_complete: (scan.completed_services as f32 / scan.total_services as f32) * 100.0,
    },
    current_task: Some("Scanning EC2 instances in us-east-1".to_string()),
    elapsed_seconds: scan.elapsed_seconds(),
})?;
```

**TypeScript Listener**:
```typescript
interface ScanProgress {
  totalServices: number;
  completedServices: number;
  currentService?: string;
  currentRegion?: string;
  resourcesFound: number;
  percentComplete: number;
}

interface ScanProgressPayload {
  scanId: string;
  status: string;
  progress: ScanProgress;
  currentTask?: string;
  elapsedSeconds: number;
}

await listen<ScanProgressPayload>('scan_progress', (event) => {
  const { scanId, progress, currentTask, elapsedSeconds } = event.payload;

  // Update progress bar
  updateProgressBar(progress.percentComplete);

  // Update status text
  updateStatusText(currentTask || `Scanning ${progress.currentService}...`);

  // Update counters
  updateCounters({
    completed: progress.completedServices,
    total: progress.totalServices,
    resourcesFound: progress.resourcesFound,
    elapsed: elapsedSeconds
  });

  console.log(`Scan ${scanId}: ${progress.percentComplete.toFixed(1)}% complete`);
  console.log(`Found ${progress.resourcesFound} resources so far`);
});
```

---

### 3. `scan_service_complete`

Emitted when a specific service completes scanning.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ScanServiceCompletePayload {
    scan_id: String,
    service: String,
    region: String,
    account_id: String,
    resources_found: usize,
    duration_ms: u64,
    success: bool,
    error: Option<String>,
}

app_handle.emit_all("scan_service_complete", ScanServiceCompletePayload {
    scan_id: scan.id.clone(),
    service: "ec2".to_string(),
    region: "us-east-1".to_string(),
    account_id: "123456789012".to_string(),
    resources_found: 42,
    duration_ms: 3500,
    success: true,
    error: None,
})?;
```

**TypeScript Listener**:
```typescript
interface ScanServiceCompletePayload {
  scanId: string;
  service: string;
  region: string;
  accountId: string;
  resourcesFound: number;
  durationMs: number;
  success: boolean;
  error?: string;
}

await listen<ScanServiceCompletePayload>('scan_service_complete', (event) => {
  const { service, region, resourcesFound, success, error } = event.payload;

  if (success) {
    console.log(`✓ ${service} in ${region}: ${resourcesFound} resources (${event.payload.durationMs}ms)`);

    // Mark service as complete in UI
    markServiceComplete(service, region);
  } else {
    console.error(`✗ ${service} in ${region} failed: ${error}`);

    // Show error in UI
    showServiceError(service, region, error);
  }
});
```

---

### 4. `scan_complete`

Emitted when the entire scan finishes (success or failure).

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ScanCompletePayload {
    scan_id: String,
    status: String,  // "completed", "failed", "cancelled"
    started_at: String,
    completed_at: String,
    duration_seconds: u64,
    summary: ScanSummary,
    errors: Vec<ScanError>,
}

#[derive(Clone, serde::Serialize)]
struct ScanSummary {
    total_resources_found: usize,
    resources_updated: usize,
    resources_deleted: usize,
    services_scanned: usize,
    services_failed: usize,
}

#[derive(Clone, serde::Serialize)]
struct ScanError {
    service: String,
    region: String,
    account_id: String,
    error: String,
    timestamp: String,
}

app_handle.emit_all("scan_complete", ScanCompletePayload {
    scan_id: scan.id.clone(),
    status: "completed".to_string(),
    started_at: scan.started_at.to_rfc3339(),
    completed_at: Utc::now().to_rfc3339(),
    duration_seconds: scan.duration_seconds(),
    summary: ScanSummary {
        total_resources_found: 1234,
        resources_updated: 45,
        resources_deleted: 3,
        services_scanned: 15,
        services_failed: 0,
    },
    errors: vec![],
})?;
```

**TypeScript Listener**:
```typescript
interface ScanSummary {
  totalResourcesFound: number;
  resourcesUpdated: number;
  resourcesDeleted: number;
  servicesScanned: number;
  servicesFailed: number;
}

interface ScanError {
  service: string;
  region: string;
  accountId: string;
  error: string;
  timestamp: string;
}

interface ScanCompletePayload {
  scanId: string;
  status: 'completed' | 'failed' | 'cancelled';
  startedAt: string;
  completedAt: string;
  durationSeconds: number;
  summary: ScanSummary;
  errors: ScanError[];
}

await listen<ScanCompletePayload>('scan_complete', (event) => {
  const { scanId, status, summary, durationSeconds, errors } = event.payload;

  console.log(`Scan ${scanId} ${status}`);
  console.log(`Duration: ${durationSeconds}s`);
  console.log(`Resources found: ${summary.totalResourcesFound}`);

  if (status === 'completed') {
    showSuccessNotification({
      title: 'Scan Complete',
      message: `Found ${summary.totalResourcesFound} resources in ${durationSeconds}s`,
      resourcesUpdated: summary.resourcesUpdated,
      resourcesDeleted: summary.resourcesDeleted
    });

    // Refresh estate data
    refreshEstateSummary();
  } else if (status === 'failed') {
    showErrorNotification({
      title: 'Scan Failed',
      message: `${errors.length} errors occurred`,
      errors
    });
  } else if (status === 'cancelled') {
    showInfoNotification({
      title: 'Scan Cancelled',
      message: `Partial results available: ${summary.totalResourcesFound} resources`
    });
  }

  // Cleanup: stop listening to scan events
  // (unlisten functions should be called)
});
```

---

### 5. `scan_cancelled`

Emitted when a scan is cancelled by the user.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ScanCancelledPayload {
    scan_id: String,
    cancelled_at: String,
    reason: String,
    partial_results: ScanSummary,
}

app_handle.emit_all("scan_cancelled", ScanCancelledPayload {
    scan_id: scan.id.clone(),
    cancelled_at: Utc::now().to_rfc3339(),
    reason: "User requested cancellation".to_string(),
    partial_results: scan.get_partial_summary(),
})?;
```

**TypeScript Listener**:
```typescript
interface ScanCancelledPayload {
  scanId: string;
  cancelledAt: string;
  reason: string;
  partialResults: ScanSummary;
}

await listen<ScanCancelledPayload>('scan_cancelled', (event) => {
  const { scanId, reason, partialResults } = event.payload;

  console.log(`Scan ${scanId} cancelled: ${reason}`);
  console.log(`Partial results: ${partialResults.totalResourcesFound} resources`);

  // Update UI
  showCancelledState({
    scanId,
    message: reason,
    partialResults
  });

  // Ask if user wants to keep partial results
  showPartialResultsDialog(partialResults);
});
```

---

### 6. `scan_error`

Emitted when a non-fatal error occurs during scanning.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct ScanErrorPayload {
    scan_id: String,
    error: ScanError,
    continue_scanning: bool,
}

app_handle.emit_all("scan_error", ScanErrorPayload {
    scan_id: scan.id.clone(),
    error: ScanError {
        service: "s3".to_string(),
        region: "eu-west-1".to_string(),
        account_id: "123456789012".to_string(),
        error: "Access Denied: Insufficient permissions".to_string(),
        timestamp: Utc::now().to_rfc3339(),
    },
    continue_scanning: true,
})?;
```

**TypeScript Listener**:
```typescript
interface ScanErrorPayload {
  scanId: string;
  error: ScanError;
  continueScanning: boolean;
}

await listen<ScanErrorPayload>('scan_error', (event) => {
  const { error, continueScanning } = event.payload;

  console.error(`Scan error in ${error.service}/${error.region}: ${error.error}`);

  // Add error to error list in UI
  addScanError({
    service: error.service,
    region: error.region,
    message: error.error,
    timestamp: error.timestamp
  });

  if (continueScanning) {
    // Non-fatal error, scan continues
    showWarningToast(`Error scanning ${error.service} in ${error.region}`);
  } else {
    // Fatal error, scan will stop
    showErrorNotification({
      title: 'Scan Stopped',
      message: error.error
    });
  }
});
```

---

## Usage Patterns

### Pattern 1: Complete Scan Monitoring

Monitor a scan from start to finish with all events.

```typescript
import { listen, UnlistenFn } from '@tauri-apps/api/event';

class ScanMonitor {
  private unlistenFns: UnlistenFn[] = [];

  async startMonitoring(scanId: string): Promise<void> {
    // Listen to all scan events
    const unlistenStarted = await listen('scan_started', this.handleScanStarted);
    const unlistenProgress = await listen('scan_progress', this.handleScanProgress);
    const unlistenServiceComplete = await listen('scan_service_complete', this.handleServiceComplete);
    const unlistenComplete = await listen('scan_complete', this.handleScanComplete);
    const unlistenCancelled = await listen('scan_cancelled', this.handleScanCancelled);
    const unlistenError = await listen('scan_error', this.handleScanError);

    // Store unlisten functions for cleanup
    this.unlistenFns = [
      unlistenStarted,
      unlistenProgress,
      unlistenServiceComplete,
      unlistenComplete,
      unlistenCancelled,
      unlistenError
    ];
  }

  private handleScanStarted = (event: Event<ScanStartedPayload>) => {
    console.log('Scan started:', event.payload.scanId);
    // Update UI
  };

  private handleScanProgress = (event: Event<ScanProgressPayload>) => {
    // Update progress bar
    updateProgressBar(event.payload.progress.percentComplete);
  };

  private handleServiceComplete = (event: Event<ScanServiceCompletePayload>) => {
    // Mark service complete
  };

  private handleScanComplete = (event: Event<ScanCompletePayload>) => {
    console.log('Scan complete:', event.payload.summary);
    this.stopMonitoring(); // Cleanup
  };

  private handleScanCancelled = (event: Event<ScanCancelledPayload>) => {
    console.log('Scan cancelled');
    this.stopMonitoring(); // Cleanup
  };

  private handleScanError = (event: Event<ScanErrorPayload>) => {
    console.error('Scan error:', event.payload.error);
  };

  stopMonitoring(): void {
    // Cleanup all listeners
    this.unlistenFns.forEach(unlisten => unlisten());
    this.unlistenFns = [];
  }
}

// Usage
const monitor = new ScanMonitor();
await monitor.startMonitoring(scanId);
```

---

### Pattern 2: Progress Bar Updates

Simple progress tracking for UI.

```typescript
interface ScanState {
  scanId: string;
  progress: number;
  currentTask: string;
  resourcesFound: number;
  errors: ScanError[];
}

const [scanState, setScanState] = useState<ScanState | null>(null);

useEffect(() => {
  let unlistenProgress: UnlistenFn;
  let unlistenComplete: UnlistenFn;

  const setupListeners = async () => {
    unlistenProgress = await listen<ScanProgressPayload>('scan_progress', (event) => {
      setScanState({
        scanId: event.payload.scanId,
        progress: event.payload.progress.percentComplete,
        currentTask: event.payload.currentTask || '',
        resourcesFound: event.payload.progress.resourcesFound,
        errors: []
      });
    });

    unlistenComplete = await listen<ScanCompletePayload>('scan_complete', (event) => {
      setScanState({
        scanId: event.payload.scanId,
        progress: 100,
        currentTask: 'Complete',
        resourcesFound: event.payload.summary.totalResourcesFound,
        errors: event.payload.errors
      });
    });
  };

  setupListeners();

  return () => {
    // Cleanup on unmount
    unlistenProgress?.();
    unlistenComplete?.();
  };
}, []);

// Render
return (
  <div>
    {scanState && (
      <>
        <ProgressBar value={scanState.progress} />
        <p>{scanState.currentTask}</p>
        <p>Resources found: {scanState.resourcesFound}</p>
      </>
    )}
  </div>
);
```

---

### Pattern 3: Error Tracking

Track errors during scanning for display.

```typescript
const [scanErrors, setScanErrors] = useState<ScanError[]>([]);

useEffect(() => {
  const setupErrorListener = async () => {
    await listen<ScanErrorPayload>('scan_error', (event) => {
      setScanErrors(prev => [...prev, event.payload.error]);
    });

    await listen<ScanCompletePayload>('scan_complete', (event) => {
      // Add any final errors from completion payload
      if (event.payload.errors.length > 0) {
        setScanErrors(prev => [...prev, ...event.payload.errors]);
      }
    });
  };

  setupErrorListener();
}, []);

// Render error list
return (
  <div>
    {scanErrors.length > 0 && (
      <ErrorList>
        {scanErrors.map((error, idx) => (
          <ErrorItem key={idx}>
            <strong>{error.service}</strong> in {error.region}: {error.error}
          </ErrorItem>
        ))}
      </ErrorList>
    )}
  </div>
);
```

---

## Best Practices

### 1. Always Cleanup Listeners

```typescript
// ❌ Bad: Memory leak
await listen('scan_progress', handler);

// ✅ Good: Cleanup on unmount
useEffect(() => {
  let unlisten: UnlistenFn;

  const setup = async () => {
    unlisten = await listen('scan_progress', handler);
  };

  setup();

  return () => {
    unlisten?.();
  };
}, []);
```

### 2. Filter Events by Scan ID

```typescript
// Multiple scans might be running, filter by ID
await listen<ScanProgressPayload>('scan_progress', (event) => {
  if (event.payload.scanId === currentScanId) {
    // Handle event for this specific scan
    updateProgress(event.payload);
  }
});
```

### 3. Throttle Progress Updates

```typescript
// Don't update UI on every progress event (1-2/sec is too frequent)
import { throttle } from 'lodash';

const throttledUpdate = throttle((progress: number) => {
  updateProgressBar(progress);
}, 500); // Update UI max once per 500ms

await listen<ScanProgressPayload>('scan_progress', (event) => {
  throttledUpdate(event.payload.progress.percentComplete);
});
```

### 4. Handle All Terminal States

```typescript
// Listen for all possible end states
await listen('scan_complete', handleComplete);
await listen('scan_cancelled', handleCancelled);
await listen('scan_error', (event) => {
  if (!event.payload.continueScanning) {
    handleFatalError(event.payload.error);
  }
});
```

---

## Event Frequency

| Event | Frequency | Notes |
|-------|-----------|-------|
| `scan_started` | Once per scan | First event |
| `scan_progress` | Every 1-2 seconds | During active scanning |
| `scan_service_complete` | Once per service | 15-50 times per scan |
| `scan_complete` | Once per scan | Terminal event |
| `scan_cancelled` | Once per scan | Terminal event (if cancelled) |
| `scan_error` | As needed | Variable, typically 0-10 per scan |

---

## Next Steps

- [Execution Events →](./events-execution.md)
- [System Events →](./events-system.md)
- [Storage Commands →](./commands-storage.md)