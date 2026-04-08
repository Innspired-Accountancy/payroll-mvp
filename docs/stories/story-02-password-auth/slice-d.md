# Slice d: Account Lockout and Rate Limiting

**Story:** story-02-password-auth
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-b

## Goal

Implement comprehensive rate limiting and account lockout protection to prevent brute force attacks, credential stuffing, and automated abuse of authentication endpoints.

## Decision Checklist

- [x] All libraries/packages named: `@upstash/ratelimit@latest`, `@upstash/redis@latest` OR `rate-limiter-flexible@5.x`
- [x] SDK methods identified: `new RateLimiterRedis()`, `consume()`, `get()`, `delete()`
- [x] External service endpoints: Upstash Redis (cloud) or local Redis instance
- [x] Data contracts defined: Rate limit configuration per endpoint
- [x] Configuration variables: `REDIS_URL`, `RATE_LIMIT_LOGIN=5`, `RATE_LIMIT_WINDOW=900`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Account lockout after 5 failed attempts
- 02-09-identity-access-spec.md:§ API Contracts → All auth endpoints

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/ratelimit/index.ts` | create | Rate limiting service |
| `src/lib/ratelimit/config.ts` | create | Rate limit configurations |
| `src/server/trpc/middleware/ratelimit.ts` | create | tRPC rate limit middleware |
| `src/server/trpc/middleware/lockout.ts` | create | Account lockout middleware |

## Responsibilities
1. Implement IP-based rate limiting for auth endpoints
2. Implement user-based rate limiting for login attempts
3. Combine rate limiting with account lockout from slice-b
4. Provide clear error responses with retry-after headers
5. Support different limits for different endpoints

## Contracts

### Rate Limit Configuration
```typescript
// src/lib/ratelimit/config.ts
export const rateLimitConfig = {
  // Per IP address limits
  ip: {
    login: { points: 10, duration: 900 }, // 10 attempts per 15 minutes
    passwordReset: { points: 5, duration: 3600 }, // 5 per hour
    register: { points: 3, duration: 3600 }, // 3 per hour
  },
  // Per user account limits (by email/userId)
  user: {
    login: { points: 5, duration: 900 }, // 5 attempts per 15 minutes
    passwordReset: { points: 3, duration: 3600 }, // 3 per hour
    mfaVerify: { points: 5, duration: 900 }, // 5 per 15 minutes (story-04)
  },
} as const;

export type RateLimitKey = keyof typeof rateLimitConfig.ip;
```

### Rate Limiter Service
```typescript
// src/lib/ratelimit/index.ts
import { RateLimiterRedis } from 'rate-limiter-flexible';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

export class RateLimiter {
  private limiters: Map<string, RateLimiterRedis>;
  
  constructor() {
    this.limiters = new Map();
  }
  
  getLimiter(name: string, points: number, duration: number): RateLimiterRedis {
    const key = `${name}:${points}:${duration}`;
    
    if (!this.limiters.has(key)) {
      this.limiters.set(key, new RateLimiterRedis({
        storeClient: redis,
        keyPrefix: `rl:${name}`,
        points,
        duration,
      }));
    }
    
    return this.limiters.get(key)!;
  }
  
  async checkLimit(
    limiterName: string, 
    identifier: string, 
    points: number, 
    duration: number
  ): Promise<{ allowed: boolean; remaining: number; resetTime: Date }> {
    const limiter = this.getLimiter(limiterName, points, duration);
    const key = identifier;
    
    try {
      const result = await limiter.get(key);
      
      if (result && result.remainingPoints <= 0) {
        return {
          allowed: false,
          remaining: 0,
          resetTime: new Date(Date.now() + result.msBeforeNext),
        };
      }
      
      await limiter.consume(key);
      
      return {
        allowed: true,
        remaining: (result?.remainingPoints ?? points) - 1,
        resetTime: new Date(Date.now() + duration * 1000),
      };
      
    } catch (rejRes) {
      // Rate limit exceeded
      return {
        allowed: false,
        remaining: 0,
        resetTime: new Date(Date.now() + (rejRes.msBeforeNext || 0)),
      };
    }
  }
  
  async reset(limiterName: string, identifier: string, points: number, duration: number): Promise<void> {
    const limiter = this.getLimiter(limiterName, points, duration);
    await limiter.delete(identifier);
  }
}

export const rateLimiter = new RateLimiter();
```

### tRPC Rate Limit Middleware
```typescript
// src/server/trpc/middleware/ratelimit.ts
import { TRPCError } from '@trpc/server';
import { middleware } from '../trpc';
import { rateLimiter } from '@/lib/ratelimit';
import { rateLimitConfig } from '@/lib/ratelimit/config';

interface RateLimitOptions {
  type: 'ip' | 'user';
  endpoint: keyof typeof rateLimitConfig.ip;
  getIdentifier?: (ctx: any) => string;
}

export const rateLimit = (options: RateLimitOptions) => {
  const config = options.type === 'ip' 
    ? rateLimitConfig.ip[options.endpoint]
    : rateLimitConfig.user[options.endpoint as keyof typeof rateLimitConfig.user];
  
  return middleware(async ({ ctx, next, path }) => {
    const identifier = options.getIdentifier 
      ? options.getIdentifier(ctx)
      : options.type === 'ip' 
        ? ctx.ip || 'unknown'
        : ctx.session?.user?.id || 'anonymous';
    
    const result = await rateLimiter.checkLimit(
      `${options.endpoint}:${options.type}`,
      identifier,
      config.points,
      config.duration
    );
    
    if (!result.allowed) {
      const retryAfter = Math.ceil((result.resetTime.getTime() - Date.now()) / 1000);
      
      throw new TRPCError({
        code: 'TOO_MANY_REQUESTS',
        message: `Rate limit exceeded. Please try again in ${Math.ceil(retryAfter / 60)} minutes.`,
      });
    }
    
    return next({
      ctx: {
        ...ctx,
        rateLimit: {
          remaining: result.remaining,
          resetTime: result.resetTime,
        },
      },
    });
  });
};

// Pre-configured rate limiters
export const ipRateLimit = (endpoint: keyof typeof rateLimitConfig.ip) => 
  rateLimit({ type: 'ip', endpoint });

export const userRateLimit = (endpoint: keyof typeof rateLimitConfig.user) => 
  rateLimit({ type: 'user', endpoint });
```

### Combined Lockout Middleware
```typescript
// src/server/trpc/middleware/lockout.ts
import { TRPCError } from '@trpc/server';
import { middleware } from '../trpc';
import { isAccountLocked, getLockoutRemainingMs } from '@/lib/auth/lockout';

export const checkLockout = middleware(async ({ ctx, next }) => {
  // If user is authenticated, check if their account is locked
  if (ctx.session?.user?.id) {
    const locked = await isAccountLocked(ctx.session.user.id);
    
    if (locked) {
      const user = await ctx.db.query.users.findFirst({
        where: eq(users.id, ctx.session.user.id),
        columns: { lockedUntil: true },
      });
      
      if (user?.lockedUntil) {
        const remainingMs = getLockoutRemainingMs(user.lockedUntil);
        const remainingMinutes = Math.ceil(remainingMs / 60000);
        
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: `Account locked. Try again in ${remainingMinutes} minutes.`,
        });
      }
    }
  }
  
  return next({ ctx });
});
```

### Usage in Routers
```typescript
// src/server/trpc/routers/auth.ts
import { rateLimit, ipRateLimit, userRateLimit } from '../middleware/ratelimit';

export const authRouter = router({
  login: publicProcedure
    .use(ipRateLimit('login'))
    .use(userRateLimit('login'))
    .input(loginInputSchema)
    .mutation(async ({ input, ctx }) => {
      // ... login logic from slice-b
    }),
  
  requestPasswordReset: publicProcedure
    .use(ipRateLimit('passwordReset'))
    .input(passwordResetInputSchema)
    .mutation(async ({ input }) => {
      // ... password reset logic
    }),
});
```

## Business Rules & Invariants
1. IP-based rate limiting applies to all clients from same IP
2. User-based rate limiting applies per account (by email/userId)
3. Rate limits are in addition to account lockout (defense in depth)
4. Rate limit errors include retry-after information
5. Rate limit counters reset after duration expires

## Edge Cases
1. **Shared IP (corporate NAT)** — Higher IP limits for login, user limits are primary defense
2. **Distributed attack (many IPs)** — User-based limits catch this
3. **Rate limiter unavailable** — Fail open (allow request) but log error
4. **Clock skew between servers** — Redis TTL handles this automatically
5. **Rate limit reset on success** — Only reset failed login attempts, not general rate limits

## Tests

### src/lib/ratelimit/ratelimit.test.ts
- `should allow requests under limit`: Verifies normal operation
- `should block requests over limit`: Verifies blocking
- `should reset after duration`: Verifies time-based reset
- `should track separate limits per identifier`: Verifies isolation
- `should handle Redis unavailable`: Verifies graceful degradation

### src/server/trpc/middleware/ratelimit.test.ts
- `should apply IP rate limit`: Verifies IP tracking
- `should apply user rate limit`: Verifies user tracking
- `should return correct retry-after`: Verifies header accuracy

## Verification
```bash
npm run test:unit src/lib/ratelimit/ratelimit.test.ts
npm run test:integration src/server/trpc/middleware/ratelimit.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → Security, Account lockout
