# Security & Privacy Architecture

**Status**: To be documented
**Purpose**: Security model, encryption, credential management, and privacy guarantees

---

## Planned Content

### Credential Security
- OS Keychain integration (macOS Keychain, Windows Credential Manager, Linux Secret Service)
- AWS credentials never sent to server
- Secure storage and access patterns

### Data Encryption
- Application-level encryption (AES-256-GCM)
- At-rest encryption for local Qdrant data
- Encryption key management

### Privacy Model
- Client-side AWS estate (never leaves device)
- Server receives only context, not credentials
- Chat history encryption and local storage

### Threat Model
- Attack vectors and mitigations
- Trust boundaries
- Security best practices

### Compliance
- Data residency requirements
- Audit logging
- Access control

---

This directory will contain comprehensive security and privacy documentation for the entire system.