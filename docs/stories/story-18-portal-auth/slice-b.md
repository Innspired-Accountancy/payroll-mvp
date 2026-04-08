# Slice b: MFA Setup and Verification Flow

**Story:** story-18-portal-auth
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** slice-a (login page and BetterAuth integration)

---

## Goal

Implement mandatory MFA for all employee portal users using TOTP (Time-based One-Time Password). This slice covers both initial MFA setup (QR code scanning) and verification flow for subsequent logins.

---

## Decision Checklist

- [x] All libraries/packages named: BetterAuth 0.5.x (built-in MFA), qrcode 1.5.x, otplib 12.0.x
- [x] SDK methods/API calls identified: betterAuth.api.enableMFA(), betterAuth.api.verifyMFA(), betterAuth.api.getMFASetup()
- [x] External service endpoints: None (TOTP generated locally)
- [x] Data contracts defined: MFASetupResponse, MFAVerifyInput, TOTPBackupCodes
- [x] Configuration variables: MFA_ISSUER="Payroll Portal", MFA_DIGITS=6, MFA_STEP=30
- [x] Error scenarios identified: Invalid TOTP code, setup failure, backup code used
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-09-identity-access-spec.md — MFA requirements, TOTP support
- 02-05-employee-portal-spec.md — Security requirements, MFA mandatory

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(employee)/mfa/setup/page.tsx` | create | MFA setup page with QR code |
| `src/app/(employee)/mfa/verify/page.tsx` | create | MFA verification page |
| `src/components/auth/mfa-setup-form.tsx` | create | QR code display and verification |
| `src/components/auth/mfa-verify-form.tsx` | create | TOTP code input form |
| `src/server/routers/mfa.ts` | create | tRPC procedures for MFA operations |
| `src/lib/auth/mfa.ts` | create | MFA utility functions |

---

## Responsibilities

1. Generate TOTP secret and display as QR code for authenticator app scanning
2. Verify TOTP code during setup to confirm successful configuration
3. Generate and display backup codes for account recovery
4. Require TOTP verification on every login after initial setup
5. Validate TOTP codes with time drift tolerance (±1 step)
6. Store encrypted MFA secret in database

---

## Contracts

### mfa.getSetup
- **Method:** tRPC query `mfa.getSetup`
- **Input:** None (uses session context)
- **Output:**
  ```typescript
  {
    qrCodeUrl: string;        // otpauth:// URL for QR generation
    secret: string;           // Base32 encoded secret (display as backup)
    backupCodes: string[];    // 10 single-use backup codes
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — No valid session
  - `MFA_ALREADY_ENABLED` — MFA already configured
- **Auth:** Protected procedure with valid employee session

### mfa.verifySetup
- **Method:** tRPC mutation `mfa.verifySetup`
- **Input:**
  ```typescript
  {
    code: string;             // 6-digit TOTP code
  }
  ```
- **Output:**
  ```typescript
  {
    success: boolean;
    backupCodes: string[];    // Display once, then hashed storage
  }
  ```
- **Errors:**
  - `BAD_REQUEST` — Invalid code format
  - `UNAUTHORIZED` — Code verification failed
- **Auth:** Protected procedure

### mfa.verify
- **Method:** tRPC mutation `mfa.verify`
- **Input:**
  ```typescript
  {
    code: string;             // 6-digit TOTP or backup code
  }
  ```
- **Output:**
  ```typescript
  {
    success: boolean;
    sessionToken: string;     // Full session token post-MFA
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid code
  - `BACKUP_CODE_USED` — Backup code consumed (warning)
- **Auth:** Partial session (post-credential, pre-MFA)

---

## Business Rules & Invariants

1. MFA is mandatory for all employee portal access (no opt-out)
2. Backup codes are single-use and invalidated after consumption
3. TOTP codes are 6 digits, valid for 30 seconds
4. Time drift tolerance allows ±30 seconds (1 step before/after)
5. QR code displays issuer as "Payroll Portal" with employee email
6. Backup codes must be displayed once during setup, then hashed

---

## Edge Cases

1. **Employee loses authenticator device** — Use backup code, then re-setup MFA
2. **No backup codes remaining** — Contact employer admin for manual reset
3. **Clock drift on device** — Allow ±1 step tolerance, show warning if drift detected
4. **Backup code entered as TOTP** — Detect format (8 chars vs 6 digits), route correctly
5. **MFA setup abandoned mid-flow** — Require completion before portal access

---

## Tests

### mfa-setup-form.test.tsx
- QR code rendered as image from otpauth URL
- Verify button submits code to API
- Invalid code shows error message
- Valid code displays backup codes
- Copy backup codes button works

### mfa-verify-form.test.tsx
- 6-digit input with auto-focus
- Submit validates code format
- Valid TOTP grants full session
- Valid backup code grants session with warning
- Rate limiting on failed attempts

### mfa.router.test.ts
- getSetup generates valid TOTP secret
- verifySetup validates TOTP against secret
- verify accepts valid TOTP code
- verify accepts valid backup code and marks used
- verify rejects invalid/expired code

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/components/auth/mfa-*.test.tsx
npx vitest run src/server/routers/mfa.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Security → MFA required for all employees
- 02-09-identity-access-spec.md § Security → MFA required for all bureau staff (adapted for employees)
- 02-05-employee-portal-spec.md § Security → MFA required, session timeout
