# Slice a: TOTP Generation and Encryption

**Story:** story-04-mfa
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** story-03-jwt-implementation

## Goal

Implement TOTP (Time-based One-Time Password) secret generation using RFC 6238/4226, with secure encryption for storage and time-window validation for verification.

## Decision Checklist

- [x] All libraries/packages named: `otpauth@9.x`, `hi-base32@latest`, `crypto` (Node.js built-in)
- [x] SDK methods identified: `new OTPAuth.Secret()`, `new OTPAuth.TOTP()`, `crypto.createCipheriv()`, `crypto.createDecipheriv()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: TOTP secret encryption/decryption, verification result
- [x] Configuration variables: `MFA_ENCRYPTION_KEY` (32-byte hex for AES-256)
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → MFA required for bureau staff
- 02-09-identity-access-spec.md:§ User Journeys → Bureau User Login (step 4-5)

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/mfa/totp.ts` | create | TOTP generation and verification |
| `src/lib/mfa/encryption.ts` | create | AES-256 encryption for secrets |
| `src/lib/mfa/constants.ts` | create | MFA configuration constants |
| `src/lib/db/schema/users.ts` | update | Add mfa_secret field (if not in story-01) |

## Responsibilities
1. Generate cryptographically secure TOTP secrets (20 bytes)
2. Encrypt secrets with AES-256-GCM before storage
3. Decrypt secrets for verification
4. Validate TOTP codes with time-window tolerance
5. Support multiple authenticator apps (Google, Authy, Microsoft)

## Contracts

### TOTP Service
```typescript
// src/lib/mfa/totp.ts
import * as OTPAuth from 'otpauth';
import { encrypt, decrypt } from './encryption';

export interface TOTPSetup {
  secret: string;        // Base32 encoded secret (for QR code)
  uri: string;           // otpauth:// URI for QR code
  encryptedSecret: string; // For database storage
}

export interface TOTPVerification {
  valid: boolean;
  delta?: number;        // Time steps difference (for resync)
}

export function generateTOTPSecret(userId: string, email: string): TOTPSetup {
  // Generate cryptographically secure secret
  const secret = new OTPAuth.Secret({ size: 20 });
  
  // Create TOTP object
  const totp = new OTPAuth.TOTP({
    issuer: 'UK Bureau Payroll',
    label: email,
    algorithm: 'SHA1',
    digits: 6,
    period: 30,
    secret: secret,
  });
  
  // Encrypt secret for storage
  const secretBase32 = secret.base32;
  const encryptedSecret = encrypt(secretBase32);
  
  return {
    secret: secretBase32,
    uri: totp.toString(),
    encryptedSecret,
  };
}

export function verifyTOTP(encryptedSecret: string, code: string): TOTPVerification {
  try {
    // Decrypt the secret
    const secretBase32 = decrypt(encryptedSecret);
    
    // Create TOTP object
    const totp = new OTPAuth.TOTP({
      secret: OTPAuth.Secret.fromBase32(secretBase32),
      algorithm: 'SHA1',
      digits: 6,
      period: 30,
    });
    
    // Verify with window of +/- 1 step (30 seconds before/after)
    const delta = totp.validate({ token: code, window: 1 });
    
    if (delta === null) {
      return { valid: false };
    }
    
    return { valid: true, delta };
    
  } catch (error) {
    return { valid: false };
  }
}

export function generateTOTPCode(encryptedSecret: string): string {
  const secretBase32 = decrypt(encryptedSecret);
  
  const totp = new OTPAuth.TOTP({
    secret: OTPAuth.Secret.fromBase32(secretBase32),
    algorithm: 'SHA1',
    digits: 6,
    period: 30,
  });
  
  return totp.generate();
}
```

### AES-256-GCM Encryption
```typescript
// src/lib/mfa/encryption.ts
import { createCipheriv, createDecipheriv, randomBytes, scryptSync } from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const IV_LENGTH = 16;
const AUTH_TAG_LENGTH = 16;
const SALT_LENGTH = 32;

// Derive key from environment variable (should be 32 bytes hex)
function getKey(): Buffer {
  const keyHex = process.env.MFA_ENCRYPTION_KEY!;
  if (!keyHex || keyHex.length !== 64) {
    throw new Error('MFA_ENCRYPTION_KEY must be 64 hex characters (32 bytes)');
  }
  return Buffer.from(keyHex, 'hex');
}

/**
 * Encrypt a TOTP secret
 * Format: salt(32) + iv(16) + ciphertext + authTag(16)
 */
export function encrypt(plaintext: string): string {
  const key = getKey();
  const salt = randomBytes(SALT_LENGTH);
  const iv = randomBytes(IV_LENGTH);
  
  const cipher = createCipheriv(ALGORITHM, key, iv);
  
  const encrypted = Buffer.concat([
    cipher.update(plaintext, 'utf8'),
    cipher.final(),
  ]);
  
  const authTag = cipher.getAuthTag();
  
  // Combine: salt + iv + encrypted + authTag
  const result = Buffer.concat([salt, iv, encrypted, authTag]);
  
  return result.toString('base64');
}

/**
 * Decrypt a TOTP secret
 */
export function decrypt(encryptedData: string): string {
  const key = getKey();
  const data = Buffer.from(encryptedData, 'base64');
  
  // Extract components
  const salt = data.subarray(0, SALT_LENGTH);
  const iv = data.subarray(SALT_LENGTH, SALT_LENGTH + IV_LENGTH);
  const authTag = data.subarray(data.length - AUTH_TAG_LENGTH);
  const encrypted = data.subarray(SALT_LENGTH + IV_LENGTH, data.length - AUTH_TAG_LENGTH);
  
  const decipher = createDecipheriv(ALGORITHM, key, iv);
  decipher.setAuthTag(authTag);
  
  const decrypted = Buffer.concat([
    decipher.update(encrypted),
    decipher.final(),
  ]);
  
  return decrypted.toString('utf8');
}
```

### Configuration Constants
```typescript
// src/lib/mfa/constants.ts
export const MFA_CONFIG = {
  // TOTP settings
  TOTP_DIGITS: 6,
  TOTP_PERIOD: 30, // seconds
  TOTP_ALGORITHM: 'SHA1',
  TOTP_WINDOW: 1, // Accept codes from 1 step before/after current
  
  // Verification limits
  MAX_VERIFICATION_ATTEMPTS: 5,
  VERIFICATION_WINDOW_MINUTES: 15,
  
  // Setup requirements
  REQUIRE_MFA_FOR_BUREAU: true,
  REQUIRE_MFA_FOR_CLIENT: false, // Configurable per bureau
  
  // Backup codes
  BACKUP_CODES_COUNT: 10,
  BACKUP_CODE_LENGTH: 8,
} as const;
```

### Environment Variables
```bash
# .env.example
MFA_ENCRYPTION_KEY="abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890"
```

## Business Rules & Invariants
1. TOTP secrets are encrypted at rest with AES-256-GCM
2. TOTP codes are 6 digits, valid for 30 seconds
3. Accept codes from 1 time step before/after (clock drift tolerance)
4. Secrets are 20 bytes (160 bits) of randomness
5. Only decrypt secrets in memory, never log or expose

## Edge Cases
1. **Encryption key missing/invalid** — Throw at startup, prevent app from running
2. **Corrupted encrypted data** — Return verification failure, don't crash
3. **Clock skew > 30 seconds** — Accept with delta tracking for monitoring
4. **Code reuse within same window** — Track used codes in short-term cache
5. **Concurrent verification attempts** — Idempotent within time window

## Tests

### src/lib/mfa/totp.test.ts
- `should generate valid TOTP secret`: Verifies secret generation
- `should encrypt and decrypt secret`: Verifies encryption roundtrip
- `should verify valid TOTP code`: Verifies code validation
- `should reject invalid TOTP code`: Verifies rejection
- `should accept code within time window`: Verifies window tolerance
- `should reject code outside window`: Verifies window enforcement
- `should handle clock skew`: Verifies drift tolerance

### src/lib/mfa/encryption.test.ts
- `should encrypt plaintext`: Verifies encryption
- `should decrypt ciphertext`: Verifies decryption
- `should produce different ciphertexts for same input`: Verifies IV randomness
- `should reject tampered ciphertext`: Verifies auth tag
- `should throw on missing key`: Verifies key validation

## Verification
```bash
# Generate encryption key for testing
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

npm run test:unit src/lib/mfa/totp.test.ts
npm run test:unit src/lib/mfa/encryption.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → MFA
