# System Events

**Purpose**: Application lifecycle, errors, notifications, and system-level events

---

## Overview

System events handle application-wide concerns that aren't specific to scans or executions. These include:
- Session management (token expiry, timeouts)
- Application lifecycle (startup, shutdown)
- Network connectivity
- Background task notifications
- System errors
- User notifications

---

## Event List

### 1. `session_expired`

Emitted when the user session expires (absolute timeout or idle timeout).

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct SessionExpiredPayload {
    reason: String,  // "absolute_timeout", "idle_timeout", "token_expired"
    expired_at: String,
    session_duration: u64,  // seconds
}

app_handle.emit_all("session_expired", SessionExpiredPayload {
    reason: "idle_timeout".to_string(),
    expired_at: Utc::now().to_rfc3339(),
    session_duration: 1800,  // 30 minutes
})?;
```

**TypeScript Listener**:
```typescript
interface SessionExpiredPayload {
  reason: 'absolute_timeout' | 'idle_timeout' | 'token_expired';
  expiredAt: string;
  sessionDuration: number;
}

await listen<SessionExpiredPayload>('session_expired', (event) => {
  const { reason, sessionDuration } = event.payload;

  console.log(`Session expired: ${reason}`);

  // Clear all application state
  clearUserState();
  clearTokens();

  // Show expiry notification
  showSessionExpiredNotification({
    reason: reason === 'idle_timeout'
      ? 'Your session expired due to inactivity'
      : 'Your session has expired',
    duration: Math.floor(sessionDuration / 60) // Convert to minutes
  });

  // Redirect to login
  window.location.href = '/login';
});
```

---

### 2. `token_refresh_success`

Emitted when access tokens are successfully refreshed.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct TokenRefreshSuccessPayload {
    refreshed_at: String,
    expires_in: u64,  // seconds
    next_refresh_in: u64,  // seconds
}

app_handle.emit_all("token_refresh_success", TokenRefreshSuccessPayload {
    refreshed_at: Utc::now().to_rfc3339(),
    expires_in: 3600,  // 1 hour
    next_refresh_in: 3300,  // 55 minutes (refresh 5 min before expiry)
})?;
```

**TypeScript Listener**:
```typescript
interface TokenRefreshSuccessPayload {
  refreshedAt: string;
  expiresIn: number;
  nextRefreshIn: number;
}

await listen<TokenRefreshSuccessPayload>('token_refresh_success', (event) => {
  console.log('Tokens refreshed successfully');
  console.log(`Next refresh in ${event.payload.nextRefreshIn}s`);

  // Update session timer in UI (optional)
  updateSessionTimer(event.payload.expiresIn);
});
```

---

### 3. `token_refresh_failed`

Emitted when token refresh fails (user must re-login).

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct TokenRefreshFailedPayload {
    error: String,
    failed_at: String,
    requires_login: bool,
}

app_handle.emit_all("token_refresh_failed", TokenRefreshFailedPayload {
    error: "Refresh token expired".to_string(),
    failed_at: Utc::now().to_rfc3339(),
    requires_login: true,
})?;
```

**TypeScript Listener**:
```typescript
interface TokenRefreshFailedPayload {
  error: string;
  failedAt: string;
  requiresLogin: boolean;
}

await listen<TokenRefreshFailedPayload>('token_refresh_failed', (event) => {
  console.error('Token refresh failed:', event.payload.error);

  if (event.payload.requiresLogin) {
    // Clear state and redirect to login
    clearUserState();

    showErrorNotification({
      title: 'Session Expired',
      message: 'Please log in again'
    });

    window.location.href = '/login';
  }
});
```

---

### 4. `network_status_changed`

Emitted when network connectivity changes.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct NetworkStatusChangedPayload {
    online: bool,
    changed_at: String,
    previous_status: bool,
}

app_handle.emit_all("network_status_changed", NetworkStatusChangedPayload {
    online: false,
    changed_at: Utc::now().to_rfc3339(),
    previous_status: true,
})?;
```

**TypeScript Listener**:
```typescript
interface NetworkStatusChangedPayload {
  online: boolean;
  changedAt: string;
  previousStatus: boolean;
}

await listen<NetworkStatusChangedPayload>('network_status_changed', (event) => {
  const { online } = event.payload;

  if (online) {
    console.log('Network connection restored');

    showSuccessToast('Back online');

    // Retry failed requests
    retryFailedRequests();

    // Refresh data
    refreshEstateData();
  } else {
    console.log('Network connection lost');

    showErrorToast('You are offline');

    // Enable offline mode
    enableOfflineMode();
  }
});
```

---

### 5. `notification`

General-purpose notification event for alerts, reminders, system messages.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct NotificationPayload {
    id: String,
    title: String,
    message: String,
    severity: String,  // "info", "success", "warning", "error"
    timestamp: String,
    action: Option<NotificationAction>,
}

#[derive(Clone, serde::Serialize)]
struct NotificationAction {
    label: String,
    action_type: String,
    action_data: serde_json::Value,
}

app_handle.emit_all("notification", NotificationPayload {
    id: "notif_123".to_string(),
    title: "New Recommendation Available".to_string(),
    message: "3 cost optimization recommendations found".to_string(),
    severity: "info".to_string(),
    timestamp: Utc::now().to_rfc3339(),
    action: Some(NotificationAction {
        label: "View Recommendations".to_string(),
        action_type: "navigate".to_string(),
        action_data: json!({"route": "/recommendations"}),
    }),
})?;
```

**TypeScript Listener**:
```typescript
interface NotificationAction {
  label: string;
  actionType: string;
  actionData: any;
}

interface NotificationPayload {
  id: string;
  title: string;
  message: string;
  severity: 'info' | 'success' | 'warning' | 'error';
  timestamp: string;
  action?: NotificationAction;
}

await listen<NotificationPayload>('notification', (event) => {
  const { title, message, severity, action } = event.payload;

  // Display notification
  showNotification({
    title,
    message,
    type: severity,
    action: action ? {
      label: action.label,
      onClick: () => handleNotificationAction(action)
    } : undefined
  });
});

function handleNotificationAction(action: NotificationAction): void {
  switch (action.actionType) {
    case 'navigate':
      navigate(action.actionData.route);
      break;
    case 'execute_command':
      executeCommand(action.actionData);
      break;
    case 'open_modal':
      openModal(action.actionData);
      break;
  }
}
```

---

### 6. `background_task_complete`

Emitted when a background task finishes (backups, sync, cleanup, etc.).

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct BackgroundTaskCompletePayload {
    task_id: String,
    task_type: String,  // "backup", "cleanup", "sync", "update_check"
    success: bool,
    completed_at: String,
    duration_ms: u64,
    result: Option<serde_json::Value>,
    error: Option<String>,
}

app_handle.emit_all("background_task_complete", BackgroundTaskCompletePayload {
    task_id: "task_backup_123".to_string(),
    task_type: "backup".to_string(),
    success: true,
    completed_at: Utc::now().to_rfc3339(),
    duration_ms: 2500,
    result: Some(json!({
        "backup_path": "/backups/escher_2025_10_10.db",
        "size_bytes": 10485760
    })),
    error: None,
})?;
```

**TypeScript Listener**:
```typescript
interface BackgroundTaskCompletePayload {
  taskId: string;
  taskType: 'backup' | 'cleanup' | 'sync' | 'update_check';
  success: boolean;
  completedAt: string;
  durationMs: number;
  result?: any;
  error?: string;
}

await listen<BackgroundTaskCompletePayload>('background_task_complete', (event) => {
  const { taskType, success, result, error } = event.payload;

  if (success) {
    console.log(`Background task ${taskType} completed successfully`);

    // Show success notification
    if (taskType === 'backup') {
      showSuccessToast(`Backup created: ${result.backup_path}`);
    } else if (taskType === 'update_check' && result.updateAvailable) {
      showUpdateNotification(result);
    }
  } else {
    console.error(`Background task ${taskType} failed:`, error);

    showErrorToast(`${taskType} task failed: ${error}`);
  }
});
```

---

### 7. `app_update_available`

Emitted when a new application version is available.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct AppUpdateAvailablePayload {
    current_version: String,
    new_version: String,
    release_notes: String,
    download_url: String,
    mandatory: bool,
    checked_at: String,
}

app_handle.emit_all("app_update_available", AppUpdateAvailablePayload {
    current_version: "2.1.0".to_string(),
    new_version: "2.2.0".to_string(),
    release_notes: "- New feature: AI recommendations\n- Bug fixes".to_string(),
    download_url: "https://releases.escher.ai/v2.2.0".to_string(),
    mandatory: false,
    checked_at: Utc::now().to_rfc3339(),
})?;
```

**TypeScript Listener**:
```typescript
interface AppUpdateAvailablePayload {
  currentVersion: string;
  newVersion: string;
  releaseNotes: string;
  downloadUrl: string;
  mandatory: boolean;
  checkedAt: string;
}

await listen<AppUpdateAvailablePayload>('app_update_available', (event) => {
  const { currentVersion, newVersion, releaseNotes, mandatory } = event.payload;

  console.log(`Update available: ${currentVersion} → ${newVersion}`);

  if (mandatory) {
    // Force update
    showMandatoryUpdateDialog({
      newVersion,
      releaseNotes,
      onUpdate: () => downloadAndInstallUpdate(event.payload.downloadUrl)
    });
  } else {
    // Optional update
    showUpdateNotification({
      newVersion,
      releaseNotes,
      onUpdate: () => downloadAndInstallUpdate(event.payload.downloadUrl),
      onDismiss: () => dismissUpdate()
    });
  }
});
```

---

### 8. `system_error`

Emitted when a critical system error occurs.

**Rust Emission**:
```rust
#[derive(Clone, serde::Serialize)]
struct SystemErrorPayload {
    error_id: String,
    error_type: String,  // "database", "storage", "network", "unknown"
    error_message: String,
    timestamp: String,
    recoverable: bool,
    stack_trace: Option<String>,
}

app_handle.emit_all("system_error", SystemErrorPayload {
    error_id: "err_db_001".to_string(),
    error_type: "database".to_string(),
    error_message: "Database connection failed".to_string(),
    timestamp: Utc::now().to_rfc3339(),
    recoverable: true,
    stack_trace: Some("...".to_string()),
})?;
```

**TypeScript Listener**:
```typescript
interface SystemErrorPayload {
  errorId: string;
  errorType: 'database' | 'storage' | 'network' | 'unknown';
  errorMessage: string;
  timestamp: string;
  recoverable: boolean;
  stackTrace?: string;
}

await listen<SystemErrorPayload>('system_error', (event) => {
  const { errorType, errorMessage, recoverable } = event.payload;

  console.error(`System error [${errorType}]: ${errorMessage}`);

  if (recoverable) {
    // Show error and allow retry
    showErrorNotification({
      title: 'System Error',
      message: errorMessage,
      actions: [
        { label: 'Retry', onClick: () => retryOperation() },
        { label: 'Dismiss', onClick: () => {} }
      ]
    });
  } else {
    // Fatal error - show error page
    showFatalErrorScreen({
      errorId: event.payload.errorId,
      message: errorMessage,
      stackTrace: event.payload.stackTrace
    });
  }
});
```

---

## Usage Patterns

### Pattern 1: Global Event Listener Setup

Setup system event listeners once at app startup.

```typescript
// App.tsx or main entry point
import { listen } from '@tauri-apps/api/event';

function App() {
  useEffect(() => {
    const setupSystemListeners = async () => {
      // Session expiry
      await listen('session_expired', handleSessionExpired);

      // Token refresh
      await listen('token_refresh_failed', handleTokenRefreshFailed);

      // Network status
      await listen('network_status_changed', handleNetworkChange);

      // Notifications
      await listen('notification', handleNotification);

      // Background tasks
      await listen('background_task_complete', handleBackgroundTask);

      // System errors
      await listen('system_error', handleSystemError);

      // App updates
      await listen('app_update_available', handleAppUpdate);
    };

    setupSystemListeners();
  }, []); // Run once on mount

  return <Router>...</Router>;
}
```

---

### Pattern 2: Notification Center

Centralized notification management.

```typescript
class NotificationCenter {
  private notifications: NotificationPayload[] = [];
  private listeners: Set<(notifications: NotificationPayload[]) => void> = new Set();

  async init(): Promise<void> {
    await listen<NotificationPayload>('notification', (event) => {
      this.addNotification(event.payload);
    });
  }

  private addNotification(notification: NotificationPayload): void {
    this.notifications.push(notification);

    // Limit to last 50 notifications
    if (this.notifications.length > 50) {
      this.notifications = this.notifications.slice(-50);
    }

    // Notify listeners
    this.notifyListeners();

    // Auto-dismiss after delay (info/success only)
    if (['info', 'success'].includes(notification.severity)) {
      setTimeout(() => {
        this.removeNotification(notification.id);
      }, 5000);
    }
  }

  removeNotification(id: string): void {
    this.notifications = this.notifications.filter(n => n.id !== id);
    this.notifyListeners();
  }

  subscribe(callback: (notifications: NotificationPayload[]) => void): () => void {
    this.listeners.add(callback);
    return () => this.listeners.delete(callback);
  }

  private notifyListeners(): void {
    this.listeners.forEach(listener => listener([...this.notifications]));
  }

  getNotifications(): NotificationPayload[] {
    return [...this.notifications];
  }
}

export const notificationCenter = new NotificationCenter();
```

---

### Pattern 3: Session Management

Handle session lifecycle with system events.

```typescript
class SessionManager {
  private sessionTimer: NodeJS.Timeout | null = null;

  async init(): Promise<void> {
    // Listen for session events
    await listen('session_expired', this.handleSessionExpired);
    await listen('token_refresh_success', this.handleTokenRefresh);
    await listen('token_refresh_failed', this.handleTokenRefreshFailed);

    // Start session monitoring
    this.startSessionMonitoring();
  }

  private startSessionMonitoring(): void {
    // Check session validity every minute
    this.sessionTimer = setInterval(async () => {
      const session = await authService.getCurrentSession();

      if (!session) {
        this.handleSessionExpired({ reason: 'no_session' });
        return;
      }

      // Check if expiring soon (< 5 minutes)
      const expiresIn = new Date(session.expiresAt).getTime() - Date.now();
      if (expiresIn < 5 * 60 * 1000) {
        console.log('Session expiring soon, refreshing tokens...');
        try {
          await authService.refreshAccessToken();
        } catch (error) {
          console.error('Failed to refresh tokens:', error);
        }
      }
    }, 60 * 1000); // Every 1 minute
  }

  private handleSessionExpired = (event: any) => {
    console.log('Session expired:', event);

    // Clear session timer
    if (this.sessionTimer) {
      clearInterval(this.sessionTimer);
      this.sessionTimer = null;
    }

    // Clear all state
    clearUserState();

    // Redirect to login
    window.location.href = '/login';
  };

  private handleTokenRefresh = (event: any) => {
    console.log('Tokens refreshed successfully');
  };

  private handleTokenRefreshFailed = (event: any) => {
    console.error('Token refresh failed:', event);
    this.handleSessionExpired({ reason: 'token_refresh_failed' });
  };
}

export const sessionManager = new SessionManager();
```

---

## Best Practices

### 1. Setup System Listeners Early

```typescript
// ✅ Good: Setup in App component or main entry
function App() {
  useEffect(() => {
    setupSystemListeners();
  }, []);

  return <Router>...</Router>;
}

// ❌ Bad: Setup in nested components
```

### 2. Handle All Critical Events

```typescript
// Always handle these critical events:
- session_expired     // Must redirect to login
- token_refresh_failed // Must handle re-authentication
- system_error        // Must show error UI
- network_status_changed // Must handle offline mode
```

### 3. Don't Block UI on Events

```typescript
// ❌ Bad: Blocking operation
await listen('notification', async (event) => {
  await someSlowOperation(); // Blocks event loop
});

// ✅ Good: Non-blocking
await listen('notification', (event) => {
  // Handle synchronously or queue async work
  queueNotification(event.payload);
});
```

### 4. Cleanup on Unmount

```typescript
// For component-specific listeners
useEffect(() => {
  let unlisten: UnlistenFn;

  const setup = async () => {
    unlisten = await listen('notification', handleNotification);
  };

  setup();

  return () => {
    unlisten?.();
  };
}, []);
```

---

## Event Frequency

| Event | Frequency | Notes |
|-------|-----------|-------|
| `session_expired` | Once per session | Terminal event |
| `token_refresh_success` | Every 55 minutes | During active session |
| `token_refresh_failed` | Rarely | Only on refresh errors |
| `network_status_changed` | As needed | When connectivity changes |
| `notification` | Variable | 0-100 per day |
| `background_task_complete` | Variable | As tasks complete |
| `app_update_available` | Once per update | When update detected |
| `system_error` | Rarely | Only on critical errors |

---

## Next Steps

- [Scan Events →](./events-scan.md)
- [Execution Events →](./events-execution.md)
- [Authentication Commands →](./commands-auth.md)