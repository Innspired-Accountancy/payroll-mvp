# Story: Client Portal Authentication

**Epic:** epic-07-employer-portal
**Priority:** 1 of 8
**Sprint:** Sprint 4
**Dependencies:** story-05-rbac, story-04-mfa

## Goal

Implement secure authentication for client portal users (employer admins and managers) with role-based access, MFA enforcement, and employer-scoped data visibility. Client users must only access their own employer's data and operations.

## Acceptance Criteria

- [ ] Client portal login page with email/password
- [ ] MFA verification required for all client users (TOTP/SMS)
- [ ] Role detection and redirection (admin/manager/viewer)
- [ ] Employer-scoped session enforcement (tenant isolation)
- [ ] Password reset flow for client users
- [ ] Account lockout after 5 failed attempts
- [ ] Session timeout after 30 minutes idle
- [ ] Secure cookie configuration (httpOnly, secure, sameSite)
- [ ] Audit logging of all login attempts and sessions
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Session hijacking across employers | Low | High | Strict employer_id scoping in JWT; validate on every request |
| MFA bypass attacks | Low | Critical | Enforce MFA server-side; no client-side bypass |
| Brute force password attacks | Medium | Medium | Rate limiting; account lockout; CAPTCHA after failures |
| Session fixation | Low | High | Regenerate session ID on login; secure cookie attributes |
| Stolen device with remembered session | Medium | Medium | Short session lifetime; require re-auth for sensitive actions |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Client user login and MFA flow | M | story-05-rbac, story-04-mfa |
| slice-b | Session management and employer scoping | M | slice-a |
| slice-c | Password reset and account security | S | slice-a |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Client portal login with MFA verification
- [slice-b.md](./slice-b.md) — Session management with employer scoping
- [slice-c.md](./slice-c.md) — Password reset and security features
