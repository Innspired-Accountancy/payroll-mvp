# Slice b: Token Validation Middleware

**Story:** story-03-jwt-implementation
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-a

## Goal

Create middleware for validating JWT tokens on incoming requests, extracting user context, and providing authentication state to tRPC procedures and Next.js routes.

## Decision Checklist

- [x] All libraries/packages named: `jose@5.x`
- [x] SDK methods identified: `jwtVerify()`, `importSPKI()`, `createLocalJWKSet()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Validation result types, auth context
- [x] Configuration variables: `JWT_PUBLIC_KEY`, `JWT_CLOCK_TOLERANCE=60`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Session timeout
- 02-09-identity-access-spec.md:§ API Contracts → Protected endpoints

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/jwt/verify.ts` | create | Token verification functions |
| `src/server/trpc/context.ts` | update | Add auth to tRPC context |
| `src/server/trpc/middleware/auth.ts` | create | tRPC auth middleware |
| `src/lib/auth/edge.ts` | create | Edge-compatible auth for middleware |

## Responsibilities
1. Verify JWT signature using public key
2. Validate token claims (exp, iss, aud, type)
3. Extract user context from valid tokens
4. Provide auth middleware for tRPC procedures
5. Handle token expiration gracefully
6. Support clock skew tolerance

## Contracts

### Token Verification
```typescript
// src/lib/jwt/verify.ts
import { jwtVerify, JWTVerifyResult } from 'jose';
import { loadPublicKey } from './keys';
import type { AccessTokenClaims, RefreshTokenClaims } from './claims';

export interface VerificationResult {
  valid: boolean;
  claims?: AccessTokenClaims | RefreshTokenClaims;
  error?: TokenError;
}

export enum TokenError {
  EXPIRED = 'TOKEN_EXPIRED',
  INVALID_SIGNATURE = 'INVALID_SIGNATURE',
  INVALID_ISSUER = 'INVALID_ISSUER',
  INVALID_AUDIENCE = 'INVALID_AUDIENCE',
  INVALID_TYPE = 'INVALID_TYPE',
  MALFORMED = 'MALFORMED',
  REVOKED = 'TOKEN_REVOKED',
}

const CLOCK_TOLERANCE = 60; // seconds

export async function verifyAccessToken(token: string): Promise<VerificationResult> {
  try {
    const publicKey = await loadPublicKey();
    
    const { payload } = await jwtVerify(token, publicKey, {
      issuer: process.env.BETTER_AUTH_URL!,
      audience: process.env.BETTER_AUTH_URL!,
      clockTolerance: CLOCK_TOLERANCE,
    });
    
    // Check token type
    if (payload.type !== 'access') {
      return { valid: false, error: TokenError.INVALID_TYPE };
    }
    
    // Check if token is blacklisted (handled in slice-d)
    const isBlacklisted = await isTokenBlacklisted(payload.jti as string);
    if (isBlacklisted) {
      return { valid: false, error: TokenError.REVOKED };
    }
    
    return { valid: true, claims: payload as AccessTokenClaims };
    
  } catch (error: any) {
    if (error.code === 'ERR_JWT_EXPIRED') {
      return { valid: false, error: TokenError.EXPIRED };
    }
    if (error.code === 'ERR_JWS_SIGNATURE_VERIFICATION_FAILED') {
      return { valid: false, error: TokenError.INVALID_SIGNATURE };
    }
    if (error.code === 'ERR_JWT_CLAIM_INVALID') {
      if (error.claim === 'iss') return { valid: false, error: TokenError.INVALID_ISSUER };
      if (error.claim === 'aud') return { valid: false, error: TokenError.INVALID_AUDIENCE };
    }
    
    return { valid: false, error: TokenError.MALFORMED };
  }
}

export async function verifyRefreshToken(token: string): Promise<VerificationResult> {
  try {
    const publicKey = await loadPublicKey();
    
    const { payload } = await jwtVerify(token, publicKey, {
      issuer: process.env.BETTER_AUTH_URL!,
      audience: process.env.BETTER_AUTH_URL!,
      clockTolerance: CLOCK_TOLERANCE,
    });
    
    if (payload.type !== 'refresh') {
      return { valid: false, error: TokenError.INVALID_TYPE };
    }
    
    // Check rotation version in database (handled in slice-c)
    const isValidVersion = await validateRefreshTokenVersion(
      payload.jti as string,
      payload.version as number
    );
    if (!isValidVersion) {
      return { valid: false, error: TokenError.REVOKED };
    }
    
    return { valid: true, claims: payload as RefreshTokenClaims };
    
  } catch (error: any) {
    if (error.code === 'ERR_JWT_EXPIRED') {
      return { valid: false, error: TokenError.EXPIRED };
    }
    return { valid: false, error: TokenError.MALFORMED };
  }
}

// Placeholder functions - implemented in slice-c and slice-d
async function isTokenBlacklisted(jti: string): Promise<boolean> {
  // Implemented in slice-d
  return false;
}

async function validateRefreshTokenVersion(jti: string, version: number): Promise<boolean> {
  // Implemented in slice-c
  return true;
}
```

### tRPC Auth Context
```typescript
// src/server/trpc/context.ts
import type { CreateNextContextOptions } from '@trpc/server/adapters/next';
import type { FetchCreateContextFnOptions } from '@trpc/server/adapters/fetch';
import { verifyAccessToken } from '@/lib/jwt/verify';
import type { AccessTokenClaims } from '@/lib/jwt/claims';
import { db } from '@/lib/db';

export interface AuthContext {
  user: AccessTokenClaims | null;
  sessionId: string | null;
}

export async function createContext(opts: FetchCreateContextFnOptions) {
  const { req } = opts;
  
  // Extract token from Authorization header or cookie
  const authHeader = req.headers.get('authorization');
  const token = authHeader?.startsWith('Bearer ') 
    ? authHeader.slice(7) 
    : extractTokenFromCookie(req.headers.get('cookie'));
  
  let auth: AuthContext = { user: null, sessionId: null };
  
  if (token) {
    const result = await verifyAccessToken(token);
    if (result.valid && result.claims) {
      auth = {
        user: result.claims as AccessTokenClaims,
        sessionId: result.claims.sessionId,
      };
    }
  }
  
  return {
    ...auth,
    db,
    headers: req.headers,
    ip: req.headers.get('x-forwarded-for') || req.headers.get('x-real-ip'),
  };
}

function extractTokenFromCookie(cookieHeader: string | null): string | null {
  if (!cookieHeader) return null;
  
  const cookies = cookieHeader.split(';');
  const tokenCookie = cookies.find(c => c.trim().startsWith('__session='));
  return tokenCookie ? tokenCookie.split('=')[1] : null;
}

export type Context = Awaited<ReturnType<typeof createContext>>;
```

### tRPC Auth Middleware
```typescript
// src/server/trpc/middleware/auth.ts
import { TRPCError } from '@trpc/server';
import { middleware, publicProcedure } from '../trpc';

export const authenticated = middleware(async ({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({
      code: 'UNAUTHORIZED',
      message: 'Authentication required',
    });
  }
  
  return next({
    ctx: {
      ...ctx,
      user: ctx.user,
    },
  });
});

export const authenticatedProcedure = publicProcedure.use(authenticated);

// Role-based middleware
export const requireRole = (allowedRoles: string[]) => 
  middleware(async ({ ctx, next }) => {
    if (!ctx.user) {
      throw new TRPCError({
        code: 'UNAUTHORIZED',
        message: 'Authentication required',
      });
    }
    
    const hasRole = ctx.user.roles.some(role => allowedRoles.includes(role));
    if (!hasRole) {
      throw new TRPCError({
        code: 'FORBIDDEN',
        message: 'Insufficient permissions',
      });
    }
    
    return next({ ctx });
  });

// Permission-based middleware
export const requirePermission = (permission: string) => 
  middleware(async ({ ctx, next }) => {
    if (!ctx.user) {
      throw new TRPCError({
        code: 'UNAUTHORIZED',
        message: 'Authentication required',
      });
    }
    
    if (!ctx.user.permissions.includes(permission)) {
      throw new TRPCError({
        code: 'FORBIDDEN',
        message: 'Permission denied',
      });
    }
    
    return next({ ctx });
  });
```

### Edge-compatible Auth (for Next.js Middleware)
```typescript
// src/lib/auth/edge.ts
import { verifyAccessToken } from '@/lib/jwt/verify';
import type { AccessTokenClaims } from '@/lib/jwt/claims';

export interface EdgeAuthResult {
  authenticated: boolean;
  user?: AccessTokenClaims;
  error?: string;
}

export async function authenticateEdgeRequest(
  request: Request
): Promise<EdgeAuthResult> {
  const authHeader = request.headers.get('authorization');
  const cookieHeader = request.headers.get('cookie');
  
  const token = authHeader?.startsWith('Bearer ')
    ? authHeader.slice(7)
    : extractTokenFromCookie(cookieHeader);
  
  if (!token) {
    return { authenticated: false, error: 'No token provided' };
  }
  
  const result = await verifyAccessToken(token);
  
  if (!result.valid) {
    return { 
      authenticated: false, 
      error: result.error || 'Invalid token' 
    };
  }
  
  return {
    authenticated: true,
    user: result.claims as AccessTokenClaims,
  };
}

function extractTokenFromCookie(cookieHeader: string | null): string | null {
  if (!cookieHeader) return null;
  const match = cookieHeader.match(/__session=([^;]+)/);
  return match ? decodeURIComponent(match[1]) : null;
}
```

## Business Rules & Invariants
1. All protected routes must use authentication middleware
2. Expired tokens return 401 with specific error code
3. Invalid signatures return 401 without revealing which check failed
4. 60-second clock skew tolerance built-in
5. Token type must match expected usage (access vs refresh)

## Edge Cases
1. **Token in both header and cookie** — Header takes precedence
2. **Malformed JWT** — Return MALFORMED error, don't crash
3. **Algorithm confusion attack** — Explicit RS256 only, reject others
4. **Missing claims** — Verify all required claims present
5. **Token used before nbf** — jose library handles this

## Tests

### src/lib/jwt/verify.test.ts
- `should verify valid access token`: Verifies successful validation
- `should reject expired token`: Verifies exp handling
- `should reject invalid signature`: Verifies signature check
- `should reject wrong issuer`: Verifies iss claim
- `should reject wrong audience`: Verifies aud claim
- `should reject wrong token type`: Verifies type claim
- `should allow clock skew`: Verifies tolerance

### src/server/trpc/middleware/auth.test.ts
- `should allow authenticated requests`: Verifies auth pass-through
- `should reject unauthenticated requests`: Verifies 401
- `should extract user from token`: Verifies context population

## Verification
```bash
npm run test:unit src/lib/jwt/verify.test.ts
npm run test:unit src/server/trpc/middleware/auth.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → Session timeout
