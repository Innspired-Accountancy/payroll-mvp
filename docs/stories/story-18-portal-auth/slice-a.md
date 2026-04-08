# Slice a: Employee Login Page and BetterAuth Integration

**Story:** story-18-portal-auth
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** epic-05-identity-access (user authentication infrastructure)

---

## Goal

Create the employee login page and integrate with BetterAuth for credential-based authentication. This slice establishes the authentication entry point for all employee portal users with proper form validation and error handling.

---

## Decision Checklist

- [x] All libraries/packages named: BetterAuth 0.5.x, React Hook Form 7.51.x, Zod 3.22.x
- [x] SDK methods/API calls identified: betterAuth.api.signInEmail(), betterAuth.api.signUpEmail()
- [x] External service endpoints: None (internal tRPC/BetterAuth)
- [x] Data contracts defined: EmployeeLoginInput Zod schema, Session context type
- [x] Configuration variables: BETTER_AUTH_SECRET, BETTER_AUTH_URL, SESSION_MAX_AGE=1800
- [x] Error scenarios identified: Invalid credentials, account locked, unverified email
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-09-identity-access-spec.md — Authentication section, MFA requirements
- 02-05-employee-portal-spec.md — Portal session requirements
- 08-architecture-and-patterns.md — BetterAuth integration pattern

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(employee)/login/page.tsx` | create | Employee login page component |
| `src/app/(employee)/layout.tsx` | create | Employee portal layout with auth check |
| `src/components/auth/employee-login-form.tsx` | create | Login form with validation |
| `src/lib/validation/auth.ts` | update | Add employee login schemas |
| `src/server/routers/auth.ts` | update | Add employee-specific auth procedures |
| `src/middleware.ts` | update | Add employee portal route protection |

---

## Responsibilities

1. Render employee login form with email and password fields
2. Integrate with BetterAuth credential provider for authentication
3. Validate input using Zod schemas
4. Handle authentication errors with user-friendly messages
5. Redirect to MFA setup/verification upon successful credential validation
6. Set secure session cookies with HttpOnly, SameSite=strict

---

## Contracts

### betterAuth.api.signInEmail
- **Method:** `POST /api/auth/sign-in/email` (BetterAuth internal)
- **SDK:** `betterAuthClient.signIn.email({ email, password, callbackURL })`
- **Input:**
  ```typescript
  {
    email: string;      // Valid email format
    password: string;   // Min 12 characters
    callbackURL?: string; // "/employee/dashboard"
  }
  ```
- **Output:**
  ```typescript
  {
    token: string;      // JWT access token
    user: {
      id: string;
      email: string;
      userType: "employee";
      employeeId: string;
      mfaEnabled: boolean;
    }
  }
  ```
- **Errors:**
  - `INVALID_CREDENTIALS` — Email or password incorrect
  - `ACCOUNT_LOCKED` — Account locked due to failed attempts
  - `EMAIL_NOT_VERIFIED` — Email verification required
- **Auth:** None (public endpoint for login)

### EmployeeLoginInput Schema
```typescript
export const employeeLoginInput = z.object({
  email: z.string().email("Enter a valid email address"),
  password: z.string().min(12, "Password must be at least 12 characters"),
});
```

---

## Business Rules & Invariants

1. Only users with `user_type = "employee"` can access employee portal
2. Password must be minimum 12 characters (enforced by BetterAuth)
3. Session cookie must be HttpOnly, Secure (production), SameSite=strict
4. Failed login attempts increment counter (tracked by BetterAuth)
5. Account locks after 5 consecutive failures for 15 minutes

---

## Edge Cases

1. **Employee tries to login with bureau email** — Return "Invalid credentials" (don't reveal user exists)
2. **Account locked** — Show lockout message with time remaining, prevent login attempts
3. **First-time login** — Redirect to password setup if using invitation flow
4. **Session expired mid-action** — Redirect to login with return URL preserved
5. **Concurrent login from different devices** — Allow, track both sessions independently

---

## Tests

### employee-login-form.test.tsx
- Render login form with email and password fields
- Submit with valid credentials calls signIn.email
- Submit with invalid email shows validation error
- Submit with short password shows validation error
- API error displays user-friendly message
- Loading state during submission

### auth.router.test.ts
- signIn.email succeeds with valid employee credentials
- signIn.email fails with invalid credentials
- signIn.email fails when account locked
- Session context contains employeeId and userType

---

## Verification

```bash
# Type checking
cd /Users/josephstephenson-mouzo/Projects/03\ -\ development/16\ -\ payroll\ mvp
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/components/auth/employee-login-form.test.tsx
npx vitest run src/server/routers/auth.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Authentication → Employee login flow
- 02-09-identity-access-spec.md § Journey 1 → Bureau User Login (adapted for employees)
- 02-05-employee-portal-spec.md § Security → MFA required, session timeout
