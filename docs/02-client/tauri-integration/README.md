# Tauri Integration

**Purpose**: Bridge between React frontend and Rust backend modules
**Technology**: Tauri 2.0 IPC (Inter-Process Communication)
**Status**: ✅ Commands and Events complete, Setup pending

---

## Documentation Status

✅ **Complete**:

**Commands** (70+ documented):
- [Storage Service Commands](./commands-storage.md) - 25+ commands documented
- [Execution Engine Commands](./commands-execution.md) - 15+ commands documented
- [Estate Scanner Commands](./commands-estate-scanner.md) - 15+ commands documented
- [Authentication Commands](./commands-auth.md) - 15+ commands documented

**Events** (15+ documented):
- [Scan Events](./events-scan.md) - Real-time scan progress and completion
- [Execution Events](./events-execution.md) - Command output streaming and playbook tracking
- [System Events](./events-system.md) - Session, network, notifications, errors

⚠️ **Preliminary**:
- [Request Builder Commands](./commands-request-builder.md) - Design phase, pending architecture decisions

⏳ **Pending**:
- Setup and configuration guides
- TypeScript service integration guide

---

## Overview

Tauri provides the communication layer between the frontend (React/TypeScript) and backend (Rust modules). This integration enables the desktop app to leverage native Rust performance while maintaining a modern web-based UI.

```
┌─────────────────────────────────────────────────────┐
│          React Frontend (TypeScript)                │
│  • UI Components                                    │
│  • Controllers                                      │
│  • State Management                                 │
└─────────────────────────────────────────────────────┘
                        ↕
                   Tauri IPC
                        ↕
┌─────────────────────────────────────────────────────┐
│           Rust Backend (Tauri)                      │
│  • Storage Service                                  │
│  • Execution Engine                                 │
│  • Estate Scanner                                   │
│  • Request Builder                                  │
└─────────────────────────────────────────────────────┘
```

---

## Communication Patterns

### 1. Commands (Frontend → Rust)

**Pattern**: Frontend invokes Rust functions

```typescript
// Frontend calls Rust
const result = await invoke('command_name', { param1, param2 });
```

```rust
// Rust handler
#[tauri::command]
async fn command_name(param1: String, param2: i32) -> Result<String, String> {
    // Implementation
    Ok(result)
}
```

**Use Cases**:
- Query data from Storage Service
- Trigger estate scans
- Execute commands
- Manage authentication tokens

---

### 2. Events (Rust → Frontend)

**Pattern**: Rust emits events, frontend listens

```rust
// Rust emits event
app_handle.emit_all("event_name", payload)?;
```

```typescript
// Frontend listens
await listen('event_name', (event) => {
    console.log(event.payload);
});
```

**Use Cases**:
- Real-time scan progress updates
- Command execution output streaming
- Background task notifications

---

## Command Categories

### Storage Service Commands
- **Purpose**: Access local storage, estate data, chat history
- **Module**: `cloudops-storage-service`
- [Complete Command Reference →](./commands-storage.md)

### Execution Engine Commands
- **Purpose**: Execute AWS CLI commands, manage executions
- **Module**: `cloudops-execution-engine`
- [Complete Command Reference →](./commands-execution.md)

### Estate Scanner Commands
- **Purpose**: Trigger scans, get scan status
- **Module**: `cloudops-estate-scanner`
- [Complete Command Reference →](./commands-estate-scanner.md)

### Request Builder Commands
- **Purpose**: Enrich context, prepare server requests
- **Module**: `cloudops-request-builder`
- [Complete Command Reference →](./commands-request-builder.md)

### Authentication Commands
- **Purpose**: Secure token storage in OS keychain
- **Module**: Custom auth module
- [Complete Command Reference →](./commands-auth.md)

---

## Event Categories

### Scan Events
- **Purpose**: Real-time estate scan progress
- [Complete Event Reference →](./events-scan.md)

### Execution Events
- **Purpose**: Command execution output streaming
- [Complete Event Reference →](./events-execution.md)

### System Events
- **Purpose**: App lifecycle, errors, notifications
- [Complete Event Reference →](./events-system.md)

---

## TypeScript Service Wrappers

All Tauri commands are wrapped in TypeScript services for:
- Type safety
- Error handling
- Consistent API
- Easier testing (mocking)

**Example Service**:
```typescript
// services/tauri/StorageService.ts
export class StorageService {
  async searchResources(query: string): Promise<AWSResource[]> {
    return await invoke('search_resources', { query });
  }

  async getEstateSummary(): Promise<EstateSummary> {
    return await invoke('get_estate_summary');
  }
}
```

---

## Error Handling

### Rust Error Format
All Tauri commands return `Result<T, String>`:
- Success: `Ok(data)`
- Error: `Err(error_message)`

### Frontend Handling
```typescript
try {
  const result = await invoke('command_name', { params });
  // Handle success
} catch (error) {
  // error is string from Rust
  console.error('Command failed:', error);
  // Show user-friendly message
}
```

---

## Setup & Configuration

### Tauri Configuration
- [Setup Guide →](./setup.md)
- [Security Configuration →](./security.md)
- [Build Configuration →](./build-config.md)

---

## Documentation Structure

```
tauri-integration/
├── README.md                      # This file (overview)
│
├── commands-storage.md            # Storage Service commands
├── commands-execution.md          # Execution Engine commands
├── commands-estate-scanner.md     # Estate Scanner commands
├── commands-request-builder.md    # Request Builder commands
├── commands-auth.md               # Auth/token commands
│
├── events-scan.md                 # Scan progress events
├── events-execution.md            # Execution output events
├── events-system.md               # System events
│
├── setup.md                       # Tauri setup & configuration
├── security.md                    # Security & permissions
├── build-config.md                # Build configuration
│
└── typescript-services.md         # TypeScript wrapper services
```

---

## Quick Reference

### Most Common Commands

```typescript
// Storage
await invoke('search_resources', { query: 'prod' })
await invoke('get_estate_summary')
await invoke('save_chat_history', { conversationId, history })

// Execution
await invoke('execute_command', { command, commandType: 'aws_cli' })
await invoke('cancel_execution', { executionId })

// Estate Scanner
await invoke('start_estate_scan', { accounts, regions, services })

// Auth
await invoke('store_token', { key: 'escher_access_token', value: token })
await invoke('get_token', { key: 'escher_access_token' })
```

### Most Common Events

```typescript
// Scan progress
await listen('scan_progress', (event) => { /* update UI */ })
await listen('scan_complete', (event) => { /* show summary */ })

// Execution output
await listen('execution_output', (event) => { /* stream output */ })
await listen('execution_complete', (event) => { /* show result */ })
```

---

## Best Practices

### 1. Always Use Service Wrappers
❌ Don't call `invoke()` directly in components
✅ Use TypeScript service wrappers

### 2. Handle Errors Gracefully
❌ Don't show raw Rust error messages
✅ Map errors to user-friendly messages

### 3. Type Safety
❌ Don't use `any` types
✅ Define proper TypeScript interfaces matching Rust types

### 4. Async/Await
❌ Don't block the UI
✅ All Tauri commands are async, use await

### 5. Event Cleanup
❌ Don't leave event listeners running
✅ Unlisten when component unmounts

---

## Next Steps

1. Review [Storage Commands →](./commands-storage.md)
2. Review [Execution Commands →](./commands-execution.md)
3. Review [Event System →](./events-scan.md)
4. Setup Tauri [Configuration →](./setup.md)