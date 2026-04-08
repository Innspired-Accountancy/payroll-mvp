# Story: Password Authentication

**Epic:** epic-05-identity-access
**Priority:** 2 of 8
**Sprint:** Sprint 1
**Dependencies:** story-01-user-model

## Goal

Implement secure password-based authentication using BetterAuth, including user login, password hashing with bcrypt/Argon2, password reset via email, and account lockout after failed attempts. This enables users to authenticate with email/password credentials.

## Acceptance Criteria

- [ ] BetterAuth configured with PostgreSQL adapter
- [ ] Login endpoint functional with email/password validation
- [ ] Password hashing using Argon2id (minimum 12 character passwords)
- [ ] Password reset flow with secure token (email link, 1-hour expiry)
- [ ] Account lockout after 5 failed login attempts (30-minute lockout)
- [ ] Failed login attempt counter reset on successful login
- [ ] Secure session cookie configuration (httpOnly, secure, sameSite)
- [ ] Input validation for all authentication endpoints
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| BetterAuth integration complexity with custom User model | Medium | High | Review BetterAuth schema customization docs; test early |
| Email delivery failures for password reset | Medium | Medium | Implement email queue with retry; add admin override |
| Timing attacks on login endpoint | Low | High | Use constant-time comparison; uniform response times |
| Password reset token exposure in logs | Low | High | Never log tokens; hash in database |
| Rate limiting bypass | Medium | High | Implement IP-based + user-based rate limiting |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | BetterAuth setup and configuration | M | story-01-user-model |
| slice-b | Login endpoint and validation | M | slice-a |
| slice-c | Password reset flow | M | slice-b |
| slice-d | Account lockout and rate limiting | S | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — BetterAuth configuration with PostgreSQL
- [slice-b.md](./slice-b.md) — Login endpoint and password validation
- [slice-c.md](./slice-c.md) — Password reset with email tokens
- [slice-d.md](./slice-d.md) — Account lockout and rate limiting
