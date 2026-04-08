# Epic: Identity & Access Management

**Date:** 2026-04-08
**Sprint(s):** Month 1 (foundation)
**Dependencies:** None (required by all other epics)

---

## Scope & Deliverables

RBAC system with multi-tenant scoping, MFA, JWT authentication, and segregation of duties enforcement.

### In Scope
- User management (invite, activate, deactivate)
- Role definition and assignment
- JWT authentication
- MFA (TOTP)
- Permission enforcement
- Session management
- Audit logging of auth events

### Out of Scope
- SSO/SAML (phase 3)
- Biometric auth (phase 3)

---

## Decisions

### Libraries & Packages

| Package | Version | Rationale | License |
|---------|---------|-----------|---------|
| python-jose | 3.3.x | JWT encoding/decoding | MIT |
| passlib | 1.7.x | Password hashing (bcrypt) | BSD |
| pyotp | 2.9.x | TOTP MFA | MIT |
| qrcode | 7.x | QR code for MFA setup | BSD |
| slowapi | 0.1.x | Rate limiting | MIT |

### Auth Strategy

| Aspect | Decision | Details |
|--------|----------|---------|
| Tokens | JWT | Access (15min) + Refresh (7d) |
| Storage | HttpOnly cookies | XSS protection |
| MFA | TOTP | Google Authenticator compatible |
| Rate limiting | 5 attempts/15min | Account lockout |

### RBAC Model

| Element | Implementation |
|---------|----------------|
| Roles | Predefined + custom per bureau |
| Permissions | Granular (resource:action) |
| Scoping | bureau/employer/employee level |
| Enforcement | Middleware + decorators |

---

## Build Order (Story Sequence)

| # | Story | Description | Effort | Dependencies |
|---|-------|-------------|--------|--------------|
| 1 | story-01-user-model | User, role models | S | None |
| 2 | story-02-password-auth | Login, password reset | M | story-01 |
| 3 | story-03-jwt-implementation | Token generation, validation | M | story-02 |
| 4 | story-04-mfa | TOTP setup, verification | M | story-03 |
| 5 | story-05-rbac | Permission system | L | story-01 |
| 6 | story-06-tenant-isolation | Multi-tenant scoping | M | story-05 |
| 7 | story-07-session-mgmt | Session tracking, logout | S | story-03 |
| 8 | story-08-auth-audit | Login/logout logging | S | story-02 |

---

## Decision Completeness Checklist

- [x] All third-party libraries named with versions
- [x] Token strategy defined (JWT, 15min/7d)
- [x] MFA approach defined (TOTP)
- [x] RBAC model specified
- [x] Rate limiting defined
- [x] No "TBD", slash-notation, or placeholder text remaining
