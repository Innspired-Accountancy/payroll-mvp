# Story: JWT Implementation

**Epic:** epic-05-identity-access
**Priority:** 3 of 8
**Sprint:** Sprint 2
**Dependencies:** story-02-password-auth

## Goal

Implement JWT token generation and validation for stateless authentication. Deliver access tokens (short-lived, 1 hour) and refresh tokens (long-lived, 7 days) with secure storage, rotation, and proper claims including user roles and tenant scope.

## Acceptance Criteria

- [ ] JWT access token generation with proper claims (sub, exp, iat, roles, tenant_id)
- [ ] JWT refresh token generation with rotation support
- [ ] Token validation middleware for protected routes
- [ ] Access token expiry: 1 hour (3600 seconds)
- [ ] Refresh token expiry: 7 days (604800 seconds)
- [ ] Refresh token rotation on use (invalidate old, issue new)
- [ ] Secure token storage (httpOnly cookies)
- [ ] Token blacklisting for logout support
- [ ] All protected routes require valid access token
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Token secret key exposure | Low | Critical | Use environment variables; never commit secrets |
| JWT algorithm confusion attack | Low | Critical | Explicitly specify algorithm (RS256 or HS256); validate alg header |
| Token size bloat from large role claims | Medium | Medium | Use role references not full permissions; compress if needed |
| Clock skew causing token validation failures | Medium | Low | Add leeway (60 seconds) to exp/nbf validation |
| Refresh token theft detection gaps | Medium | High | Bind tokens to device/session fingerprint; detect reuse |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | JWT token generation and claims | M | story-02-password-auth |
| slice-b | Token validation middleware | M | slice-a |
| slice-c | Refresh token rotation | M | slice-b |
| slice-d | Token blacklisting | S | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — JWT generation with claims structure
- [slice-b.md](./slice-b.md) — Token validation middleware
- [slice-c.md](./slice-c.md) — Refresh token rotation strategy
- [slice-d.md](./slice-d.md) — Token blacklisting for logout
