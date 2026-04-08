# Slice d: Backup Codes and Recovery

**Story:** story-04-mfa
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-c

## Goal

Implement backup code generation and verification for account recovery when users lose access to their authenticator app. Each backup code is single-use and regenerating codes invalidates old ones.

## Decision Checklist

- [x] All libraries/packages named: `crypto` (Node.js built-in), `drizzle-orm@latest`
- [x] SDK methods identified: `randomBytes()`, `timingSafeEqual()`, `db.update()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Backup code structure, verification result
- [x] Configuration variables: `BACKUP_CODES_COUNT=10`, `BACKUP_CODE_BYTES=5`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ User Journeys → Account recovery flows
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → MFA backup

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/mfa/backup-codes.ts` | create | Backup code generation and verification |
| `src/lib/mfa/encryption.ts` | update | Add backup code encryption helpers |
| `src/server/trpc/routers/mfa.ts` | update | Add backup code endpoints |

## Responsibilities
1. Generate cryptographically random backup codes
2. Hash backup codes before storage (not raw encryption)
3. Verify backup codes with constant-time comparison
4. Mark codes as used after verification
5. Regenerate codes on request (invalidating old ones)

## Contracts

### Backup Code Service
```typescript
// src/lib/mfa/backup-codes.ts
import { randomBytes, timingSafeEqual, createHash } from 'crypto';
import { eq } from 'drizzle-orm';
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';
import { encrypt, decrypt } from './encryption';

const BACKUP_CODES_COUNT = 10;
const BACKUP_CODE_BYTES = 5; // Results in ~8 alphanumeric chars

export interface BackupCode {
  code: string;        // The actual code shown to user (e.g., "A1B2C3D4")
  hash: string;        // SHA-256 hash for storage
  used: boolean;
}

export interface BackupCodesResult {
  codes: string[];     // Plain codes to show user (once only)
  encryptedCodes: string[]; // For database storage
}

export interface VerifyBackupCodeResult {
  valid: boolean;
  remainingCodes: number;
  regenerateRecommended: boolean;
}

/**
 * Generate a new set of backup codes
 */
export function generateBackupCodes(): BackupCodesResult {
  const codes: string[] = [];
  const encryptedCodes: string[] = [];
  
  for (let i = 0; i < BACKUP_CODES_COUNT; i++) {
    // Generate random code
    const code = generateBackupCode();
    codes.push(code);
    
    // Hash for storage (we encrypt the hash for extra protection)
    const hash = hashBackupCode(code);
    encryptedCodes.push(encrypt(hash));
  }
  
  return { codes, encryptedCodes };
}

function generateBackupCode(): string {
  // Generate random bytes and convert to alphanumeric
  const bytes = randomBytes(BACKUP_CODE_BYTES);
  const code = bytes
    .toString('base64')
    .replace(/[^a-zA-Z0-9]/g, '')
    .slice(0, 8)
    .toUpperCase();
  
  // Ensure we have 8 characters
  if (code.length < 8) {
    const padding = randomBytes(4)
      .toString('base64')
      .replace(/[^a-zA-Z0-9]/g, '')
      .slice(0, 8 - code.length)
      .toUpperCase();
    return code + padding;
  }
  
  return code;
}

function hashBackupCode(code: string): string {
  return createHash('sha256')
    .update(code.toUpperCase())
    .digest('hex');
}

/**
 * Verify a backup code for a user
 */
export async function verifyBackupCode(
  userId: string,
  inputCode: string
): Promise<VerifyBackupCodeResult> {
  // Normalize input
  const normalizedCode = inputCode.toUpperCase().replace(/[^A-Z0-9]/g, '');
  
  if (normalizedCode.length !== 8) {
    return {
      valid: false,
      remainingCodes: 0,
      regenerateRecommended: false,
    };
  }
  
  // Get user's backup codes
  const user = await db.query.users.findFirst({
    where: eq(users.id, userId),
    columns: { mfaBackupCodes: true },
  });
  
  if (!user?.mfaBackupCodes) {
    return {
      valid: false,
      remainingCodes: 0,
      regenerateRecommended: false,
    };
  }
  
  const inputHash = hashBackupCode(normalizedCode);
  const storedCodes: string[] = user.mfaBackupCodes;
  
  // Find matching code
  let matchIndex = -1;
  const decryptedCodes = storedCodes.map((encrypted, index) => {
    try {
      const hash = decrypt(encrypted);
      return { hash, index, used: hash.startsWith('USED:') };
    } catch {
      return { hash: '', index, used: true };
    }
  });
  
  for (const entry of decryptedCodes) {
    if (entry.used) continue;
    
    // Constant-time comparison to prevent timing attacks
    const expectedHash = Buffer.from(entry.hash, 'hex');
    const actualHash = Buffer.from(inputHash, 'hex');
    
    if (expectedHash.length === actualHash.length &&
        timingSafeEqual(expectedHash, actualHash)) {
      matchIndex = entry.index;
      break;
    }
  }
  
  if (matchIndex === -1) {
    // Count remaining valid codes
    const remaining = decryptedCodes.filter(c => !c.used).length;
    return {
      valid: false,
      remainingCodes: remaining,
      regenerateRecommended: remaining <= 2,
    };
  }
  
  // Mark code as used
  const updatedCodes = [...storedCodes];
  updatedCodes[matchIndex] = encrypt(`USED:${inputHash}`);
  
  await db.update(users)
    .set({ mfaBackupCodes: updatedCodes })
    .where(eq(users.id, userId));
  
  // Count remaining
  const remaining = decryptedCodes.filter((c, i) => !c.used && i !== matchIndex).length;
  
  // Log backup code use
  await logSecurityEvent({
    type: 'BACKUP_CODE_USED',
    userId,
    remainingCodes: remaining,
    timestamp: new Date(),
  });
  
  return {
    valid: true,
    remainingCodes: remaining,
    regenerateRecommended: remaining <= 2,
  };
}

/**
 * Regenerate backup codes (invalidates all old codes)
 */
export async function regenerateBackupCodes(userId: string): Promise<string[]> {
  const { codes, encryptedCodes } = generateBackupCodes();
  
  await db.update(users)
    .set({ mfaBackupCodes: encryptedCodes })
    .where(eq(users.id, userId));
  
  // Log regeneration
  await logSecurityEvent({
    type: 'BACKUP_CODES_REGENERATED',
    userId,
    timestamp: new Date(),
  });
  
  return codes;
}

/**
 * Get count of remaining unused backup codes
 */
export async function getRemainingBackupCodesCount(userId: string): Promise<number> {
  const user = await db.query.users.findFirst({
    where: eq(users.id, userId),
    columns: { mfaBackupCodes: true },
  });
  
  if (!user?.mfaBackupCodes) return 0;
  
  let count = 0;
  for (const encrypted of user.mfaBackupCodes) {
    try {
      const hash = decrypt(encrypted);
      if (!hash.startsWith('USED:')) {
        count++;
      }
    } catch {
      // Invalid entry, count as used
    }
  }
  
  return count;
}
```

### tRPC Backup Code Endpoints
```typescript
// Add to src/server/trpc/routers/mfa.ts

  // Get remaining backup codes count
  getBackupCodeStatus: authenticatedProcedure
    .query(async ({ ctx }) => {
      const count = await getRemainingBackupCodesCount(ctx.user!.sub);
      
      return {
        remainingCodes: count,
        regenerateRecommended: count <= 2,
        totalCodes: 10,
      };
    }),
  
  // Regenerate backup codes
  regenerateBackupCodes: authenticatedProcedure
    .input(z.object({
      verificationCode: z.string().length(6).regex(/^\d+$/),
    }))
    .mutation(async ({ input, ctx }) => {
      const userId = ctx.user!.sub;
      
      // Verify current MFA code before allowing regeneration
      const user = await db.query.users.findFirst({
        where: eq(users.id, userId),
        columns: { mfaSecret: true, mfaEnabled: true },
      });
      
      if (!user?.mfaEnabled || !user.mfaSecret) {
        throw new TRPCError({
          code: 'CONFLICT',
          message: 'MFA is not enabled',
        });
      }
      
      const verification = verifyTOTP(user.mfaSecret, input.verificationCode);
      if (!verification.valid) {
        throw new TRPCError({
          code: 'UNAUTHORIZED',
          message: 'Invalid verification code',
        });
      }
      
      // Regenerate codes
      const newCodes = await regenerateBackupCodes(userId);
      
      return {
        success: true,
        backupCodes: newCodes,
        message: 'Backup codes regenerated. Save these securely - they will not be shown again.',
      };
    }),
```

## Business Rules & Invariants
1. 10 backup codes generated per user on MFA setup
2. Each backup code is 8 alphanumeric characters
3. Backup codes are single-use only
4. Codes are hashed (SHA-256) and encrypted before storage
5. Used codes are marked with "USED:" prefix in encrypted storage
6. Regenerating codes invalidates all old codes

## Edge Cases
1. **All codes used** — User must contact admin or regenerate
2. **Regeneration without MFA** — Require TOTP verification first
3. **Case sensitivity** — Normalize to uppercase before verification
4. **Timing attack on code verification** — Use constant-time comparison
5. **Database corruption of codes** — Log and alert, mark as used

## Tests

### src/lib/mfa/backup-codes.test.ts
- `should generate 10 unique codes`: Verifies count
- `should verify valid backup code`: Verifies success
- `should mark code as used after verification`: Verifies consumption
- `should reject invalid code`: Verifies failure
- `should reject reused code`: Verifies single-use
- `should regenerate new codes`: Verifies regeneration
- `should invalidate old codes on regenerate`: Verifies invalidation
- `should use constant-time comparison`: Verifies timing safety

## Verification
```bash
npm run test:unit src/lib/mfa/backup-codes.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → MFA backup
