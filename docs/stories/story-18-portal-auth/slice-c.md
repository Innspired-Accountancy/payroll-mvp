# Slice c: Session Management and Security Middleware

**Story:** story-18-portal-auth
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** slice-a (login page and BetterAuth integration)

---

## Goal

Implement secure session management with 30-minute idle timeout, secure cookie settings, and middleware to enforce authentication on all employee portal routes. This slice ensures the security foundation for the entire employee portal.

---

## Decision Checklist

- [x] All libraries/packages named: BetterAuth 0.5.x, iron-session 8.0.x
- [x] SDK methods/API calls identified: betterAuth.api.getSession(), betterAuth.api.signOut()
- [x] External service endpoints: None
- [x] Data contracts defined: EmployeeSession, SessionContext
- [x] Configuration variables: SESSION_MAX_AGE=1800, SESSION_IDLE_TIMEOUT=1800, COOKIE_SECURE=true
- [x] Error scenarios identified: Session expired, session hijacking attempt, CSRF violation
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-09-identity-access-spec.md — Session timeout, secure cookies
- 02-05-employee-portal-spec.md — Session timeout after 30 minutes
- 08-architecture-and-patterns.md — Session management patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/middleware.ts` | update | Route protection and session validation |
| `src/lib/auth/session.ts` | create | Session utilities and context |
| `src/server/trpc.ts` | update | Add employee session context |
| `src/hooks/use-session.ts` | create | React hook for session management |
| `src/app/(employee)/logout/page.tsx` | create | Logout handler |

---

## Responsibilities

1. Enforce authentication on all /employee/* routes via middleware
2. Implement 30-minute idle session timeout with sliding refresh
3. Set secure cookie attributes (HttpOnly, Secure, SameSite=strict)
4. Validate session on every request via tRPC context
5. Provide logout functionality that clears session
6. Log all session events (login, logout, expiry) with IP address

---

## Contracts

### SessionContext
```typescript
interface EmployeeSessionContext {
  session: {
    id: string;
    userId: string;
    employeeId: string;
    expiresAt: Date;
    ipAddress: string;
    userAgent: string;
    mfaVerified: boolean;
  };
  user: {
    id: string;
    email: string;
    userType: "employee";
    permissions: ["payslip:view_own", "leave:request", "profile:edit_own"];
  };
}
```

### auth.getSession
- **Method:** tRPC query `auth.getSession`
- **Input:** None
- **Output:**
  ```typescript
  {
    user: {
      id: string;
      email: string;
      employeeId: string;
    };
    expiresAt: string;        // ISO date
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — No valid session
- **Auth:** Protected procedure (valid session required)

### auth.signOut
- **Method:** tRPC mutation `auth.signOut`
- **Input:** None
- **Output:**
  ```typescript
  {
    success: boolean;
  }
  ```
- **Auth:** Protected procedure

---

## Business Rules & Invariants

1. Session expires after 30 minutes of inactivity (no requests)
2. Session cookie is HttpOnly (not accessible via JavaScript)
3. Session cookie uses Secure flag in production (HTTPS only)
4. SameSite=strict prevents CSRF attacks
5. Each session is bound to IP address (optional: detect changes and require re-auth)
6. Session ID is rotated periodically for security
7. All session access logged with IP, timestamp, and user agent

---

## Edge Cases

1. **Session expires during active use** — Redirect to login with return URL
2. **Session cookie tampering** — Invalidate session, redirect to login
3. **Concurrent sessions from multiple devices** — Allow, track separately
4. **Clock skew on client** — Server-side expiration check only
5. **Logout from one device** — Only invalidate that session, others remain active

---

## Tests

### middleware.test.ts
- Unauthenticated request to /employee/* redirects to login
- Authenticated request passes through
- Expired session triggers redirect
- API routes return 401 for invalid session

### use-session.test.ts
- Hook returns session data when authenticated
- Hook returns null when unauthenticated
- Hook updates on session change
- Idle timeout warning at 25 minutes

### auth.router.test.ts
- getSession returns current session
- signOut invalidates session
- Session logging captures all events

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/middleware.test.ts
npx vitest run src/hooks/use-session.test.ts
npx vitest run src/server/routers/auth.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Security → Session timeout after 30 minutes
- 02-09-identity-access-spec.md § Security → Session timeout: 30 minutes, secure cookies
- 02-05-employee-portal-spec.md § Security → Session timeout after 30 minutes inactivity
