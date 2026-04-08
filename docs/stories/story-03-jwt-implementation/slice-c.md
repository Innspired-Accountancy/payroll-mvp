# Slice c: Refresh Token Rotation

**Story:** story-03-jwt-implementation
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-b

## Goal

Implement refresh token rotation where using a refresh token invalidates the old one and issues a new token pair, preventing replay attacks and enabling secure long-lived sessions.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `uuid@latest`
- [x] SDK methods identified: `db.insert()`, `db.update()`, `db.query()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Refresh token storage schema, rotation logic
- [x] Configuration variables: `MAX_REFRESH_ROTATIONS=100` (detect abuse)
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Session timeout

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/refresh-tokens.ts` | create | Refresh token tracking table |
| `src/lib/jwt/rotation.ts` | create | Token rotation logic |
| `src/server/trpc/routers/token.ts` | create | Token refresh endpoint |
| `src/lib/auth/refresh.ts` | create | Refresh flow orchestration |

## Responsibilities
1. Store refresh tokens with metadata (userId, sessionId, version, expiresAt)
2. On refresh: validate token, increment version, issue new pair
3. Invalidate old refresh token (mark as rotated)
4. Detect and handle token reuse (potential theft)
5. Limit total rotations to detect abuse

## Contracts

### Refresh Token Schema
```typescript
// src/lib/db/schema/refresh-tokens.ts
import { pgTable, uuid, timestamp, integer, varchar, boolean } from 'drizzle-orm/pg-core';
import { users } from './users';
import { relations } from 'drizzle-orm';

export const refreshTokens = pgTable('refresh_tokens', {
  id: uuid('id').defaultRandom().primaryKey(),
  jti: varchar('jti', { length: 36 }).notNull().unique(), // JWT ID
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  sessionId: uuid('session_id').notNull(),
  version: integer('version').notNull().default(1),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  rotatedAt: timestamp('rotated_at', { withTimezone: true }), // When this was rotated (null if current)
  rotatedToJti: varchar('rotated_to_jti', { length: 36 }), // JTI of the replacement token
  revokedAt: timestamp('revoked_at', { withTimezone: true }), // When explicitly revoked
  ipAddress: varchar('ip_address', { length: 45 }), // IPv6 compatible
  userAgent: varchar('user_agent', { length: 500 }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

export const refreshTokensRelations = relations(refreshTokens, ({ one }) => ({
  user: one(users, { fields: [refreshTokens.userId], references: [users.id] }),
}));

export type RefreshToken = typeof refreshTokens.$inferSelect;
export type NewRefreshToken = typeof refreshTokens.$inferInsert;
```

### Token Rotation Logic
```typescript
// src/lib/jwt/rotation.ts
import { eq } from 'drizzle-orm';
import { db } from '@/lib/db';
import { refreshTokens } from '@/lib/db/schema';
import { verifyRefreshToken, TokenError } from './verify';
import { generateTokenPair } from './tokens';
import type { TokenPair } from './claims';

export interface RotationResult {
  success: boolean;
  tokens?: TokenPair;
  error?: RotationError;
}

export enum RotationError {
  INVALID_TOKEN = 'INVALID_TOKEN',
  TOKEN_EXPIRED = 'TOKEN_EXPIRED',
  TOKEN_REVOKED = 'TOKEN_REVOKED',
  TOKEN_REUSED = 'TOKEN_REUSED', // Potential theft
  SESSION_INVALID = 'SESSION_INVALID',
  USER_INACTIVE = 'USER_INACTIVE',
}

export async function rotateRefreshToken(
  refreshToken: string,
  ipAddress?: string,
  userAgent?: string
): Promise<RotationResult> {
  // 1. Verify the token cryptographically
  const verification = await verifyRefreshToken(refreshToken);
  
  if (!verification.valid) {
    if (verification.error === TokenError.EXPIRED) {
      return { success: false, error: RotationError.TOKEN_EXPIRED };
    }
    return { success: false, error: RotationError.INVALID_TOKEN };
  }
  
  const claims = verification.claims!;
  
  // 2. Look up token in database
  const storedToken = await db.query.refreshTokens.findFirst({
    where: eq(refreshTokens.jti, claims.jti),
  });
  
  if (!storedToken) {
    return { success: false, error: RotationError.INVALID_TOKEN };
  }
  
  // 3. Check if already rotated (potential theft)
  if (storedToken.rotatedAt) {
    // Token reuse detected - revoke entire session chain
    await revokeTokenFamily(storedToken.sessionId);
    return { success: false, error: RotationError.TOKEN_REUSED };
  }
  
  // 4. Check if explicitly revoked
  if (storedToken.revokedAt) {
    return { success: false, error: RotationError.TOKEN_REVOKED };
  }
  
  // 5. Check if version matches
  if (storedToken.version !== claims.version) {
    return { success: false, error: RotationError.INVALID_TOKEN };
  }
  
  // 6. Verify user is still active
  const user = await db.query.users.findFirst({
    where: eq(users.id, claims.sub),
    columns: { id: true, status: true, email: true, userType: true, bureauId: true, employerId: true },
  });
  
  if (!user || user.status !== 'active') {
    return { success: false, error: RotationError.USER_INACTIVE };
  }
  
  // 7. Get user's current roles
  const roleAssignments = await db.query.userRoleAssignments.findMany({
    where: eq(userRoleAssignments.userId, user.id),
  });
  
  // 8. Generate new token pair
  const newTokens = await generateTokenPair(user, roleAssignments, claims.sessionId);
  
  // 9. Parse new refresh token to get JTI
  const newRefreshClaims = await parseRefreshTokenClaims(newTokens.refreshToken);
  
  // 10. Mark old token as rotated
  await db.update(refreshTokens)
    .set({
      rotatedAt: new Date(),
      rotatedToJti: newRefreshClaims.jti,
    })
    .where(eq(refreshTokens.jti, claims.jti));
  
  // 11. Store new refresh token
  await db.insert(refreshTokens).values({
    jti: newRefreshClaims.jti,
    userId: user.id,
    sessionId: claims.sessionId,
    version: newRefreshClaims.version,
    expiresAt: newTokens.refreshTokenExpiresAt,
    ipAddress: ipAddress || null,
    userAgent: userAgent || null,
  });
  
  return { success: true, tokens: newTokens };
}

async function revokeTokenFamily(sessionId: string): Promise<void> {
  // Revoke all tokens in the session chain - potential theft response
  await db.update(refreshTokens)
    .set({ revokedAt: new Date() })
    .where(eq(refreshTokens.sessionId, sessionId));
  
  // Also revoke the session itself
  await db.update(sessions)
    .set({ revokedAt: new Date() })
    .where(eq(sessions.id, sessionId));
  
  // Log security event for admin review
  await logSecurityEvent({
    type: 'TOKEN_REUSE_DETECTED',
    sessionId,
    severity: 'high',
  });
}

async function parseRefreshTokenClaims(token: string): Promise<{ jti: string; version: number }> {
  // Parse without verification just to get JTI
  const parts = token.split('.');
  const payload = JSON.parse(Buffer.from(parts[1], 'base64').toString());
  return { jti: payload.jti, version: payload.version };
}
```

### tRPC Token Router
```typescript
// src/server/trpc/routers/token.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { publicProcedure, router } from '../trpc';
import { rotateRefreshToken, RotationError } from '@/lib/jwt/rotation';
import { ipRateLimit } from '../middleware/ratelimit';

export const tokenRouter = router({
  refresh: publicProcedure
    .use(ipRateLimit('login')) // Same limit as login
    .input(z.object({
      refreshToken: z.string(),
    }))
    .mutation(async ({ input, ctx }) => {
      const result = await rotateRefreshToken(
        input.refreshToken,
        ctx.ip || undefined,
        ctx.headers.get('user-agent') || undefined
      );
      
      if (!result.success) {
        const errorMap: Record<RotationError, { code: string; message: string }> = {
          [RotationError.INVALID_TOKEN]: { code: 'UNAUTHORIZED', message: 'Invalid refresh token' },
          [RotationError.TOKEN_EXPIRED]: { code: 'UNAUTHORIZED', message: 'Refresh token expired' },
          [RotationError.TOKEN_REVOKED]: { code: 'UNAUTHORIZED', message: 'Token has been revoked' },
          [RotationError.TOKEN_REUSED]: { 
            code: 'UNAUTHORIZED', 
            message: 'Security violation detected. Please log in again.' 
          },
          [RotationError.SESSION_INVALID]: { code: 'UNAUTHORIZED', message: 'Session no longer valid' },
          [RotationError.USER_INACTIVE]: { code: 'FORBIDDEN', message: 'Account is not active' },
        };
        
        const error = errorMap[result.error!];
        throw new TRPCError({
          code: error.code as any,
          message: error.message,
        });
      }
      
      return {
        accessToken: result.tokens!.accessToken,
        refreshToken: result.tokens!.refreshToken,
        expiresIn: 3600,
      };
    }),
});

export type TokenRouter = typeof tokenRouter;
```

## Business Rules & Invariants
1. Each refresh token can only be used once (single-use)
2. Reusing a rotated token triggers security alert and revokes entire session
3. New token inherits session ID for continuity
4. Version number increments on each rotation
5. Rotation chain is tracked (rotatedToJti) for audit

## Edge Cases
1. **Concurrent refresh attempts** — First succeeds, second gets TOKEN_REUSED
2. **Refresh during active session** — Normal operation, extends session
3. **User deactivated during session** — Next refresh fails with USER_INACTIVE
4. **Database unavailable during rotation** — Fail closed (don't issue new tokens)
5. **Token parse failure** — Return INVALID_TOKEN without crashing

## Tests

### src/lib/jwt/rotation.test.ts
- `should rotate valid refresh token`: Verifies normal rotation
- `should reject expired token`: Verifies expiry check
- `should reject reused token`: Verifies theft detection
- `should revoke session family on reuse`: Verifies cascade revoke
- `should reject revoked token`: Verifies revocation check
- `should reject inactive user`: Verifies user status check
- `should increment version on rotation`: Verifies version tracking

## Verification
```bash
npm run db:generate
npm run test:unit src/lib/jwt/rotation.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → Session management
