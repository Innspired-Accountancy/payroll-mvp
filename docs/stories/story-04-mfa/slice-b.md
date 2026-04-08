# Slice b: MFA Setup and QR Code

**Story:** story-04-mfa
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-a

## Goal

Implement the MFA setup flow where users can enable MFA, scan a QR code with their authenticator app, and verify the setup with an initial TOTP code.

## Decision Checklist

- [x] All libraries/packages named: `qrcode@1.x`, `otpauth@9.x`
- [x] SDK methods identified: `QRCode.toDataURL()`, `QRCode.toString()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Setup state, verification request
- [x] Configuration variables: `MFA_SETUP_EXPIRY_MINUTES=10`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ User Journeys → Bureau User Login (MFA setup implied)
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → MFA required

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/mfa/setup.ts` | create | MFA setup orchestration |
| `src/server/trpc/routers/mfa.ts` | create | MFA tRPC procedures |
| `src/app/(auth)/mfa/setup/page.tsx` | create | MFA setup UI |
| `src/components/mfa/qr-code.tsx` | create | QR code display component |

## Responsibilities
1. Initiate MFA setup (generate encrypted secret, store temporarily)
2. Generate QR code PNG/SVG for authenticator app scanning
3. Verify initial code to confirm setup success
4. Store encrypted secret on successful verification
5. Enable mfa_enabled flag on user record

## Contracts

### MFA Setup Flow
```typescript
// src/lib/mfa/setup.ts
import { generateTOTPSecret, verifyTOTP } from './totp';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);
const SETUP_PREFIX = 'mfa:setup:';
const SETUP_EXPIRY = 600; // 10 minutes

export interface MFASetupSession {
  userId: string;
  encryptedSecret: string;
  secretUri: string;
  createdAt: string;
}

export interface MFASetupInitResult {
  setupToken: string;
  qrCodeDataUrl: string;
  secretUri: string;
  manualEntryCode: string;
}

/**
 * Initialize MFA setup for a user
 */
export async function initiateMFASetup(
  userId: string, 
  email: string
): Promise<MFASetupInitResult> {
  // Generate new TOTP secret
  const setup = generateTOTPSecret(userId, email);
  
  // Create setup session (temporary, not yet enabled)
  const setupSession: MFASetupSession = {
    userId,
    encryptedSecret: setup.encryptedSecret,
    secretUri: setup.uri,
    createdAt: new Date().toISOString(),
  };
  
  // Store in Redis with expiry
  const setupToken = crypto.randomUUID();
  await redis.setex(
    `${SETUP_PREFIX}${setupToken}`,
    SETUP_EXPIRY,
    JSON.stringify(setupSession)
  );
  
  // Generate QR code
  const qrCodeDataUrl = await generateQRCode(setup.uri);
  
  return {
    setupToken,
    qrCodeDataUrl,
    secretUri: setup.uri,
    manualEntryCode: setup.secret,
  };
}

/**
 * Verify setup code and enable MFA
 */
export async function completeMFASetup(
  setupToken: string,
  verificationCode: string
): Promise<{ success: boolean; backupCodes?: string[]; error?: string }> {
  // Retrieve setup session
  const sessionData = await redis.get(`${SETUP_PREFIX}${setupToken}`);
  if (!sessionData) {
    return { success: false, error: 'Setup session expired or invalid' };
  }
  
  const session: MFASetupSession = JSON.parse(sessionData);
  
  // Verify the code
  const verification = verifyTOTP(session.encryptedSecret, verificationCode);
  if (!verification.valid) {
    return { success: false, error: 'Invalid verification code' };
  }
  
  // Generate backup codes
  const backupCodes = generateBackupCodes();
  const encryptedBackupCodes = backupCodes.map(code => encrypt(code));
  
  // Update user record
  await db.update(users)
    .set({
      mfaEnabled: true,
      mfaSecret: session.encryptedSecret,
      mfaBackupCodes: encryptedBackupCodes, // JSON array
      updatedAt: new Date(),
    })
    .where(eq(users.id, session.userId));
  
  // Clean up setup session
  await redis.del(`${SETUP_PREFIX}${setupToken}`);
  
  // Log MFA enable event
  await logSecurityEvent({
    type: 'MFA_ENABLED',
    userId: session.userId,
    timestamp: new Date(),
  });
  
  return { success: true, backupCodes };
}

async function generateQRCode(uri: string): Promise<string> {
  const QRCode = await import('qrcode');
  return QRCode.toDataURL(uri, {
    type: 'image/png',
    width: 400,
    margin: 2,
    errorCorrectionLevel: 'H',
  });
}

function generateBackupCodes(): string[] {
  const codes: string[] = [];
  for (let i = 0; i < 10; i++) {
    // Generate 8-character alphanumeric code
    const code = randomBytes(5)
      .toString('base64')
      .replace(/[^a-zA-Z0-9]/g, '')
      .slice(0, 8)
      .toUpperCase();
    codes.push(code);
  }
  return codes;
}
```

### tRPC MFA Router
```typescript
// src/server/trpc/routers/mfa.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { authenticatedProcedure, router } from '../trpc';
import { initiateMFASetup, completeMFASetup } from '@/lib/mfa/setup';
import { verifyTOTP } from '@/lib/mfa/totp';
import { userRateLimit } from '../middleware/ratelimit';

export const mfaRouter = router({
  // Start MFA setup
  setup: authenticatedProcedure
    .mutation(async ({ ctx }) => {
      const user = await db.query.users.findFirst({
        where: eq(users.id, ctx.user!.sub),
        columns: { id: true, email: true, mfaEnabled: true },
      });
      
      if (!user) {
        throw new TRPCError({ code: 'NOT_FOUND', message: 'User not found' });
      }
      
      if (user.mfaEnabled) {
        throw new TRPCError({ 
          code: 'CONFLICT', 
          message: 'MFA is already enabled' 
        });
      }
      
      const result = await initiateMFASetup(user.id, user.email);
      
      return {
        setupToken: result.setupToken,
        qrCodeUrl: result.qrCodeDataUrl,
        manualEntryCode: result.manualEntryCode,
      };
    }),
  
  // Complete MFA setup with verification code
  completeSetup: authenticatedProcedure
    .input(z.object({
      setupToken: z.string().uuid(),
      verificationCode: z.string().length(6).regex(/^\d+$/),
    }))
    .mutation(async ({ input, ctx }) => {
      const result = await completeMFASetup(
        input.setupToken,
        input.verificationCode
      );
      
      if (!result.success) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: result.error || 'Verification failed',
        });
      }
      
      return {
        success: true,
        backupCodes: result.backupCodes,
        message: 'MFA enabled successfully. Save your backup codes securely.',
      };
    }),
  
  // Verify MFA code (for login flow)
  verify: publicProcedure
    .use(userRateLimit('mfaVerify'))
    .input(z.object({
      sessionToken: z.string(), // Temporary token from login step 1
      code: z.string().length(6).regex(/^\d+$/),
    }))
    .mutation(async ({ input }) => {
      // Retrieve pending session
      const pendingSession = await getPendingMFASession(input.sessionToken);
      if (!pendingSession) {
        throw new TRPCError({
          code: 'UNAUTHORIZED',
          message: 'Session expired or invalid',
        });
      }
      
      // Get user's MFA secret
      const user = await db.query.users.findFirst({
        where: eq(users.id, pendingSession.userId),
        columns: { mfaSecret: true, mfaEnabled: true },
      });
      
      if (!user?.mfaEnabled || !user.mfaSecret) {
        throw new TRPCError({
          code: 'INTERNAL_SERVER_ERROR',
          message: 'MFA configuration error',
        });
      }
      
      // Check if it's a backup code
      if (input.code.length === 8) {
        // Handle backup code (implemented in slice-d)
        const backupResult = await verifyBackupCode(pendingSession.userId, input.code);
        if (!backupResult.valid) {
          throw new TRPCError({
            code: 'UNAUTHORIZED',
            message: 'Invalid backup code',
          });
        }
      } else {
        // Verify TOTP
        const verification = verifyTOTP(user.mfaSecret, input.code);
        if (!verification.valid) {
          throw new TRPCError({
            code: 'UNAUTHORIZED',
            message: 'Invalid verification code',
          });
        }
      }
      
      // Complete login - issue tokens
      const tokens = await completeLogin(pendingSession.userId);
      
      // Clean up pending session
      await clearPendingMFASession(input.sessionToken);
      
      return {
        success: true,
        accessToken: tokens.accessToken,
        refreshToken: tokens.refreshToken,
        expiresIn: 3600,
      };
    }),
  
  // Disable MFA (requires verification)
  disable: authenticatedProcedure
    .input(z.object({
      verificationCode: z.string().length(6).regex(/^\d+$/),
    }))
    .mutation(async ({ input, ctx }) => {
      const userId = ctx.user!.sub;
      
      const user = await db.query.users.findFirst({
        where: eq(users.id, userId),
        columns: { mfaSecret: true, mfaEnabled: true },
      });
      
      if (!user?.mfaEnabled) {
        throw new TRPCError({
          code: 'CONFLICT',
          message: 'MFA is not enabled',
        });
      }
      
      // Verify code before disabling
      const verification = verifyTOTP(user.mfaSecret, input.verificationCode);
      if (!verification.valid) {
        throw new TRPCError({
          code: 'UNAUTHORIZED',
          message: 'Invalid verification code',
        });
      }
      
      // Disable MFA
      await db.update(users)
        .set({
          mfaEnabled: false,
          mfaSecret: null,
          mfaBackupCodes: null,
          updatedAt: new Date(),
        })
        .where(eq(users.id, userId));
      
      // Log event
      await logSecurityEvent({
        type: 'MFA_DISABLED',
        userId,
        timestamp: new Date(),
      });
      
      return { success: true, message: 'MFA disabled successfully' };
    }),
});

export type MfaRouter = typeof mfaRouter;
```

## Business Rules & Invariants
1. Setup session expires after 10 minutes
2. User must verify a code before MFA is fully enabled
3. Backup codes generated and shown once on setup completion
4. QR code compatible with Google Authenticator, Authy, Microsoft Authenticator
5. Manual entry code provided as fallback

## Edge Cases
1. **User loses setup session** — Must restart setup (old secret discarded)
2. **Code verification fails** — Allow retry, session still valid
3. **MFA already enabled** — Return conflict, require disable first
4. **User clears cookies mid-setup** — Session in Redis survives
5. **Multiple setup attempts** — Each generates new session, old one expires

## Tests

### src/lib/mfa/setup.test.ts
- `should initiate setup session`: Verifies session creation
- `should generate QR code`: Verifies QR generation
- `should complete setup with valid code`: Verifies enable flow
- `should reject setup with invalid code`: Verifies validation
- `should expire setup session`: Verifies TTL enforcement
- `should generate unique backup codes`: Verifies code uniqueness

### src/server/trpc/routers/mfa.test.ts
- `should return setup token`: Verifies setup endpoint
- `should reject duplicate MFA setup`: Verifies conflict detection
- `should complete setup flow`: Verifies end-to-end

## Verification
```bash
npm run test:unit src/lib/mfa/setup.test.ts
npm run test:unit src/server/trpc/routers/mfa.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → MFA
