# Slice a: BetterAuth Setup and Configuration

**Story:** story-02-password-auth
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** story-01-user-model

## Goal

Configure BetterAuth with PostgreSQL adapter, integrate with the existing User model from story-01, and set up the authentication provider for Next.js 14 with tRPC.

## Decision Checklist

- [x] All libraries/packages named: `better-auth@0.8.x`, `@better-auth/expo` (if needed), `postgres@latest`
- [x] SDK methods identified: `betterAuth()`, `pgAdapter()`, `nextCookies()`
- [x] External service endpoints: N/A (internal auth)
- [x] Data contracts defined: BetterAuth configuration and callbacks
- [x] Configuration variables: `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `DATABASE_URL`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ API Contracts → POST /api/v1/auth/login
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Security

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/auth/index.ts` | create | BetterAuth configuration |
| `src/lib/auth/client.ts` | create | Client-side auth utilities |
| `src/lib/auth/hooks.ts` | create | Auth hooks for React |
| `src/app/api/auth/[...all]/route.ts` | create | BetterAuth API route handler |
| `src/middleware.ts` | create/update | Next.js auth middleware |
| `.env.example` | update | Auth environment variables |

## Responsibilities
1. Initialize BetterAuth with PostgreSQL adapter
2. Map BetterAuth's user model to our existing User schema
3. Configure session management (cookies, expiry)
4. Set up password hashing (Argon2id via BetterAuth default)
5. Create Next.js middleware for route protection

## Contracts

### BetterAuth Configuration
```typescript
import { betterAuth } from 'better-auth';
import { pgAdapter } from 'better-auth/adapters/pg';
import { nextCookies } from 'better-auth/next-js';
import postgres from 'postgres';

const sql = postgres(process.env.DATABASE_URL!);

export const auth = betterAuth({
  database: pgAdapter(sql, {
    // Use our existing tables
    userTableName: 'users',
    accountTableName: 'accounts',
    sessionTableName: 'sessions',
    verificationTokenTableName: 'verification_tokens',
  }),
  
  secret: process.env.BETTER_AUTH_SECRET!,
  
  // URL configuration
  baseURL: process.env.BETTER_AUTH_URL!,
  
  // Cookie configuration
  cookies: {
    sessionToken: {
      name: '__session',
      options: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        maxAge: 60 * 60 * 24 * 7, // 7 days
      },
    },
  },
  
  // Session configuration
  session: {
    expiresIn: 60 * 60 * 24 * 7, // 7 days
    updateAge: 60 * 60 * 24, // Update session every 24 hours
  },
  
  // Email/password configuration
  emailAndPassword: {
    enabled: true,
    autoSignIn: true,
    minPasswordLength: 12,
    maxPasswordLength: 128,
    sendResetPasswordEmail: async (user, url) => {
      // Handled in slice-c
      await sendPasswordResetEmail(user.email, url);
    },
  },
  
  // Callbacks for integrating with our User model
  callbacks: {
    async signIn(user, account, profile) {
      // Update last_login timestamp
      await updateUserLastLogin(user.id);
      return true;
    },
    
    async session(session, user) {
      // Enrich session with our custom user fields
      const userData = await getUserWithRoles(user.id);
      return {
        ...session,
        user: {
          ...session.user,
          userType: userData.userType,
          bureauId: userData.bureauId,
          employerId: userData.employerId,
          roles: userData.roles,
        },
      };
    },
  },
  
  plugins: [nextCookies()],
});

export type AuthSession = typeof auth.$Infer.Session;
```

### Next.js API Route Handler
```typescript
// src/app/api/auth/[...all]/route.ts
import { auth } from '@/lib/auth';

const handler = auth.handler;

export { handler as GET, handler as POST };
```

### Next.js Middleware
```typescript
// src/middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { auth } from '@/lib/auth';

const publicRoutes = ['/login', '/register', '/forgot-password', '/reset-password'];
const apiAuthRoutes = ['/api/auth'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  
  // Allow public routes
  if (publicRoutes.some(route => pathname.startsWith(route))) {
    return NextResponse.next();
  }
  
  // Allow auth API routes
  if (apiAuthRoutes.some(route => pathname.startsWith(route))) {
    return NextResponse.next();
  }
  
  // Check session
  const session = await auth.api.getSession(request);
  
  if (!session) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', pathname);
    return NextResponse.redirect(loginUrl);
  }
  
  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|public).*)'],
};
```

### Environment Variables
```bash
# .env.example
BETTER_AUTH_SECRET="your-secret-key-min-32-chars-long"
BETTER_AUTH_URL="http://localhost:3000"
DATABASE_URL="postgresql://user:pass@localhost:5432/payroll"
```

## Business Rules & Invariants
1. BetterAuth must use our existing users table (not create its own)
2. Password minimum length is 12 characters (enforced in config and DB)
3. Session cookies are httpOnly and secure in production
4. All authentication routes are prefixed with /api/auth

## Edge Cases
1. **Database connection failure on startup** — BetterAuth retries; log and alert
2. **Missing environment variables** — Throw at startup with clear error
3. **Schema mismatch between BetterAuth and our User table** — Map fields in adapter config
4. **Session cookie too large** — BetterAuth compresses; monitor size

## Tests

### src/lib/auth/index.test.ts
- `should initialize BetterAuth without errors`: Verifies config valid
- `should use PostgreSQL adapter`: Verifies database connection
- `should have correct cookie settings`: Verifies security config
- `should enforce minimum password length`: Verifies validation

## Verification
```bash
npm run build
npm run test:unit src/lib/auth/index.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § API Contracts → Authentication endpoints
- 02-09-identity-access-spec.md § Non-Functional Requirements → Security section
