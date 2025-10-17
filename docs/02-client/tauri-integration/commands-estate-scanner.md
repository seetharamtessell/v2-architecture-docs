# Estate Scanner Commands

**Module**: `cloudops-estate-scanner`
**Purpose**: Trigger multi-cloud estate scans (AWS/Azure/GCP) and monitor scan progress

---

## Overview

The Estate Scanner discovers and catalogs all cloud resources (AWS/Azure/GCP) across accounts, regions, and services. It provides:
- Full estate scans
- Incremental scans
- Service-specific scans
- Real-time progress tracking via events
- Scan history and statistics

**Key Feature**: Scan progress streams via Tauri events, not command returns

---

## Commands

### 1. Scan Triggers

#### `start_estate_scan`

Start a comprehensive estate scan.

**Rust Signature**:
```rust
#[tauri::command]
async fn start_estate_scan(
    config: ScanConfig
) -> Result<ScanHandle, String>
```

**TypeScript Usage**:
```typescript
interface ScanConfig {
  cloudProvider: 'aws' | 'azure' | 'gcp';  // Cloud provider
  accounts: string[];  // Account IDs (AWS), Subscription IDs (Azure), Project IDs (GCP)
  regions: string[];   // Cloud regions
  services: string[];  // Cloud services (e.g., 'ec2', 'compute', 'vm')
  scanType: 'full' | 'incremental' | 'quick';
  parallelism?: number;  // Number of concurrent scans (default: 5)
}

interface ScanHandle {
  scanId: string;
  startedAt: string;
  status: 'running' | 'completed' | 'failed' | 'cancelled';
  config: ScanConfig;
}

// AWS Full estate scan
const awsHandle = await invoke<ScanHandle>('start_estate_scan', {
  config: {
    cloudProvider: 'aws',
    accounts: ['123456789012', '987654321098'],
    regions: ['us-east-1', 'us-west-2', 'eu-west-1'],
    services: ['ec2', 's3', 'rds', 'lambda', 'dynamodb'],
    scanType: 'full',
    parallelism: 10
  }
});

// Azure Full estate scan
const azureHandle = await invoke<ScanHandle>('start_estate_scan', {
  config: {
    cloudProvider: 'azure',
    accounts: ['sub-12345', 'sub-67890'],  // Subscription IDs
    regions: ['eastus', 'westus2', 'westeurope'],
    services: ['vm', 'sql-database', 'blob-storage', 'functions', 'key-vault'],
    scanType: 'full',
    parallelism: 10
  }
});

// GCP Full estate scan
const gcpHandle = await invoke<ScanHandle>('start_estate_scan', {
  config: {
    cloudProvider: 'gcp',
    accounts: ['my-project-123', 'my-project-456'],  // Project IDs
    regions: ['us-central1', 'us-west1', 'europe-west1'],
    services: ['compute-engine', 'cloud-sql', 'cloud-storage', 'cloud-functions'],
    scanType: 'full',
    parallelism: 10
  }
});

// Listen for progress (see events-scan.md)
await listen('scan_progress', (event) => {
  console.log(`Progress: ${event.payload.progress}%`);
});

// Listen for completion
await listen('scan_complete', (event) => {
  console.log('Scan complete:', event.payload);
});
```

**Returns**: ScanHandle immediately (non-blocking)
**Progress**: Streamed via `scan_progress` event
**Completion**: Signaled via `scan_complete` event

**Errors**:
- `"Invalid scan configuration: {error}"` - Configuration validation failed
- `"No cloud credentials found for {provider}"` - Cloud credentials not configured (AWS/Azure/GCP)
- `"Account not accessible: {accountId}"` - Cannot access account/subscription/project
- `"Failed to start scan: {error}"` - Scanner initialization failed

---

#### `start_quick_scan`

Start a quick scan (metadata only, no deep inspection).

**Rust Signature**:
```rust
#[tauri::command]
async fn start_quick_scan(
    accounts: Vec<String>,
    regions: Vec<String>
) -> Result<ScanHandle, String>
```

**TypeScript Usage**:
```typescript
// Quick scan for estate overview
const handle = await invoke<ScanHandle>('start_quick_scan', {
  accounts: ['123456789012'],
  regions: ['us-east-1', 'us-west-2']
});
```

**Note**: Quick scans collect resource counts and basic metadata only. Use for rapid estate discovery.

---

#### `start_service_scan`

Scan a specific AWS service across accounts/regions.

**Rust Signature**:
```rust
#[tauri::command]
async fn start_service_scan(
    service: String,
    accounts: Vec<String>,
    regions: Vec<String>
) -> Result<ScanHandle, String>
```

**TypeScript Usage**:
```typescript
// Scan only EC2 resources
const handle = await invoke<ScanHandle>('start_service_scan', {
  service: 'ec2',
  accounts: ['123456789012'],
  regions: ['us-east-1', 'us-west-2', 'eu-west-1']
});
```

---

#### `start_incremental_scan`

Scan only resources that changed since last scan.

**Rust Signature**:
```rust
#[tauri::command]
async fn start_incremental_scan(
    accounts: Vec<String>,
    regions: Vec<String>,
    services: Vec<String>
) -> Result<ScanHandle, String>
```

**TypeScript Usage**:
```typescript
// Update only changed resources
const handle = await invoke<ScanHandle>('start_incremental_scan', {
  accounts: ['123456789012'],
  regions: ['us-east-1'],
  services: ['ec2', 'rds']
});
```

**Note**: Uses CloudTrail events to identify changes. Much faster than full scan.

---

### 2. Scan Management

#### `cancel_scan`

Cancel a running scan.

**Rust Signature**:
```rust
#[tauri::command]
async fn cancel_scan(scan_id: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('cancel_scan', {
  scanId: 'scan_123'
});

// Listen for cancellation confirmation
await listen('scan_cancelled', (event) => {
  console.log('Scan cancelled:', event.payload.scanId);
});
```

---

#### `get_scan_status`

Get the current status of a scan.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_scan_status(scan_id: String) -> Result<ScanStatus, String>
```

**TypeScript Usage**:
```typescript
interface ScanStatus {
  scanId: string;
  status: 'running' | 'completed' | 'failed' | 'cancelled';
  startedAt: string;
  completedAt?: string;
  duration?: number;  // seconds
  progress: {
    totalServices: number;
    completedServices: number;
    currentService?: string;
    resourcesFound: number;
    percentComplete: number;
  };
  errors?: ScanError[];
}

interface ScanError {
  service: string;
  region: string;
  accountId: string;
  error: string;
  timestamp: string;
}

const status = await invoke<ScanStatus>('get_scan_status', {
  scanId: 'scan_123'
});

console.log(`Scan ${status.progress.percentComplete}% complete`);
console.log(`Found ${status.progress.resourcesFound} resources`);
```

---

#### `get_active_scans`

List all currently running scans.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_active_scans() -> Result<Vec<ScanStatus>, String>
```

**TypeScript Usage**:
```typescript
const activeScans = await invoke<ScanStatus[]>('get_active_scans');

// Display in UI
activeScans.forEach(scan => {
  console.log(`Scan ${scan.scanId}: ${scan.progress.percentComplete}% complete`);
});
```

---

### 3. Scan History

#### `get_scan_history`

Retrieve scan history with optional filters.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_scan_history(
    filters: Option<ScanFilters>,
    pagination: Option<Pagination>
) -> Result<PaginatedScans, String>
```

**TypeScript Usage**:
```typescript
interface ScanFilters {
  scanType?: 'full' | 'incremental' | 'quick';
  status?: string;
  startDate?: string;
  endDate?: string;
  accountId?: string;
}

interface ScanRecord {
  scanId: string;
  scanType: string;
  status: string;
  startedAt: string;
  completedAt?: string;
  duration?: number;
  resourcesFound: number;
  resourcesUpdated: number;
  resourcesDeleted: number;
  errors: number;
}

interface PaginatedScans {
  scans: ScanRecord[];
  totalCount: number;
  page: number;
  pageSize: number;
}

const history = await invoke<PaginatedScans>('get_scan_history', {
  filters: {
    scanType: 'full',
    status: 'completed'
  },
  pagination: {
    page: 1,
    pageSize: 20
  }
});
```

---

#### `get_last_scan_time`

Get timestamp of the last successful scan.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_last_scan_time() -> Result<Option<String>, String>
```

**TypeScript Usage**:
```typescript
const lastScan = await invoke<string | null>('get_last_scan_time');

if (lastScan) {
  console.log(`Last scan: ${new Date(lastScan).toLocaleString()}`);
} else {
  console.log('No scans completed yet');
}
```

---

#### `delete_scan_history`

Delete scan history records.

**Rust Signature**:
```rust
#[tauri::command]
async fn delete_scan_history(scan_ids: Vec<String>) -> Result<u32, String>
```

**TypeScript Usage**:
```typescript
const deletedCount = await invoke<number>('delete_scan_history', {
  scanIds: ['scan_123', 'scan_456']
});

console.log(`Deleted ${deletedCount} scan records`);
```

---

### 4. Scan Configuration

#### `get_supported_services`

Get list of cloud services supported by the scanner (AWS/Azure/GCP).

**Rust Signature**:
```rust
#[tauri::command]
async fn get_supported_services(
    cloud_provider: String
) -> Result<Vec<CloudService>, String>
```

**TypeScript Usage**:
```typescript
interface CloudService {
  id: string;          // e.g., 'ec2', 'vm', 'compute-engine'
  name: string;        // e.g., 'Amazon EC2', 'Azure Virtual Machines', 'GCP Compute Engine'
  cloudProvider: 'aws' | 'azure' | 'gcp';
  category: string;    // e.g., 'Compute', 'Storage', 'Database'
  resourceTypes: string[];  // e.g., ['instance', 'volume', 'ami']
  scanDuration: number;     // Estimated seconds
}

// Get AWS services
const awsServices = await invoke<CloudService[]>('get_supported_services', {
  cloudProvider: 'aws'
});
// Returns: ['ec2', 's3', 'rds', 'lambda', 'dynamodb', 'vpc', ...]

// Get Azure services
const azureServices = await invoke<CloudService[]>('get_supported_services', {
  cloudProvider: 'azure'
});
// Returns: ['vm', 'sql-database', 'blob-storage', 'functions', 'key-vault', ...]

// Get GCP services
const gcpServices = await invoke<CloudService[]>('get_supported_services', {
  cloudProvider: 'gcp'
});
// Returns: ['compute-engine', 'cloud-sql', 'cloud-storage', 'cloud-functions', ...]

// Group by category
const byCategory = awsServices.reduce((acc, service) => {
  acc[service.category] = acc[service.category] || [];
  acc[service.category].push(service);
  return acc;
}, {} as Record<string, CloudService[]>);
```

---

#### `get_default_scan_config`

Get default scan configuration based on AWS accounts.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_default_scan_config() -> Result<ScanConfig, String>
```

**TypeScript Usage**:
```typescript
const defaultConfig = await invoke<ScanConfig>('get_default_scan_config');

// Modify if needed
defaultConfig.parallelism = 20;

// Start scan with modified config
await invoke('start_estate_scan', { config: defaultConfig });
```

---

#### `validate_scan_config`

Validate a scan configuration before starting.

**Rust Signature**:
```rust
#[tauri::command]
async fn validate_scan_config(config: ScanConfig) -> Result<ValidationResult, String>
```

**TypeScript Usage**:
```typescript
interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
  estimatedDuration?: number;  // seconds
  estimatedResourceCount?: number;
}

const validation = await invoke<ValidationResult>('validate_scan_config', {
  config: {
    accounts: ['123456789012'],
    regions: ['us-east-1', 'us-west-2'],
    services: ['ec2', 's3', 'rds'],
    scanType: 'full'
  }
});

if (!validation.valid) {
  console.error('Invalid configuration:', validation.errors);
} else {
  console.log(`Estimated duration: ${validation.estimatedDuration} seconds`);
  console.log(`Expected resources: ~${validation.estimatedResourceCount}`);
}
```

---

### 5. Scan Statistics

#### `get_scan_statistics`

Get aggregated statistics from scan history.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_scan_statistics() -> Result<ScanStatistics, String>
```

**TypeScript Usage**:
```typescript
interface ScanStatistics {
  totalScans: number;
  successfulScans: number;
  failedScans: number;
  totalResourcesDiscovered: number;
  averageScanDuration: number;  // seconds
  lastScanTimestamp?: string;
  scansByType: Record<string, number>;
  resourcesByService: Record<string, number>;
  scanTrend: {
    date: string;
    count: number;
    resourcesFound: number;
  }[];
}

const stats = await invoke<ScanStatistics>('get_scan_statistics');

console.log(`Total scans: ${stats.totalScans}`);
console.log(`Resources discovered: ${stats.totalResourcesDiscovered}`);
console.log(`Average duration: ${stats.averageScanDuration}s`);
```

---

#### `get_resource_coverage`

Get scan coverage report (which resources have been scanned).

**Rust Signature**:
```rust
#[tauri::command]
async fn get_resource_coverage() -> Result<CoverageReport, String>
```

**TypeScript Usage**:
```typescript
interface CoverageReport {
  accounts: {
    accountId: string;
    regions: {
      region: string;
      services: {
        service: string;
        lastScanned?: string;
        resourceCount: number;
        scanStatus: 'complete' | 'partial' | 'never';
      }[];
    }[];
  }[];
}

const coverage = await invoke<CoverageReport>('get_resource_coverage');

// Identify gaps in coverage
coverage.accounts.forEach(account => {
  account.regions.forEach(region => {
    region.services.forEach(service => {
      if (service.scanStatus === 'never') {
        console.log(`Never scanned: ${service.service} in ${region.region}`);
      }
    });
  });
});
```

---

## TypeScript Service Wrapper

**File**: `src/services/tauri/EstateScannerService.ts`

```typescript
import { invoke } from '@tauri-apps/api/tauri';
import { listen, UnlistenFn } from '@tauri-apps/api/event';

export class EstateScannerService {
  private scanListeners: Map<string, UnlistenFn[]> = new Map();

  // Scan Triggers
  async startEstateScan(config: ScanConfig): Promise<ScanHandle> {
    return invoke('start_estate_scan', { config });
  }

  async startQuickScan(accounts: string[], regions: string[]): Promise<ScanHandle> {
    return invoke('start_quick_scan', { accounts, regions });
  }

  async startServiceScan(
    service: string,
    accounts: string[],
    regions: string[]
  ): Promise<ScanHandle> {
    return invoke('start_service_scan', { service, accounts, regions });
  }

  async startIncrementalScan(
    accounts: string[],
    regions: string[],
    services: string[]
  ): Promise<ScanHandle> {
    return invoke('start_incremental_scan', { accounts, regions, services });
  }

  // Scan Management
  async cancelScan(scanId: string): Promise<boolean> {
    return invoke('cancel_scan', { scanId });
  }

  async getScanStatus(scanId: string): Promise<ScanStatus> {
    return invoke('get_scan_status', { scanId });
  }

  async getActiveScans(): Promise<ScanStatus[]> {
    return invoke('get_active_scans');
  }

  // Scan History
  async getScanHistory(
    filters?: ScanFilters,
    pagination?: Pagination
  ): Promise<PaginatedScans> {
    return invoke('get_scan_history', { filters, pagination });
  }

  async getLastScanTime(): Promise<string | null> {
    return invoke('get_last_scan_time');
  }

  async deleteScanHistory(scanIds: string[]): Promise<number> {
    return invoke('delete_scan_history', { scanIds });
  }

  // Scan Configuration
  async getSupportedServices(): Promise<AWSService[]> {
    return invoke('get_supported_services');
  }

  async getDefaultScanConfig(): Promise<ScanConfig> {
    return invoke('get_default_scan_config');
  }

  async validateScanConfig(config: ScanConfig): Promise<ValidationResult> {
    return invoke('validate_scan_config', { config });
  }

  // Scan Statistics
  async getScanStatistics(): Promise<ScanStatistics> {
    return invoke('get_scan_statistics');
  }

  async getResourceCoverage(): Promise<CoverageReport> {
    return invoke('get_resource_coverage');
  }

  // Event Listeners
  async listenToScanProgress(
    scanId: string,
    callback: (progress: ScanProgress) => void
  ): Promise<void> {
    const unlisten = await listen<ScanProgress>('scan_progress', (event) => {
      if (event.payload.scanId === scanId) {
        callback(event.payload);
      }
    });

    const listeners = this.scanListeners.get(scanId) || [];
    listeners.push(unlisten);
    this.scanListeners.set(scanId, listeners);
  }

  async listenToScanComplete(
    scanId: string,
    callback: (result: ScanResult) => void
  ): Promise<void> {
    const unlisten = await listen<ScanResult>('scan_complete', (event) => {
      if (event.payload.scanId === scanId) {
        callback(event.payload);
        this.stopListeningToScan(scanId);
      }
    });

    const listeners = this.scanListeners.get(scanId) || [];
    listeners.push(unlisten);
    this.scanListeners.set(scanId, listeners);
  }

  stopListeningToScan(scanId: string): void {
    const listeners = this.scanListeners.get(scanId);
    if (listeners) {
      listeners.forEach(unlisten => unlisten());
      this.scanListeners.delete(scanId);
    }
  }
}

export const estateScannerService = new EstateScannerService();
```

---

## Usage in Controllers

**Example**: EstateScanController using EstateScannerService

```typescript
import { estateScannerService } from '@/services/tauri/EstateScannerService';

export class EstateScanController {
  async startFullScan(accounts: string[], regions: string[]): Promise<string> {
    try {
      // Get supported services
      const services = await estateScannerService.getSupportedServices();
      const serviceIds = services.map(s => s.id);

      // Create config
      const config: ScanConfig = {
        accounts,
        regions,
        services: serviceIds,
        scanType: 'full',
        parallelism: 10
      };

      // Validate config
      const validation = await estateScannerService.validateScanConfig(config);
      if (!validation.valid) {
        throw new Error(`Invalid configuration: ${validation.errors.join(', ')}`);
      }

      // Start scan
      const handle = await estateScannerService.startEstateScan(config);

      // Listen for progress
      await estateScannerService.listenToScanProgress(handle.scanId, (progress) => {
        this.handleScanProgress(progress);
      });

      // Listen for completion
      await estateScannerService.listenToScanComplete(handle.scanId, (result) => {
        this.handleScanComplete(result);
      });

      return handle.scanId;
    } catch (error) {
      console.error('Failed to start scan:', error);
      throw new Error('Scan failed to start');
    }
  }

  private handleScanProgress(progress: ScanProgress): void {
    console.log(`Scan ${progress.percentComplete}% complete`);
    // Update UI
  }

  private handleScanComplete(result: ScanResult): void {
    console.log(`Scan complete: ${result.resourcesFound} resources found`);
    // Update UI, refresh estate data
  }
}
```

---

## Error Handling

1. **Configuration Errors**:
```typescript
try {
  const handle = await estateScannerService.startEstateScan(config);
} catch (error) {
  if (error.includes('Invalid scan configuration')) {
    showError('Please check your scan settings');
  }
}
```

2. **AWS Access Errors**:
```typescript
estateScannerService.listenToScanProgress(scanId, (progress) => {
  if (progress.errors && progress.errors.length > 0) {
    progress.errors.forEach(err => {
      console.error(`Scan error in ${err.service}/${err.region}: ${err.error}`);
    });
  }
});
```

3. **Scan Failures**:
```typescript
estateScannerService.listenToScanComplete(scanId, (result) => {
  if (result.status === 'failed') {
    showError(`Scan failed: ${result.error}`);
  }
});
```

---

## Performance Notes

- **Parallelism**: Default is 5 concurrent scans. Increase for faster scans (up to 20 recommended)
- **Incremental Scans**: Use for daily updates (10x faster than full scans)
- **Quick Scans**: Use for initial estate discovery (50x faster, metadata only)
- **Service Scans**: Use when you only need specific resource types

---

## Best Practices

1. **First Scan**: Use `start_quick_scan` to get rapid overview
2. **Regular Updates**: Use `start_incremental_scan` daily
3. **Deep Inspection**: Use `start_estate_scan` with `scanType: 'full'` weekly
4. **Error Handling**: Always listen to `scan_progress` to catch errors early
5. **Cleanup**: Call `stopListeningToScan()` when component unmounts

---

## Next Steps

- [Scan Events →](./events-scan.md)
- [Request Builder Commands →](./commands-request-builder.md)
- [Authentication Commands →](./commands-auth.md)