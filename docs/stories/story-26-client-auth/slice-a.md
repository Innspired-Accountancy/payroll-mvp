# Slice a: Client Portal Login and MFA Flow

**Story:** story-26-client-auth
**Epic:** epic-07-employer-portal
**Effort:** M
**Dependencies:** story-05-rbac, story-04-mfa

## Goal

Implement the client portal authentication flow including login page, credential validation, MFA verification, and initial session establishment with employer scoping.

## Decision Checklist

- [x] All libraries/packages named: `better-auth@latest`, `zod@3.x`, `drizzle-orm@latest`
- [x] SDK methods identified: `betterAuth.api.signInEmail()`, `betterAuth.api.verifyTOTP()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: LoginInput, LoginResponse, MFAInput, MFAResponse
- [x] Configuration variables: `AUTH_SECRET`, `AUTH_URL`, `SESSION_TIMEOUT_MINUTES=30`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ Security → MFA requirements
- 02-09-identity-access-spec.md:§ Client Portal Roles → Role definitions
- 02-09-identity-access-spec.md:§ API Contracts → Login endpoints

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(portal)/login/page.tsx` | create | Client portal login page |
| `src/app/(portal)/mfa/page.tsx` | create | MFA verification page |
| `src/lib/auth/client.ts` | create | Client auth utilities |
| `src/lib/auth/validation.ts` | create | Login input validation |
| `src/server/api/routers/auth.ts` | update | Add client auth procedures |
| `src/lib/db/schema/employer-users.ts` | create | Employer portal user table |

## Responsibilities
1. Render client-specific login form (email/password)
2. Validate credentials against employer portal user records
3. Enforce MFA for all client users
4. Detect user role and employer association
5. Establish secure session with employer scoping
6. Redirect to appropriate dashboard based on role

## Contracts

### LoginInput Schema
```typescript
// src/lib/auth/validation.ts
import { z } from 'zod';

export const clientLoginSchema = z.object({
  email: z.string().email('Enter a valid email address'),
  password: z.string().min(1, 'Password is required'),
  rememberMe: z.boolean().default(false),
});

export type ClientLoginInput = z.infer<typeof clientLoginSchema>;
```

### LoginResponse Type
```typescript
// src/lib/auth/types.ts
export interface ClientLoginResponse {
  success: boolean;
  mfaRequired: boolean;
  tempToken?: string;  // Short-lived token for MFA step
  error?: string;
}
```

### MFAInput Schema
```typescript
// src/lib/auth/validation.ts
export const mfaVerifySchema = z.object({
  tempToken: z.string(),
  code: z.string().length(6, 'Enter 6-digit code'),
});

export type MFAVerifyInput = z.infer<typeof mfaVerifySchema>;
```

### MFAResponse Type
```typescript
// src/lib/auth/types.ts
export interface MFAVerifyResponse {
  success: boolean;
  accessToken?: string;
  refreshToken?: string;
  expiresIn: number;
  user: {
    id: string;
    email: string;
    firstName: string;
    lastName: string;
    role: 'admin' | 'manager' | 'viewer';
    employerId: string;
    employerName: string;
    canApprovePayroll: boolean;
    canEditEmployees: boolean;
  };
  error?: string;
}
```

### Auth Router Procedures
```typescript
// src/server/api/routers/auth.ts
export const authRouter = {
  // Client login - returns temp token if MFA required
  clientLogin: publicProcedure
    .input(clientLoginSchema)
    .mutation(async ({ input, ctx }): Promise<ClientLoginResponse> => {
      // Validate credentials
      // Check account lockout
      // Increment failed attempts on failure
      // Return temp token for MFA if credentials valid
    }),

  // Verify MFA and complete login
  verifyMFA: publicProcedure
    .input(mfaVerifySchema)
    .mutation(async ({ input, ctx }): Promise<MFAVerifyResponse> => {
      // Validate temp token
      // Verify TOTP code
      // Create session with employer scoping
      // Clear failed login attempts
      // Return full session
    }),
};
```

## Business Rules & Invariants
1. All client portal users must complete MFA verification
2. Account locks after 5 consecutive failed login attempts
3. Locked accounts require 30-minute cooldown or admin unlock
4. Sessions expire after 30 minutes of inactivity
5. Temp tokens for MFA step expire after 5 minutes
6. User can only access data for their assigned employer
7. Failed MFA attempts count toward lockout threshold

## Edge Cases
1. **User not found** — Return generic error (don't reveal existence)
2. **Account locked** — Return error with remaining lockout time
3. **MFA not configured** — Require MFA setup before allowing access
4. **Expired temp token** — Require re-login from beginning
5. **Concurrent login attempts** — Allow; last valid wins
6. **Employer inactive/suspended** — Block login with specific message

## Tests

### src/lib/auth/client.test.ts
- `should authenticate valid credentials`: Successful login initiation
- `should require MFA for all client users`: MFA flag returned
- `should lock account after 5 failures`: Lockout mechanism
- `should return generic error for non-existent user`: Security
- `should validate email format`: Input validation
- `should validate password presence`: Input validation

### src/server/api/routers/auth.test.ts
- `should verify valid MFA code`: Successful MFA verification
- `should reject invalid MFA code`: MFA validation
- `should reject expired temp token`: Token expiry
- `should include employer scope in session`: Tenant isolation
- `should return correct user role`: Role detection
- `should clear failed attempts on success`: Lockout reset

## Verification
```bash
npm run test:unit src/lib/auth/client.test.ts
npm run test:unit src/server/api/routers/auth.test.ts
npm run lint
npm run typecheck
npm run build
```

## Source Sections
- 02-06-employer-portal-spec.md § User Journeys → Journey 1
- 02-06-employer-portal-spec.md § Security → MFA and session requirements
- 02-09-identity-access-spec.md § Client Portal Roles → Role definitions
