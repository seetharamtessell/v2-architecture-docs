# Authentication Commands

**Module**: Custom auth module (Tauri plugin or custom implementation)
**Purpose**: Secure token storage in OS keychain and session management

---

## Overview

Authentication commands provide secure storage for JWT tokens and session management. All tokens are stored in the OS keychain (not localStorage or files) for maximum security.

**Storage Mechanism**:
- **macOS**: Keychain Access
- **Windows**: Windows Credential Manager
- **Linux**: Secret Service API (GNOME Keyring, KWallet)

---

## Commands

### 1. Token Storage

#### `store_token`

Store a token securely in the OS keychain.

**Rust Signature**:
```rust
#[tauri::command]
async fn store_token(key: String, value: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
// Store access token
const success = await invoke<boolean>('store_token', {
  key: 'escher_access_token',
  value: 'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...'
});

// Store refresh token
await invoke<boolean>('store_token', {
  key: 'escher_refresh_token',
  value: 'refresh_token_here'
});

// Store ID token
await invoke<boolean>('store_token', {
  key: 'escher_id_token',
  value: 'id_token_here'
});
```

**Key Names** (standardized):
- `escher_access_token` - Access token (1 hour TTL)
- `escher_id_token` - ID token with user info
- `escher_refresh_token` - Refresh token (30 days TTL)

**Errors**:
- `"Keychain access denied"` - User denied keychain access
- `"Failed to store token: {error}"` - OS keychain error
- `"Invalid token format"` - Token validation failed

---

#### `get_token`

Retrieve a token from the OS keychain.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_token(key: String) -> Result<Option<String>, String>
```

**TypeScript Usage**:
```typescript
const accessToken = await invoke<string | null>('get_token', {
  key: 'escher_access_token'
});

if (accessToken) {
  // Token found, validate expiry
  const isValid = await validateToken(accessToken);
  if (!isValid) {
    // Token expired, refresh it
    await refreshAccessToken();
  }
} else {
  // No token found, user needs to login
  redirectToLogin();
}
```

**Returns**: Token string or `null` if not found

---

#### `delete_token`

Delete a token from the OS keychain.

**Rust Signature**:
```rust
#[tauri::command]
async fn delete_token(key: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
// Delete single token
const success = await invoke<boolean>('delete_token', {
  key: 'escher_access_token'
});

// Logout: delete all tokens
await invoke('delete_token', { key: 'escher_access_token' });
await invoke('delete_token', { key: 'escher_id_token' });
await invoke('delete_token', { key: 'escher_refresh_token' });
```

---

#### `clear_all_tokens`

Delete all Escher tokens from the OS keychain.

**Rust Signature**:
```rust
#[tauri::command]
async fn clear_all_tokens() -> Result<u32, String>
```

**TypeScript Usage**:
```typescript
// Complete logout - removes all tokens
const deletedCount = await invoke<number>('clear_all_tokens');

console.log(`Deleted ${deletedCount} tokens`);
```

**Returns**: Number of tokens deleted

---

### 2. Token Validation

#### `validate_token`

Validate a JWT token (check signature, expiry, etc.).

**Rust Signature**:
```rust
#[tauri::command]
async fn validate_token(token: String) -> Result<TokenValidation, String>
```

**TypeScript Usage**:
```typescript
interface TokenValidation {
  valid: boolean;
  expired: boolean;
  expiresAt?: string;
  claims?: Record<string, any>;
  error?: string;
}

const accessToken = await invoke<string | null>('get_token', {
  key: 'escher_access_token'
});

if (accessToken) {
  const validation = await invoke<TokenValidation>('validate_token', {
    token: accessToken
  });

  if (!validation.valid) {
    if (validation.expired) {
      // Refresh the token
      await refreshAccessToken();
    } else {
      // Invalid token, logout
      await logout();
    }
  }
}
```

---

#### `decode_token`

Decode JWT token payload without validation (for reading claims).

**Rust Signature**:
```rust
#[tauri::command]
async fn decode_token(token: String) -> Result<TokenPayload, String>
```

**TypeScript Usage**:
```typescript
interface TokenPayload {
  sub: string;           // User ID
  email?: string;
  name?: string;
  iat: number;           // Issued at (Unix timestamp)
  exp: number;           // Expiry (Unix timestamp)
  custom_claims?: Record<string, any>;
}

const idToken = await invoke<string | null>('get_token', {
  key: 'escher_id_token'
});

if (idToken) {
  const payload = await invoke<TokenPayload>('decode_token', {
    token: idToken
  });

  console.log(`Logged in as: ${payload.email}`);
  console.log(`Token expires: ${new Date(payload.exp * 1000).toLocaleString()}`);
}
```

---

### 3. Session Management

#### `create_session`

Create a new session after successful login.

**Rust Signature**:
```rust
#[tauri::command]
async fn create_session(tokens: SessionTokens) -> Result<Session, String>
```

**TypeScript Usage**:
```typescript
interface SessionTokens {
  accessToken: string;
  idToken: string;
  refreshToken: string;
  expiresIn: number;  // seconds
}

interface Session {
  sessionId: string;
  userId: string;
  email: string;
  createdAt: string;
  expiresAt: string;
  lastActivity: string;
}

// After successful Cognito login
const session = await invoke<Session>('create_session', {
  tokens: {
    accessToken: cognitoResponse.accessToken,
    idToken: cognitoResponse.idToken,
    refreshToken: cognitoResponse.refreshToken,
    expiresIn: 3600
  }
});

console.log(`Session created for ${session.email}`);
```

**What it does**:
1. Stores tokens in OS keychain
2. Creates session record in local storage
3. Sets up session timers (absolute + idle timeout)
4. Returns session info

---

#### `get_current_session`

Get the current active session.

**Rust Signature**:
```rust
#[tauri::command]
async fn get_current_session() -> Result<Option<Session>, String>
```

**TypeScript Usage**:
```typescript
const session = await invoke<Session | null>('get_current_session');

if (session) {
  console.log(`Logged in as: ${session.email}`);
  console.log(`Session expires: ${session.expiresAt}`);

  // Check if session is about to expire
  const expiresIn = new Date(session.expiresAt).getTime() - Date.now();
  if (expiresIn < 5 * 60 * 1000) {  // Less than 5 minutes
    console.log('Session expiring soon, should refresh');
  }
} else {
  console.log('No active session');
  redirectToLogin();
}
```

**Returns**: Session object or `null` if no active session

---

#### `update_session_activity`

Update last activity timestamp (for idle timeout tracking).

**Rust Signature**:
```rust
#[tauri::command]
async fn update_session_activity() -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
// Call this on user interactions
document.addEventListener('click', async () => {
  await invoke('update_session_activity');
});

document.addEventListener('keypress', async () => {
  await invoke('update_session_activity');
});
```

**Note**: This extends the idle timeout. Call frequently during user activity.

---

#### `invalidate_session`

Invalidate the current session (logout).

**Rust Signature**:
```rust
#[tauri::command]
async fn invalidate_session() -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
// Logout
const success = await invoke<boolean>('invalidate_session');

if (success) {
  // Clear UI state
  clearUserState();

  // Redirect to login
  redirectToLogin();
}
```

**What it does**:
1. Deletes all tokens from OS keychain
2. Clears session record
3. Cancels session timers
4. Emits `session_expired` event

---

### 4. Token Refresh

#### `refresh_access_token`

Refresh the access token using the refresh token.

**Rust Signature**:
```rust
#[tauri::command]
async fn refresh_access_token() -> Result<TokenRefreshResult, String>
```

**TypeScript Usage**:
```typescript
interface TokenRefreshResult {
  accessToken: string;
  idToken: string;
  expiresIn: number;
  refreshedAt: string;
}

try {
  const result = await invoke<TokenRefreshResult>('refresh_access_token');

  console.log('Tokens refreshed successfully');
  console.log(`New access token expires in ${result.expiresIn} seconds`);

  // Tokens are automatically stored in keychain

} catch (error) {
  console.error('Token refresh failed:', error);

  // Refresh token expired or invalid, user must re-login
  await logout();
  redirectToLogin();
}
```

**What it does**:
1. Retrieves refresh token from keychain
2. Calls Cognito refresh endpoint
3. Stores new access/ID tokens in keychain
4. Updates session expiry
5. Returns new tokens

**Errors**:
- `"Refresh token not found"` - No refresh token in keychain
- `"Refresh token expired"` - Must re-login
- `"Cognito refresh failed: {error}"` - API error

---

### 5. Multi-Factor Authentication (MFA)

#### `setup_mfa`

Initiate MFA setup (TOTP).

**Rust Signature**:
```rust
#[tauri::command]
async fn setup_mfa() -> Result<MFASetup, String>
```

**TypeScript Usage**:
```typescript
interface MFASetup {
  secretCode: string;
  qrCodeUrl: string;
  backupCodes: string[];
}

const mfaSetup = await invoke<MFASetup>('setup_mfa');

// Display QR code for user to scan with authenticator app
displayQRCode(mfaSetup.qrCodeUrl);

// Show backup codes for user to save
displayBackupCodes(mfaSetup.backupCodes);

// Wait for user to verify
// Then call verify_mfa_setup
```

---

#### `verify_mfa_setup`

Verify MFA setup with a code from authenticator app.

**Rust Signature**:
```rust
#[tauri::command]
async fn verify_mfa_setup(code: String) -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const code = getUserInputCode();

try {
  const success = await invoke<boolean>('verify_mfa_setup', { code });

  if (success) {
    showSuccess('MFA enabled successfully');
  }
} catch (error) {
  showError('Invalid code. Please try again.');
}
```

---

#### `verify_mfa_code`

Verify MFA code during login.

**Rust Signature**:
```rust
#[tauri::command]
async fn verify_mfa_code(code: String) -> Result<TokenRefreshResult, String>
```

**TypeScript Usage**:
```typescript
// After username/password authentication, if MFA is enabled
const code = getUserInputCode();

try {
  const tokens = await invoke<TokenRefreshResult>('verify_mfa_code', { code });

  // MFA verified, complete login
  await createSession(tokens);
  redirectToDashboard();

} catch (error) {
  showError('Invalid MFA code');
}
```

---

#### `disable_mfa`

Disable MFA for the current user.

**Rust Signature**:
```rust
#[tauri::command]
async fn disable_mfa() -> Result<bool, String>
```

**TypeScript Usage**:
```typescript
const success = await invoke<boolean>('disable_mfa');

if (success) {
  showSuccess('MFA disabled');
}
```

---

## TypeScript Service Wrapper

**File**: `src/services/tauri/AuthService.ts`

```typescript
import { invoke } from '@tauri-apps/api/tauri';
import { listen } from '@tauri-apps/api/event';

export class AuthService {
  // Token Storage
  async storeToken(key: string, value: string): Promise<boolean> {
    return invoke('store_token', { key, value });
  }

  async getToken(key: string): Promise<string | null> {
    return invoke('get_token', { key });
  }

  async deleteToken(key: string): Promise<boolean> {
    return invoke('delete_token', { key });
  }

  async clearAllTokens(): Promise<number> {
    return invoke('clear_all_tokens');
  }

  // Token Validation
  async validateToken(token: string): Promise<TokenValidation> {
    return invoke('validate_token', { token });
  }

  async decodeToken(token: string): Promise<TokenPayload> {
    return invoke('decode_token', { token });
  }

  // Session Management
  async createSession(tokens: SessionTokens): Promise<Session> {
    return invoke('create_session', { tokens });
  }

  async getCurrentSession(): Promise<Session | null> {
    return invoke('get_current_session');
  }

  async updateSessionActivity(): Promise<boolean> {
    return invoke('update_session_activity');
  }

  async invalidateSession(): Promise<boolean> {
    return invoke('invalidate_session');
  }

  // Token Refresh
  async refreshAccessToken(): Promise<TokenRefreshResult> {
    return invoke('refresh_access_token');
  }

  // MFA
  async setupMFA(): Promise<MFASetup> {
    return invoke('setup_mfa');
  }

  async verifyMFASetup(code: string): Promise<boolean> {
    return invoke('verify_mfa_setup', { code });
  }

  async verifyMFACode(code: string): Promise<TokenRefreshResult> {
    return invoke('verify_mfa_code', { code });
  }

  async disableMFA(): Promise<boolean> {
    return invoke('disable_mfa');
  }

  // Convenience methods
  async isAuthenticated(): Promise<boolean> {
    const session = await this.getCurrentSession();
    return session !== null;
  }

  async getUserInfo(): Promise<TokenPayload | null> {
    const idToken = await this.getToken('escher_id_token');
    if (!idToken) return null;
    return this.decodeToken(idToken);
  }

  async logout(): Promise<void> {
    await this.invalidateSession();
  }

  // Setup activity tracking
  setupActivityTracking(): void {
    const events = ['click', 'keypress', 'mousemove', 'scroll'];
    let lastActivity = Date.now();

    events.forEach(event => {
      document.addEventListener(event, () => {
        const now = Date.now();
        // Only update if more than 30 seconds since last update
        if (now - lastActivity > 30000) {
          this.updateSessionActivity();
          lastActivity = now;
        }
      });
    });
  }

  // Listen for session expiry
  async listenForSessionExpiry(callback: () => void): Promise<void> {
    await listen('session_expired', () => {
      callback();
    });
  }
}

export const authService = new AuthService();
```

---

## Usage in Authentication Flow

**Example**: Complete login flow with session management

```typescript
import { authService } from '@/services/tauri/AuthService';
import { Auth } from '@aws-amplify/auth';

export class AuthController {
  async login(username: string, password: string): Promise<void> {
    try {
      // 1. Authenticate with Cognito
      const cognitoUser = await Auth.signIn(username, password);

      // 2. Check if MFA is required
      if (cognitoUser.challengeName === 'SOFTWARE_TOKEN_MFA') {
        // Prompt user for MFA code
        const code = await this.promptForMFACode();
        const tokens = await authService.verifyMFACode(code);
        await this.completeLogin(tokens);
        return;
      }

      // 3. Get tokens from Cognito
      const session = await Auth.currentSession();
      const tokens: SessionTokens = {
        accessToken: session.getAccessToken().getJwtToken(),
        idToken: session.getIdToken().getJwtToken(),
        refreshToken: session.getRefreshToken().getToken(),
        expiresIn: 3600
      };

      // 4. Create session (stores tokens in keychain)
      await this.completeLogin(tokens);

    } catch (error) {
      console.error('Login failed:', error);
      throw new Error('Authentication failed');
    }
  }

  private async completeLogin(tokens: SessionTokens): Promise<void> {
    // Create session
    const session = await authService.createSession(tokens);

    // Setup activity tracking
    authService.setupActivityTracking();

    // Listen for session expiry
    await authService.listenForSessionExpiry(() => {
      this.handleSessionExpiry();
    });

    // Redirect to dashboard
    window.location.href = '/dashboard';
  }

  async restoreSession(): Promise<boolean> {
    try {
      // Check for existing session
      const session = await authService.getCurrentSession();

      if (!session) {
        return false;
      }

      // Validate access token
      const accessToken = await authService.getToken('escher_access_token');
      if (!accessToken) {
        return false;
      }

      const validation = await authService.validateToken(accessToken);

      if (validation.expired) {
        // Try to refresh
        try {
          await authService.refreshAccessToken();
          return true;
        } catch (error) {
          // Refresh failed, logout
          await authService.logout();
          return false;
        }
      }

      if (!validation.valid) {
        await authService.logout();
        return false;
      }

      // Session restored successfully
      authService.setupActivityTracking();
      return true;

    } catch (error) {
      console.error('Session restoration failed:', error);
      return false;
    }
  }

  async logout(): Promise<void> {
    // Invalidate session (clears keychain)
    await authService.logout();

    // Sign out from Cognito
    await Auth.signOut();

    // Clear UI state
    // ...

    // Redirect to login
    window.location.href = '/login';
  }

  private handleSessionExpiry(): void {
    console.log('Session expired');
    this.logout();
  }
}
```

---

## Error Handling

1. **Keychain Access Denied**:
```typescript
try {
  await authService.storeToken('escher_access_token', token);
} catch (error) {
  if (error.includes('access denied')) {
    showError('Please grant keychain access to Escher');
  }
}
```

2. **Token Expiry**:
```typescript
const validation = await authService.validateToken(token);
if (validation.expired) {
  await authService.refreshAccessToken();
}
```

3. **Refresh Token Expiry**:
```typescript
try {
  await authService.refreshAccessToken();
} catch (error) {
  if (error.includes('expired')) {
    // User must re-login
    await authService.logout();
    redirectToLogin();
  }
}
```

---

## Security Best Practices

1. **Never log tokens**: Don't console.log or display tokens in UI
2. **Use OS keychain**: Never store tokens in localStorage or files
3. **Validate tokens**: Always validate before using
4. **Refresh proactively**: Refresh 5 minutes before expiry
5. **Clear on logout**: Always clear all tokens on logout
6. **Activity tracking**: Update session activity to prevent idle timeout
7. **Listen for expiry**: Always listen to `session_expired` event

---

## Session Timeout Configuration

**Absolute Timeout**: 8 hours (session ends regardless of activity)
**Idle Timeout**: 30 minutes (session ends if no activity)

Configured in Rust backend, can be adjusted in config file.

---

## Next Steps

- [System Events →](./events-system.md)
- [Request Builder Commands →](./commands-request-builder.md)
- [Tauri Setup Guide →](./setup.md)