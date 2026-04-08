# Slice b: HMRC Authentication Handling

**Story:** story-03-hmrc-gateway
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement HMRC PAYE Online authentication using credentials (User ID, Password, Tax Office Number) with secure credential storage, automatic re-authentication, and session management.

---

## Decision Checklist

- [x] All libraries/packages named: Node.js crypto for credential encryption
- [x] SDK methods identified: GatewayClient.authenticate(), refreshSession()
- [x] External service endpoints: HMRC authentication endpoint
- [x] Data contracts defined: HMRCCredentials, AuthSession interfaces
- [x] Configuration: ENCRYPTION_KEY for credential storage
- [x] Error scenarios: Invalid credentials, expired session, auth service unavailable
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Integration Points → HMRC Gateway
- HMRC PAYE Online Technical Specification

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/gateway/auth.ts` | create | Authentication service |
| `src/lib/hmrc/gateway/credentials.ts` | create | Secure credential management |
| `src/lib/db/schema/hmrc-credentials.ts` | create | Credentials storage schema |

---

## Responsibilities

1. Store HMRC credentials encrypted at rest (AES-256-GCM)
2. Authenticate with HMRC Gateway
3. Manage authentication sessions with refresh
4. Handle authentication failures gracefully
5. Support multiple PAYE schemes per employer

---

## Contracts

### HMRCAuthService
- **Method:** `async authenticate(credentials: HMRCCredentials): Promise<AuthSession>`
- **Input:**
  ```typescript
  interface HMRCCredentials {
    userId: string; // HMRC User ID
    password: string; // HMRC Password
    taxOfficeNumber: string; // 3-digit tax office number
    taxOfficeReference: string; // Employer reference part
  }
  ```
- **Output:** AuthSession
  ```typescript
  interface AuthSession {
    sessionId: string;
    token: string; // Session token from HMRC
    expiresAt: Date;
    authenticatedAt: Date;
  }
  ```
- **Errors:**
  - `InvalidCredentialsError` — Authentication rejected
  - `AccountLockedError` — Too many failed attempts
  - `AuthServiceUnavailableError` — HMRC auth service down

### CredentialsStorage
- **Method:** `async store(employerId: string, credentials: HMRCCredentials): Promise<void>`
- **Encryption:** AES-256-GCM with key from ENCRYPTION_KEY env var
- **Storage:** PostgreSQL encrypted fields
- **Retrieval:** `async get(employerId: string): Promise<HMRCCredentials | null>`

### Authentication Flow
1. Retrieve encrypted credentials from database
2. Decrypt credentials using ENCRYPTION_KEY
3. POST to HMRC auth endpoint with credentials
4. Receive session token in response
5. Store session token (not credentials) in memory/cache
6. Use session token for subsequent submissions

### HMRC Auth Endpoint
```
POST /authenticate
Content-Type: application/x-www-form-urlencoded

userId={userId}&password={password}&taxOfficeNumber={taxOfficeNumber}
```

---

## Business Rules & Invariants

1. Credentials encrypted at rest with AES-256-GCM
2. Session tokens cached in Redis with 30-minute TTL
3. Automatic re-authentication when session expires
4. Failed auth attempts logged for security audit
5. Lock account after 5 consecutive failures
6. Never log credentials or session tokens

---

## Edge Cases

1. **Concurrent auth requests** — Single-flight pattern, one auth at a time
2. **Clock drift with HMRC** — Synchronize clocks, retry with adjusted time
3. **Password expiry warning** — Detect and notify payroll admin
4. **Tax office closure/merger** — Update credentials, maintain history

---

## Tests

### auth.test.ts
- Successful authentication with valid credentials
- Reject invalid credentials
- Session caching and reuse
- Automatic re-authentication on expiry
- Credential encryption/decryption
- Concurrent auth deduplication

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/gateway/auth.test.ts
npm run lint src/lib/hmrc/gateway/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § Integration Points → HMRC Gateway
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Security
