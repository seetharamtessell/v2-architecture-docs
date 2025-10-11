# Authentication & Security

**Status**: Design Complete
**Authentication Provider**: AWS Cognito
**Security Model**: JWT tokens with secure storage

---

## Overview

The Escher frontend implements a secure authentication system using AWS Cognito with JWT token management, secure local storage, and session handling.

---

## Authentication Architecture

```
┌─────────────────────────────────────────────────────┐
│                    USER                             │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│              LoginView (Frontend)                    │
│  Username + Password                                │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│          AuthController (Business Logic)            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│      CognitoAuthService (AWS SDK Integration)       │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│              AWS Cognito (Cloud)                     │
│  User Pools • Identity Verification                 │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│            Tokens Returned                          │
│  • Access Token (JWT)                               │
│  • ID Token (JWT)                                   │
│  • Refresh Token                                    │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│         Secure Storage (Tauri)                      │
│  • Tokens stored in OS Keychain                     │
│  • Session data in Rust Storage Service             │
└─────────────────────────────────────────────────────┘
```

---

## AWS Cognito Integration

### User Pool Configuration

**User Pool Setup**:
- **Pool Name**: `escher-users-{environment}`
- **Sign-in Options**: Username, Email
- **MFA**: Optional (recommended for production)
- **Password Policy**:
  - Minimum length: 12 characters
  - Require uppercase, lowercase, numbers, special characters
  - Password expiry: 90 days (optional)

**App Client Configuration**:
- **App Client Name**: `escher-desktop-app`
- **Authentication Flows**:
  - `ALLOW_USER_PASSWORD_AUTH` (for username/password)
  - `ALLOW_REFRESH_TOKEN_AUTH` (for token refresh)
  - `ALLOW_USER_SRP_AUTH` (for Secure Remote Password)
- **Token Expiry**:
  - Access Token: 1 hour
  - ID Token: 1 hour
  - Refresh Token: 30 days

**User Attributes**:
- Email (verified)
- Name
- Organization/Account ID (custom attribute)
- Role (custom attribute: admin, user, viewer)

---

## Token Management

### Token Types

#### 1. Access Token (JWT)
**Purpose**: Authorize API calls to backend services

**Contains**:
```json
{
  "sub": "user-uuid",
  "cognito:groups": ["admin"],
  "token_use": "access",
  "scope": "aws.cognito.signin.user.admin",
  "auth_time": 1234567890,
  "iss": "https://cognito-idp.region.amazonaws.com/pool-id",
  "exp": 1234571490,
  "iat": 1234567890,
  "client_id": "app-client-id",
  "username": "john.doe"
}
```

**Usage**:
- Sent with every API request to server
- Included in WebSocket connection handshake
- Validated by backend before processing requests

**Lifetime**: 1 hour

---

#### 2. ID Token (JWT)
**Purpose**: User identity information

**Contains**:
```json
{
  "sub": "user-uuid",
  "email_verified": true,
  "email": "john.doe@example.com",
  "cognito:username": "john.doe",
  "name": "John Doe",
  "custom:organization": "acme-corp",
  "custom:role": "admin",
  "aud": "app-client-id",
  "token_use": "id",
  "auth_time": 1234567890,
  "iss": "https://cognito-idp.region.amazonaws.com/pool-id",
  "exp": 1234571490,
  "iat": 1234567890
}
```

**Usage**:
- Display user information in UI
- Role-based access control
- Profile management

**Lifetime**: 1 hour

---

#### 3. Refresh Token
**Purpose**: Obtain new access/ID tokens without re-authentication

**Contains**: Opaque string (not JWT)

**Usage**:
- Used when access token expires
- Allows silent token refresh
- Long-lived (30 days)

**Lifetime**: 30 days

---

## Secure Storage Architecture

### Storage Strategy

**Security Principle**: Never store sensitive data in browser localStorage/sessionStorage

### Storage Layers

#### Layer 1: OS Keychain (via Tauri)
**What**: JWT tokens (access, ID, refresh)
**Why**: OS-level encryption, secure against XSS
**How**: Tauri `tauri-plugin-keychain` or native OS keychain API

```
Storage Location:
  - macOS: Keychain Access
  - Windows: Windows Credential Manager
  - Linux: Secret Service API (GNOME Keyring, KWallet)

Key Names:
  - escher_access_token
  - escher_id_token
  - escher_refresh_token
```

**Security Benefits**:
- Encrypted at rest by OS
- Requires OS authentication to access
- Protected from web-based attacks
- Survives app restarts

---

#### Layer 2: Rust Storage Service (SQLite)
**What**: Session metadata, user preferences
**Why**: Persistent across app launches, queryable
**How**: SQLite database with encryption

```
Stored Data:
  - User ID
  - Username
  - Email
  - Organization
  - Role
  - Last login timestamp
  - Session ID
  - User preferences
```

**NOT Stored**:
- Passwords
- JWT tokens (those go in keychain)
- AWS credentials

---

#### Layer 3: Frontend Memory (Zustand)
**What**: Current session state (UI only)
**Why**: Fast access during app runtime
**How**: Zustand store, cleared on logout

```
AuthState:
  - isAuthenticated: boolean
  - user: UserProfile
  - sessionExpiry: Date
  - lastActivity: Date
```

**Cleared On**:
- User logout
- App restart
- Session timeout

---

## Authentication Flow

### 1. Login Flow

```
User enters credentials
    ↓
AuthController.login(username, password)
    ↓
CognitoAuthService.signIn()
    ↓
AWS Cognito authenticates
    ↓
Returns: {accessToken, idToken, refreshToken}
    ↓
AuthController receives tokens
    ↓
STEP 1: Store tokens in OS Keychain (via Tauri)
    invoke('store_token', {key: 'escher_access_token', value: accessToken})
    invoke('store_token', {key: 'escher_id_token', value: idToken})
    invoke('store_token', {key: 'escher_refresh_token', value: refreshToken})
    ↓
STEP 2: Decode ID token to get user profile
    const profile = decodeJWT(idToken)
    ↓
STEP 3: Store session metadata in Rust Storage Service
    invoke('save_session', {
      userId: profile.sub,
      username: profile.username,
      email: profile.email,
      role: profile.custom:role,
      sessionId: generateSessionId()
    })
    ↓
STEP 4: Update frontend state
    AuthState.setAuthenticated(true)
    AuthState.setUser(profile)
    AuthState.setSessionExpiry(tokenExpiry)
    ↓
STEP 5: Redirect to dashboard
    navigate('/dashboard')
```

---

### 2. Token Refresh Flow

```
API call fails with 401 Unauthorized
    ↓
AuthController.refreshTokens()
    ↓
STEP 1: Retrieve refresh token from keychain
    const refreshToken = await invoke('get_token', {key: 'escher_refresh_token'})
    ↓
STEP 2: Request new tokens from Cognito
    CognitoAuthService.refreshSession(refreshToken)
    ↓
AWS Cognito returns new tokens
    {accessToken, idToken} (new refresh token NOT returned)
    ↓
STEP 3: Store new tokens in keychain
    invoke('store_token', {key: 'escher_access_token', value: accessToken})
    invoke('store_token', {key: 'escher_id_token', value: idToken})
    ↓
STEP 4: Retry original API call
    Use new access token
    ↓
Success
```

**Automatic Refresh**:
- Check token expiry before each API call
- Refresh proactively if expiring within 5 minutes
- Prevents unnecessary 401 errors

---

### 3. Logout Flow

```
User clicks "Logout"
    ↓
AuthController.logout()
    ↓
STEP 1: Revoke tokens with Cognito (optional)
    CognitoAuthService.signOut()
    ↓
STEP 2: Delete tokens from keychain
    invoke('delete_token', {key: 'escher_access_token'})
    invoke('delete_token', {key: 'escher_id_token'})
    invoke('delete_token', {key: 'escher_refresh_token'})
    ↓
STEP 3: Clear session from Rust Storage Service
    invoke('clear_session')
    ↓
STEP 4: Clear frontend state
    AuthState.reset()
    ChatUIState.clearMessages()
    EstateUIState.reset()
    ↓
STEP 5: Redirect to login
    navigate('/login')
```

---

### 4. Session Restoration (App Restart)

```
App starts
    ↓
AuthController.restoreSession()
    ↓
STEP 1: Check for tokens in keychain
    const accessToken = await invoke('get_token', {key: 'escher_access_token'})
    ↓
If no tokens → show login screen
    ↓
STEP 2: Validate token expiry
    const expiry = decodeJWT(accessToken).exp
    if (expiry < Date.now()) {
        → Token expired, try refresh
    }
    ↓
STEP 3: Try refresh if expired
    AuthController.refreshTokens()
    ↓
If refresh fails → show login screen
    ↓
STEP 4: Restore session metadata from Rust Storage
    const session = await invoke('get_session')
    ↓
STEP 5: Update frontend state
    AuthState.setAuthenticated(true)
    AuthState.setUser(session.user)
    ↓
STEP 6: Navigate to dashboard
    navigate('/dashboard')
```

---

## Session Management

### Session Timeout Strategies

#### 1. Absolute Timeout
**Definition**: Session expires after fixed duration regardless of activity

**Configuration**:
- Duration: 8 hours (configurable)
- Behavior: User must re-login after 8 hours

**Implementation**:
```
On Login:
  sessionExpiry = currentTime + 8 hours
  store in AuthState

Check on every interaction:
  if (currentTime > sessionExpiry) {
    logout()
    redirect to login with message: "Session expired"
  }
```

---

#### 2. Idle Timeout
**Definition**: Session expires after period of inactivity

**Configuration**:
- Inactivity Duration: 30 minutes (configurable)
- Behavior: User must re-login if idle for 30 minutes

**Implementation**:
```
Track last activity:
  AuthState.lastActivity = Date.now()

Update on:
  - Mouse move
  - Keyboard input
  - API calls
  - Page navigation

Check periodically (every 1 minute):
  const idleTime = currentTime - lastActivity
  if (idleTime > 30 minutes) {
    logout()
    redirect to login with message: "Session timed out due to inactivity"
  }
```

---

#### 3. Token Expiry
**Definition**: Access token expires after 1 hour

**Configuration**:
- Access Token TTL: 1 hour
- Refresh Token TTL: 30 days

**Implementation**:
```
Before each API call:
  const expiry = decodeJWT(accessToken).exp
  if (expiry - currentTime < 5 minutes) {
    await refreshTokens()
  }
```

---

### Session State Persistence

**On App Close**:
- Tokens remain in keychain (secure)
- Session metadata remains in Rust Storage
- Frontend state cleared (memory)

**On App Reopen**:
- Attempt to restore session from keychain
- Validate token expiry
- Refresh if needed
- Restore user to previous state

---

## Security Best Practices

### 1. Token Storage

**DO**:
✅ Store tokens in OS keychain (Tauri)
✅ Use Rust backend for sensitive operations
✅ Clear tokens on logout
✅ Validate token expiry before use

**DON'T**:
❌ Store tokens in localStorage
❌ Store tokens in sessionStorage
❌ Store tokens in cookies (for desktop app)
❌ Log tokens to console
❌ Send tokens in URL parameters

---

### 2. Token Transmission

**DO**:
✅ Send access token in `Authorization` header
✅ Use HTTPS for all API calls
✅ Use WSS (secure WebSocket)
✅ Validate token on backend for every request

**Format**:
```
Authorization: Bearer <access_token>
```

**WebSocket Connection**:
```
ws://server/chat?token=<access_token>

OR

Send token in first message after connection:
{
  type: 'auth',
  token: <access_token>
}
```

---

### 3. Token Validation

**Frontend Validation** (before API call):
- Check token exists
- Check token not expired
- Refresh if expiring soon

**Backend Validation** (on every request):
- Verify JWT signature
- Check token not expired
- Verify token issuer (Cognito)
- Check token audience (app client ID)
- Verify user permissions

---

### 4. XSS Protection

**Tauri Advantage**:
- Desktop app, not web browser
- No direct DOM manipulation from untrusted sources
- Content Security Policy enforced by Tauri

**Additional Measures**:
- Sanitize all user inputs
- Escape HTML in markdown rendering
- Validate data from server before rendering

---

### 5. CSRF Protection

**Not Required**:
- Desktop app, not web browser
- No cookie-based authentication
- All requests include explicit authorization header

---

## Tauri Integration

### Tauri Commands for Auth

```rust
// src-tauri/src/commands/auth.rs

#[tauri::command]
pub async fn store_token(key: String, value: String) -> Result<(), String> {
    // Store in OS keychain using keyring crate
    let entry = keyring::Entry::new("escher", &key)?;
    entry.set_password(&value)?;
    Ok(())
}

#[tauri::command]
pub async fn get_token(key: String) -> Result<String, String> {
    // Retrieve from OS keychain
    let entry = keyring::Entry::new("escher", &key)?;
    let token = entry.get_password()?;
    Ok(token)
}

#[tauri::command]
pub async fn delete_token(key: String) -> Result<(), String> {
    // Delete from OS keychain
    let entry = keyring::Entry::new("escher", &key)?;
    entry.delete_password()?;
    Ok(())
}

#[tauri::command]
pub async fn save_session(session: SessionMetadata) -> Result<(), String> {
    // Save to Rust Storage Service (SQLite)
    let storage = get_storage_service();
    storage.save_session(session)?;
    Ok(())
}

#[tauri::command]
pub async fn get_session() -> Result<SessionMetadata, String> {
    // Retrieve from Rust Storage Service
    let storage = get_storage_service();
    let session = storage.get_session()?;
    Ok(session)
}

#[tauri::command]
pub async fn clear_session() -> Result<(), String> {
    // Clear from Rust Storage Service
    let storage = get_storage_service();
    storage.clear_session()?;
    Ok(())
}
```

---

### Frontend Service Usage

```typescript
// services/tauri/AuthStorageService.ts

import { invoke } from '@tauri-apps/api/tauri';

export class AuthStorageService {

  async storeToken(key: string, value: string): Promise<void> {
    await invoke('store_token', { key, value });
  }

  async getToken(key: string): Promise<string> {
    return await invoke('get_token', { key });
  }

  async deleteToken(key: string): Promise<void> {
    await invoke('delete_token', { key });
  }

  async storeTokens(tokens: {
    accessToken: string;
    idToken: string;
    refreshToken: string;
  }): Promise<void> {
    await Promise.all([
      this.storeToken('escher_access_token', tokens.accessToken),
      this.storeToken('escher_id_token', tokens.idToken),
      this.storeToken('escher_refresh_token', tokens.refreshToken),
    ]);
  }

  async getTokens(): Promise<{
    accessToken: string;
    idToken: string;
    refreshToken: string;
  }> {
    const [accessToken, idToken, refreshToken] = await Promise.all([
      this.getToken('escher_access_token'),
      this.getToken('escher_id_token'),
      this.getToken('escher_refresh_token'),
    ]);

    return { accessToken, idToken, refreshToken };
  }

  async clearTokens(): Promise<void> {
    await Promise.all([
      this.deleteToken('escher_access_token'),
      this.deleteToken('escher_id_token'),
      this.deleteToken('escher_refresh_token'),
    ]);
  }
}
```

---

## AuthController Design

```typescript
// controllers/AuthController.ts

export class AuthController {
  private cognitoService: CognitoAuthService;
  private storageService: AuthStorageService;

  async login(username: string, password: string): Promise<void> {
    // 1. Authenticate with Cognito
    const tokens = await this.cognitoService.signIn(username, password);

    // 2. Store tokens securely
    await this.storageService.storeTokens(tokens);

    // 3. Decode user profile
    const profile = this.decodeIDToken(tokens.idToken);

    // 4. Save session metadata
    await invoke('save_session', {
      userId: profile.sub,
      username: profile.username,
      email: profile.email,
      role: profile['custom:role'],
      sessionId: this.generateSessionId(),
      loginTime: Date.now(),
    });

    // 5. Update frontend state
    const authState = useAuthState.getState();
    authState.setAuthenticated(true);
    authState.setUser(profile);
    authState.setSessionExpiry(Date.now() + 8 * 60 * 60 * 1000); // 8 hours
    authState.setLastActivity(Date.now());
  }

  async refreshTokens(): Promise<void> {
    try {
      // 1. Get refresh token
      const refreshToken = await this.storageService.getToken('escher_refresh_token');

      // 2. Request new tokens
      const tokens = await this.cognitoService.refreshSession(refreshToken);

      // 3. Store new tokens
      await this.storageService.storeToken('escher_access_token', tokens.accessToken);
      await this.storageService.storeToken('escher_id_token', tokens.idToken);

      // 4. Update state
      const authState = useAuthState.getState();
      authState.setLastActivity(Date.now());

    } catch (error) {
      // Refresh failed, logout user
      await this.logout();
      throw new Error('Session expired, please login again');
    }
  }

  async logout(): Promise<void> {
    // 1. Revoke with Cognito (optional)
    try {
      await this.cognitoService.signOut();
    } catch (error) {
      // Continue with local logout even if Cognito fails
    }

    // 2. Clear tokens
    await this.storageService.clearTokens();

    // 3. Clear session
    await invoke('clear_session');

    // 4. Clear all frontend state
    useAuthState.getState().reset();
    useChatUIState.getState().clearMessages();
    useEstateUIState.getState().reset();

    // 5. Navigate to login
    window.location.href = '/login';
  }

  async restoreSession(): Promise<boolean> {
    try {
      // 1. Check for tokens
      const tokens = await this.storageService.getTokens();

      // 2. Validate expiry
      const expiry = this.decodeJWT(tokens.accessToken).exp;
      if (expiry * 1000 < Date.now()) {
        // Expired, try refresh
        await this.refreshTokens();
      }

      // 3. Restore session metadata
      const session = await invoke('get_session');

      // 4. Update state
      const authState = useAuthState.getState();
      authState.setAuthenticated(true);
      authState.setUser(session.user);

      return true;

    } catch (error) {
      // Session restoration failed
      return false;
    }
  }

  private decodeIDToken(idToken: string): UserProfile {
    const payload = JSON.parse(atob(idToken.split('.')[1]));
    return {
      userId: payload.sub,
      username: payload['cognito:username'],
      email: payload.email,
      name: payload.name,
      role: payload['custom:role'],
      organization: payload['custom:organization'],
    };
  }

  private decodeJWT(token: string): any {
    return JSON.parse(atob(token.split('.')[1]));
  }

  private generateSessionId(): string {
    return `session_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

---

## AuthState (Zustand Store)

```typescript
// models/state/AuthState.ts

interface AuthState {
  // State
  isAuthenticated: boolean;
  user: UserProfile | null;
  sessionExpiry: number | null;
  lastActivity: number;

  // Actions
  setAuthenticated: (value: boolean) => void;
  setUser: (user: UserProfile) => void;
  setSessionExpiry: (expiry: number) => void;
  setLastActivity: (time: number) => void;
  reset: () => void;

  // Computed
  isSessionExpired: () => boolean;
  isIdle: () => boolean;
}

export const useAuthState = create<AuthState>((set, get) => ({
  isAuthenticated: false,
  user: null,
  sessionExpiry: null,
  lastActivity: Date.now(),

  setAuthenticated: (value) => set({ isAuthenticated: value }),
  setUser: (user) => set({ user }),
  setSessionExpiry: (expiry) => set({ sessionExpiry: expiry }),
  setLastActivity: (time) => set({ lastActivity: time }),

  reset: () => set({
    isAuthenticated: false,
    user: null,
    sessionExpiry: null,
    lastActivity: Date.now(),
  }),

  isSessionExpired: () => {
    const { sessionExpiry } = get();
    if (!sessionExpiry) return false;
    return Date.now() > sessionExpiry;
  },

  isIdle: () => {
    const { lastActivity } = get();
    const idleTime = Date.now() - lastActivity;
    return idleTime > 30 * 60 * 1000; // 30 minutes
  },
}));
```

---

## Session Monitoring

### Periodic Session Checks

```typescript
// Setup session monitoring
export function setupSessionMonitoring() {

  // Check every minute
  setInterval(() => {
    const authState = useAuthState.getState();

    if (!authState.isAuthenticated) return;

    // Check absolute timeout
    if (authState.isSessionExpired()) {
      authController.logout();
      showMessage('Session expired. Please login again.');
      return;
    }

    // Check idle timeout
    if (authState.isIdle()) {
      authController.logout();
      showMessage('Session timed out due to inactivity.');
      return;
    }

  }, 60 * 1000); // Every 1 minute

  // Track activity
  const updateActivity = () => {
    useAuthState.getState().setLastActivity(Date.now());
  };

  window.addEventListener('mousemove', updateActivity);
  window.addEventListener('keydown', updateActivity);
  window.addEventListener('click', updateActivity);
}
```

---

## Security Checklist

### Implementation Checklist

- [ ] Tokens stored in OS keychain (not localStorage)
- [ ] Session metadata in Rust Storage Service
- [ ] Automatic token refresh before expiry
- [ ] Session timeout implemented (absolute + idle)
- [ ] Logout clears all tokens and state
- [ ] Session restoration on app restart
- [ ] Activity tracking for idle timeout
- [ ] Authorization header on all API calls
- [ ] Token validation before each request
- [ ] Secure WebSocket connection (WSS + token)
- [ ] Error handling for expired/invalid tokens
- [ ] User feedback for session expiry
- [ ] XSS protection (input sanitization)
- [ ] No sensitive data logged

---

## Summary

**Authentication**: AWS Cognito with JWT tokens
**Token Storage**: OS Keychain via Tauri (secure)
**Session Storage**: Rust Storage Service (metadata only)
**Session Timeout**: 8 hours absolute, 30 minutes idle
**Token Refresh**: Automatic before expiry
**Security**: OS-level encryption, no localStorage, secure transmission

This architecture ensures secure, seamless authentication with proper token management and session handling.