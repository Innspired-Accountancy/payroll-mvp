# Story: Portal Authentication

**Epic:** epic-06-employee-portal
**Priority:** 1 of 8
**Sprint:** Month 4
**Dependencies:** epic-05-identity-access (user authentication, MFA infrastructure)

---

## Goal

Enable employees to securely log into the employee portal using email/password with mandatory MFA (TOTP or SMS). This is the entry point for all employee self-service features and must enforce strict self-only data visibility from the first authentication step.

---

## Acceptance Criteria

- [ ] Employee can log in with email and password
- [ ] MFA is required for all employee logins (no bypass option)
- [ ] Support TOTP-based MFA using authenticator apps (Google Authenticator, Authy)
- [ ] Session timeout after 30 minutes of inactivity
- [ ] Failed login attempts lock account after 5 failures (15-minute lockout)
- [ ] All login events logged with IP address and timestamp
- [ ] Session tokens are scoped to employee-only permissions (payslip:view_own, leave:request, profile:edit_own)
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Employee confusion with MFA setup | High | Medium | Clear setup wizard with QR code scan instructions, fallback to SMS |
| Lost MFA device recovery | Medium | High | Secure recovery via employer admin verification, not email reset |
| Session fixation attacks | Low | High | Rotate session ID after auth, secure cookie settings (HttpOnly, SameSite) |
| Brute force credential attacks | Medium | High | Rate limiting, account lockout, CAPTCHA after 3 failures |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Employee login page and BetterAuth integration | M | epic-05-identity-access |
| slice-b | MFA setup and verification flow | M | slice-a |
| slice-c | Session management and security middleware | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Employee login page with BetterAuth credentials
- [slice-b.md](./slice-b.md) — MFA setup and TOTP verification flow
- [slice-c.md](./slice-c.md) — Session management, timeout, and security middleware
