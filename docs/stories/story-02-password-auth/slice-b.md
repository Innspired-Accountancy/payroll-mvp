# Slice b: Login Endpoint and Validation

**Story:** story-02-password-auth
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-a

## Goal

Implement the login endpoint with email/password validation, secure credential verification using Argon2id, and proper error handling including account lockout tracking.

## Decision Checklist

- [x] All libraries/packages named: `better-auth@0.8.x`, `zod@3.x`
- [x] SDK methods identified: `auth.api.signInEmail()`, `auth.api.getSession()`
- [x] External service endpoints: N/A (uses BetterAuth internal)
- [x] Data contracts defined: Login input/output schemas
- [x] Configuration variables: `MAX_LOGIN_ATTEMPTS=5`, `LOCKOUT_DURATION_MINUTES=30`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ API Contracts → POST /api/v1/auth/login
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Password policy, account lockout

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/auth/validation.ts` | create | Login input validation schemas |
| `src/lib/auth/errors.ts` | create | Auth error codes and messages |
| `src/lib/auth/lockout.ts` | create | Account lockout logic |
| `src/server/trpc/routers/auth.ts` | create | tRPC auth router |
| `src/app/(auth)/login/page.tsx` | create | Login page component |

## Responsibilities
1. Validate login input (email format, password presence)
2. Check account lockout status before credential verification
3. Verify credentials via BetterAuth
4. Track failed login attempts
5. Return appropriate error responses (without credential exposure)

## Contracts

### Login Input Schema
```typescript
import { z } from 'zod';

export const loginInputSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(1, 'Password is required'),
  callbackUrl: z.string().optional(),
});

export type LoginInput = z.infer<typeof loginInputSchema>;

export const loginResponseSchema = z.object({
  success: z.boolean(),
  redirectTo: z.string().optional(),
  mfaRequired: z.boolean().optional(),
  error: z.string().optional(),
});

export type LoginResponse = z.infer<typeof loginResponseSchema>;
```

### Lockout Logic
```typescript
// src/lib/auth/lockout.ts
import { eq } from 'drizzle-orm';
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';

const MAX_FAILED_ATTEMPTS = 5;
const LOCKOUT_DURATION_MS = 30 * 60 * 1000; // 30 minutes

export async function isAccountLocked(userId: string): Promise<boolean> {
  const user = await db.query.users.findFirst({
    where: eq(users.id, userId),
    columns: { lockedUntil: true },
  });
  
  if (!user?.lockedUntil) return false;
  
  return new Date(user.lockedUntil) > new Date();
}

export async function recordFailedLogin(email: string): Promise<void> {
  const user = await db.query.users.findFirst({
    where: eq(users.email, email.toLowerCase()),
    columns: { id: true, failedLoginAttempts: true },
  });
  
  if (!user) return; // Don't reveal if email exists
  
  const newAttempts = (user.failedLoginAttempts || 0) + 1;
  const shouldLock = newAttempts >= MAX_FAILED_ATTEMPTS;
  
  await db.update(users)
    .set({
      failedLoginAttempts: newAttempts,
      lockedUntil: shouldLock ? new Date(Date.now() + LOCKOUT_DURATION_MS) : undefined,
    })
    .where(eq(users.id, user.id));
}

export async function resetFailedAttempts(userId: string): Promise<void> {
  await db.update(users)
    .set({
      failedLoginAttempts: 0,
      lockedUntil: null,
    })
    .where(eq(users.id, userId));
}

export function getLockoutRemainingMs(lockedUntil: Date): number {
  return Math.max(0, new Date(lockedUntil).getTime() - Date.now());
}
```

### tRPC Auth Router
```typescript
// src/server/trpc/routers/auth.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { publicProcedure, router } from '../trpc';
import { auth } from '@/lib/auth';
import { 
  isAccountLocked, 
  recordFailedLogin, 
  resetFailedAttempts,
  getLockoutRemainingMs 
} from '@/lib/auth/lockout';
import { loginInputSchema } from '@/lib/auth/validation';
import { db } from '@/lib/db';
import { users } from '@/lib/db/schema';
import { eq } from 'drizzle-orm';

export const authRouter = router({
  login: publicProcedure
    .input(loginInputSchema)
    .mutation(async ({ input, ctx }) => {
      const { email, password, callbackUrl } = input;
      
      // Find user to check lockout
      const user = await db.query.users.findFirst({
        where: eq(users.email, email.toLowerCase()),
        columns: { id: true, status: true, lockedUntil: true },
      });
      
      // Check if user exists and is active
      if (!user || user.status !== 'active') {
        // Generic error to prevent user enumeration
        throw new TRPCError({
          code: 'UNAUTHORIZED',
          message: 'Invalid credentials',
        });
      }
      
      // Check account lockout
      if (await isAccountLocked(user.id)) {
        const remainingMs = getLockoutRemainingMs(user.lockedUntil!);
        const remainingMinutes = Math.ceil(remainingMs / 60000);
        
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: `Account locked. Try again in ${remainingMinutes} minutes.`,
        });
      }
      
      try {
        // Attempt sign in via BetterAuth
        const result = await auth.api.signInEmail({
          body: { email, password },
          headers: ctx.headers,
        });
        
        // Reset failed attempts on success
        await resetFailedAttempts(user.id);
        
        // Check if MFA is required (handled in story-04-mfa)
        const mfaRequired = false; // Will be implemented in story-04
        
        return {
          success: true,
          redirectTo: callbackUrl || '/dashboard',
          mfaRequired,
          userId: result.user.id,
        };
        
      } catch (error) {
        // Record failed attempt
        await recordFailedLogin(email);
        
        // Generic error to prevent credential guessing
        throw new TRPCError({
          code: 'UNAUTHORIZED',
          message: 'Invalid credentials',
        });
      }
    }),
});

export type AuthRouter = typeof authRouter;
```

### Error Codes
```typescript
// src/lib/auth/errors.ts
export const AuthErrorCodes = {
  INVALID_CREDENTIALS: 'INVALID_CREDENTIALS',
  ACCOUNT_LOCKED: 'ACCOUNT_LOCKED',
  ACCOUNT_INACTIVE: 'ACCOUNT_INACTIVE',
  MFA_REQUIRED: 'MFA_REQUIRED',
  SESSION_EXPIRED: 'SESSION_EXPIRED',
  RATE_LIMITED: 'RATE_LIMITED',
} as const;

export const AuthErrorMessages: Record<string, string> = {
  [AuthErrorCodes.INVALID_CREDENTIALS]: 'Invalid email or password',
  [AuthErrorCodes.ACCOUNT_LOCKED]: 'Account is temporarily locked due to failed login attempts',
  [AuthErrorCodes.ACCOUNT_INACTIVE]: 'Account is not active',
  [AuthErrorCodes.MFA_REQUIRED]: 'Multi-factor authentication required',
  [AuthErrorCodes.SESSION_EXPIRED]: 'Your session has expired. Please log in again',
  [AuthErrorCodes.RATE_LIMITED]: 'Too many attempts. Please try again later',
};
```

## Business Rules & Invariants
1. Account locks after 5 failed attempts for 30 minutes
2. Generic error messages to prevent user enumeration ("Invalid credentials")
3. Failed attempts reset on successful login
4. Inactive accounts cannot log in
5. All error responses have consistent structure

## Edge Cases
1. **User not found** — Return generic "Invalid credentials" error
2. **Account locked during login attempt** — Check lockout before credential check
3. **Password correct but account inactive** — Check status before password verify
4. **Clock skew on lockout** — Use server time consistently
5. **Concurrent login attempts** — Row-level locking on user record

## Tests

### src/server/trpc/routers/auth.test.ts
- `should login with valid credentials`: Verifies successful login
- `should reject invalid email format`: Verifies Zod validation
- `should reject wrong password`: Verifies credential check
- `should lock account after 5 failures`: Verifies lockout trigger
- `should reject login when account locked`: Verifies lockout enforcement
- `should reset failed attempts on success`: Verifies reset behavior
- `should reject inactive account login`: Verifies status check
- `should return generic error for non-existent user`: Verifies user enumeration prevention

## Verification
```bash
npm run test:unit src/server/trpc/routers/auth.test.ts
npm run test:integration src/lib/auth/lockout.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § API Contracts → POST /api/v1/auth/login
- 02-09-identity-access-spec.md § Non-Functional Requirements → Security, Password policy
