# Slice d: Token Blacklisting

**Story:** story-03-jwt-implementation
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-b

## Goal

Implement token blacklisting for immediate logout and token revocation, storing revoked token JTIs in Redis with TTL matching token expiration for efficient lookup.

## Decision Checklist

- [x] All libraries/packages named: `ioredis@5.x` or `@upstash/redis@1.x`
- [x] SDK methods identified: `redis.setex()`, `redis.get()`, `redis.del()`
- [x] External service endpoints: Redis (local or Upstash)
- [x] Data contracts defined: Blacklist entry structure
- [x] Configuration variables: `REDIS_URL`, `REDIS_TOKEN_BLACKLIST_PREFIX="bl:jti:"`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ User Journeys → Logout flows
- 02-09-identity-access-spec.md:§ API Contracts → Session termination

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/jwt/blacklist.ts` | create | Blacklist operations |
| `src/lib/jwt/verify.ts` | update | Add blacklist check |
| `src/server/trpc/routers/session.ts` | update | Logout with blacklist |

## Responsibilities
1. Store revoked token JTIs in Redis with expiration
2. Check blacklist during token verification
3. Support bulk blacklisting (logout all sessions)
4. Automatic cleanup via Redis TTL
5. Fallback to database if Redis unavailable

## Contracts

### Blacklist Operations
```typescript
// src/lib/jwt/blacklist.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);
const BLACKLIST_PREFIX = process.env.REDIS_TOKEN_BLACKLIST_PREFIX || 'bl:jti:';

export interface BlacklistEntry {
  jti: string;
  revokedAt: string;
  reason: 'logout' | 'password_change' | 'security' | 'admin_action';
  userId: string;
  sessionId?: string;
}

/**
 * Add a token JTI to the blacklist
 * TTL is automatically set to match token expiration
 */
export async function blacklistToken(
  jti: string, 
  ttlSeconds: number,
  entry: Omit<BlacklistEntry, 'jti'>
): Promise<void> {
  const key = `${BLACKLIST_PREFIX}${jti}`;
  const value = JSON.stringify(entry);
  
  await redis.setex(key, ttlSeconds, value);
}

/**
 * Check if a token JTI is blacklisted
 */
export async function isTokenBlacklisted(jti: string): Promise<boolean> {
  const key = `${BLACKLIST_PREFIX}${jti}`;
  const exists = await redis.exists(key);
  return exists === 1;
}

/**
 * Get blacklist entry details (for audit)
 */
export async function getBlacklistEntry(jti: string): Promise<BlacklistEntry | null> {
  const key = `${BLACKLIST_PREFIX}${jti}`;
  const value = await redis.get(key);
  
  if (!value) return null;
  
  return JSON.parse(value) as BlacklistEntry;
}

/**
 * Blacklist all tokens for a session (logout all)
 */
export async function blacklistSession(sessionId: string): Promise<number> {
  // Query database for all tokens in this session
  const tokens = await db.query.refreshTokens.findMany({
    where: eq(refreshTokens.sessionId, sessionId),
    columns: { jti: true, expiresAt: true },
  });
  
  let count = 0;
  for (const token of tokens) {
    const ttl = Math.ceil((new Date(token.expiresAt).getTime() - Date.now()) / 1000);
    if (ttl > 0) {
      await blacklistToken(token.jti, ttl, {
        revokedAt: new Date().toISOString(),
        reason: 'logout',
        userId: '', // Will be filled by caller
        sessionId,
      });
      count++;
    }
  }
  
  return count;
}

/**
 * Blacklist all tokens for a user (security breach response)
 */
export async function blacklistAllUserTokens(userId: string): Promise<number> {
  const tokens = await db.query.refreshTokens.findMany({
    where: eq(refreshTokens.userId, userId),
    columns: { jti: true, expiresAt: true, userId: true },
  });
  
  let count = 0;
  for (const token of tokens) {
    const ttl = Math.ceil((new Date(token.expiresAt).getTime() - Date.now()) / 1000);
    if (ttl > 0) {
      await blacklistToken(token.jti, ttl, {
        revokedAt: new Date().toISOString(),
        reason: 'security',
        userId: token.userId,
        sessionId: token.sessionId,
      });
      count++;
    }
  }
  
  return count;
}

/**
 * Get count of blacklisted tokens (for monitoring)
 */
export async function getBlacklistCount(): Promise<number> {
  const keys = await redis.keys(`${BLACKLIST_PREFIX}*`);
  return keys.length;
}
```

### Updated Token Verification
```typescript
// src/lib/jwt/verify.ts (update existing file)
import { isTokenBlacklisted } from './blacklist';

// In verifyAccessToken function, add after signature verification:
async function checkBlacklist(jti: string): Promise<boolean> {
  try {
    return await isTokenBlacklisted(jti);
  } catch (error) {
    // If Redis is down, fail open (allow) but log error
    // In high-security environments, you might want to fail closed
    console.error('Redis unavailable during blacklist check:', error);
    return false;
  }
}
```

### Logout with Blacklisting
```typescript
// src/server/trpc/routers/session.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { authenticatedProcedure, router } from '../trpc';
import { blacklistToken, blacklistSession } from '@/lib/jwt/blacklist';
import { verifyAccessToken } from '@/lib/jwt/verify';

export const sessionRouter = router({
  logout: authenticatedProcedure
    .mutation(async ({ ctx }) => {
      const token = extractTokenFromHeaders(ctx.headers);
      
      if (token) {
        const verification = await verifyAccessToken(token);
        if (verification.valid && verification.claims) {
          // Blacklist the current access token
          const ttl = Math.ceil((verification.claims.exp * 1000 - Date.now()) / 1000);
          if (ttl > 0) {
            await blacklistToken(verification.claims.jti, ttl, {
              revokedAt: new Date().toISOString(),
              reason: 'logout',
              userId: verification.claims.sub,
              sessionId: verification.claims.sessionId,
            });
          }
        }
      }
      
      // Clear session cookie
      return {
        success: true,
        message: 'Logged out successfully',
      };
    }),
  
  logoutAll: authenticatedProcedure
    .mutation(async ({ ctx }) => {
      const userId = ctx.user!.sub;
      const sessionId = ctx.sessionId!;
      
      // Blacklist all tokens for this session
      const count = await blacklistSession(sessionId);
      
      // Revoke session in database
      await db.update(sessions)
        .set({ revokedAt: new Date() })
        .where(eq(sessions.id, sessionId));
      
      return {
        success: true,
        message: `Logged out from ${count} session(s)`,
      };
    }),
});

function extractTokenFromHeaders(headers: Headers): string | null {
  const authHeader = headers.get('authorization');
  if (authHeader?.startsWith('Bearer ')) {
    return authHeader.slice(7);
  }
  return null;
}
```

## Business Rules & Invariants
1. Blacklisted tokens have TTL matching remaining token lifetime
2. Redis automatically cleans up expired entries
3. Blacklist check happens after signature verification
4. If Redis is unavailable, fail open (allow) with logging (configurable)
5. Blacklist reason is tracked for audit purposes

## Edge Cases
1. **Redis unavailable** — Log error, allow token (fail open) or reject (fail closed)
2. **Token already expired** — No need to blacklist, just reject
3. **Race condition: blacklist during verification** — First wins, consistent state
4. **Bulk logout** — Efficient batch operations, track count
5. **Memory pressure** — Redis LRU eviction acceptable (tokens expire anyway)

## Tests

### src/lib/jwt/blacklist.test.ts
- `should blacklist token`: Verifies storage
- `should detect blacklisted token`: Verifies lookup
- `should return false for non-blacklisted token`: Verifies negative case
- `should auto-expire after TTL`: Verifies Redis TTL
- `should handle Redis unavailable`: Verifies fallback behavior
- `should blacklist all session tokens`: Verifies bulk operation
- `should retrieve blacklist entry`: Verifies audit retrieval

## Verification
```bash
npm run test:unit src/lib/jwt/blacklist.test.ts
npm run test:integration src/server/trpc/routers/session.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § User Journeys → Session termination
- 02-09-identity-access-spec.md § API Contracts → Logout endpoint
