# Slice c: MFA Verification Endpoint

**Story:** story-04-mfa
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-b

## Goal

Integrate MFA verification into the login flow, requiring TOTP codes for bureau users and supporting enforcement policies for different user types.

## Decision Checklist

- [x] All libraries/packages named: `ioredis@5.x` (for pending sessions)
- [x] SDK methods identified: `redis.setex()`, `redis.get()`, `redis.del()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Pending MFA session, login response with MFA flag
- [x] Configuration variables: `MFA_PENDING_SESSION_EXPIRY=300` (5 minutes)
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ API Contracts → POST /api/v1/auth/mfa/verify
- 02-09-identity-access-spec.md:§ User Journeys → Bureau User Login

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/mfa/pending-session.ts` | create | Pending MFA session management |
| `src/lib/mfa/policy.ts` | create | MFA enforcement policies |
| `src/server/trpc/routers/auth.ts` | update | Modify login to handle MFA |
| `src/app/(auth)/mfa/verify/page.tsx` | create | MFA verification UI |

## Responsibilities
1. Modify login flow to return mfa_required flag for MFA-enabled users
2. Create pending MFA session for intermediate state
3. Validate MFA codes against stored secret
4. Enforce MFA policy (required for bureau, optional for clients)
5. Complete login after successful MFA verification

## Contracts

### Pending MFA Session
```typescript
// src/lib/mfa/pending-session.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);
const PENDING_PREFIX = 'mfa:pending:';
const PENDING_EXPIRY = 300; // 5 minutes

export interface PendingMFASession {
  userId: string;
  email: string;
  userType: 'bureau' | 'client' | 'employee';
  roles: string[];
  ipAddress?: string;
  userAgent?: string;
  createdAt: string;
}

export async function createPendingMFASession(
  session: Omit<PendingMFASession, 'createdAt'>
): Promise<string> {
  const token = crypto.randomUUID();
  const data: PendingMFASession = {
    ...session,
    createdAt: new Date().toISOString(),
  };
  
  await redis.setex(
    `${PENDING_PREFIX}${token}`,
    PENDING_EXPIRY,
    JSON.stringify(data)
  );
  
  return token;
}

export async function getPendingMFASession(
  token: string
): Promise<PendingMFASession | null> {
  const data = await redis.get(`${PENDING_PREFIX}${token}`);
  if (!data) return null;
  return JSON.parse(data);
}

export async function clearPendingMFASession(token: string): Promise<void> {
  await redis.del(`${PENDING_PREFIX}${token}`);
}

export async function extendPendingMFASession(token: string): Promise<void> {
  await redis.expire(`${PENDING_PREFIX}${token}`, PENDING_EXPIRY);
}
```

### MFA Policy
```typescript
// src/lib/mfa/policy.ts
import type { User } from '@/lib/db/schema';

export interface MFAPolicyResult {
  required: boolean;
  enforced: boolean;
  message?: string;
}

export function checkMFAPolicy(user: User): MFAPolicyResult {
  // Bureau users: MFA always required
  if (user.userType === 'bureau') {
    if (!user.mfaEnabled) {
      return {
        required: true,
        enforced: true,
        message: 'MFA is required for bureau users. Please set up MFA to continue.',
      };
    }
    return { required: true, enforced: true };
  }
  
  // Client portal users: MFA optional by default
  if (user.userType === 'client') {
    // Could check bureau policy here for client MFA requirement
    return { required: user.mfaEnabled, enforced: false };
  }
  
  // Employee users: MFA optional
  if (user.userType === 'employee') {
    return { required: user.mfaEnabled, enforced: false };
  }
  
  return { required: false, enforced: false };
}

export function isMFARequiredForUserType(userType: string): boolean {
  return userType === 'bureau';
}
```

### Updated Login Flow
```typescript
// src/server/trpc/routers/auth.ts (updated login procedure)
login: publicProcedure
  .use(ipRateLimit('login'))
  .use(userRateLimit('login'))
  .input(loginInputSchema)
  .mutation(async ({ input, ctx }) => {
    const { email, password } = input;
    
    // ... credential verification from story-02 ...
    
    const user = await authenticateUser(email, password);
    if (!user) {
      throw new TRPCError({ code: 'UNAUTHORIZED', message: 'Invalid credentials' });
    }
    
    // Check MFA policy
    const mfaPolicy = checkMFAPolicy(user);
    
    if (mfaPolicy.required && user.mfaEnabled) {
      // MFA required and enabled - create pending session
      const pendingToken = await createPendingMFASession({
        userId: user.id,
        email: user.email,
        userType: user.userType as 'bureau' | 'client' | 'employee',
        roles: [], // Will be populated after MFA
        ipAddress: ctx.ip || undefined,
        userAgent: ctx.headers.get('user-agent') || undefined,
      });
      
      return {
        success: false,
        mfaRequired: true,
        pendingToken,
        message: 'MFA verification required',
      };
    }
    
    if (mfaPolicy.required && !user.mfaEnabled && mfaPolicy.enforced) {
      // MFA required but not enabled - redirect to setup
      const setupToken = await initiateMFASetup(user.id, user.email);
      
      return {
        success: false,
        mfaRequired: true,
        mfaSetupRequired: true,
        setupToken: setupToken.setupToken,
        message: mfaPolicy.message,
      };
    }
    
    // No MFA required - complete login immediately
    const tokens = await completeLogin(user.id);
    
    return {
      success: true,
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
      expiresIn: 3600,
    };
  }),
```

### MFA Verification Handler
```typescript
// src/lib/mfa/verify-handler.ts
import { verifyTOTP } from './totp';
import { getPendingMFASession, clearPendingMFASession } from './pending-session';
import { generateTokenPair } from '@/lib/jwt/tokens';
import { getUserRoles } from '@/lib/auth/roles';

export interface MFAVerifyResult {
  success: boolean;
  tokens?: {
    accessToken: string;
    refreshToken: string;
    expiresIn: number;
  };
  error?: string;
  errorCode?: string;
}

export async function verifyMFACode(
  pendingToken: string,
  code: string,
  ipAddress?: string,
  userAgent?: string
): Promise<MFAVerifyResult> {
  // Get pending session
  const session = await getPendingMFASession(pendingToken);
  if (!session) {
    return {
      success: false,
      error: 'Session expired. Please log in again.',
      errorCode: 'SESSION_EXPIRED',
    };
  }
  
  // Get user's MFA secret
  const user = await db.query.users.findFirst({
    where: eq(users.id, session.userId),
    columns: { id: true, mfaSecret: true, mfaEnabled: true, mfaBackupCodes: true },
  });
  
  if (!user?.mfaEnabled || !user.mfaSecret) {
    return {
      success: false,
      error: 'MFA not properly configured',
      errorCode: 'MFA_NOT_CONFIGURED',
    };
  }
  
  let valid = false;
  let isBackupCode = false;
  
  // Check if it's a backup code (8 alphanumeric characters)
  if (/^[A-Z0-9]{8}$/.test(code)) {
    valid = await verifyBackupCode(user.id, code, user.mfaBackupCodes);
    isBackupCode = valid;
  } else {
    // Verify TOTP
    const verification = verifyTOTP(user.mfaSecret, code);
    valid = verification.valid;
  }
  
  if (!valid) {
    return {
      success: false,
      error: 'Invalid verification code',
      errorCode: 'INVALID_CODE',
    };
  }
  
  // Get user roles for token
  const roleAssignments = await getUserRoles(session.userId);
  
  // Create session
  const sessionId = await createSession({
    userId: session.userId,
    ipAddress: ipAddress || session.ipAddress,
    userAgent: userAgent || session.userAgent,
  });
  
  // Generate tokens
  const tokens = await generateTokenPair(
    { id: session.userId, email: session.email, userType: session.userType },
    roleAssignments,
    sessionId
  );
  
  // Clean up pending session
  await clearPendingMFASession(pendingToken);
  
  // If backup code was used, log security event
  if (isBackupCode) {
    await logSecurityEvent({
      type: 'MFA_BACKUP_CODE_USED',
      userId: session.userId,
      timestamp: new Date(),
    });
  }
  
  // Log successful MFA verification
  await logSecurityEvent({
    type: 'MFA_VERIFIED',
    userId: session.userId,
    timestamp: new Date(),
  });
  
  return {
    success: true,
    tokens: {
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
      expiresIn: 3600,
    },
  };
}
```

## Business Rules & Invariants
1. Bureau users must have MFA enabled to log in
2. Pending MFA sessions expire after 5 minutes
3. TOTP codes valid for 30 seconds with +/- 1 step tolerance
4. Backup codes can be used when authenticator unavailable
5. Successful MFA verification clears pending session immediately

## Edge Cases
1. **Pending session expires during verification** — Return SESSION_EXPIRED, require re-login
2. **Invalid code multiple times** — Rate limiting applies (5 per 15 min)
3. **Backup code reuse** — Each backup code single-use only
4. **User disabled while pending** — Check user status before completing
5. **Clock skew on TOTP** — Verified in slice-a with window tolerance

## Tests

### src/lib/mfa/verify-handler.test.ts
- `should verify valid TOTP code`: Verifies normal flow
- `should verify valid backup code`: Verifies backup code flow
- `should reject invalid code`: Verifies rejection
- `should reject expired pending session`: Verifies expiry
- `should consume backup code on use`: Verifies single-use
- `should generate tokens on success`: Verifies completion

## Verification
```bash
npm run test:unit src/lib/mfa/verify-handler.test.ts
npm run test:integration src/server/trpc/routers/auth.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § API Contracts → POST /api/v1/auth/mfa/verify
