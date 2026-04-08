# Story: Multi-Factor Authentication (MFA)

**Epic:** epic-05-identity-access
**Priority:** 4 of 8
**Sprint:** Sprint 2
**Dependencies:** story-03-jwt-implementation

## Goal

Implement TOTP-based MFA for bureau users with setup flow, QR code generation, verification, and enforcement policies. This adds a critical security layer requiring users to provide a time-based one-time password in addition to their credentials.

## Acceptance Criteria

- [ ] TOTP secret generation using RFC 6238 standard
- [ ] MFA setup flow with QR code display (compatible with Google Authenticator, Authy)
- [ ] Backup codes generation (10 codes) for account recovery
- [ ] MFA verification endpoint for login completion
- [ ] MFA required for all bureau staff users (enforced policy)
- [ ] MFA optional for client portal users (configurable)
- [ ] MFA can be disabled by admin or self-service with verification
- [ ] Encrypted storage of TOTP secrets (AES-256)
- [ ] Rate limiting on MFA verification attempts (5 attempts per 15 minutes)
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| TOTP secret encryption key exposure | Low | Critical | Use KMS or environment secret; rotate keys regularly |
| Time drift causing TOTP failures | Medium | Medium | Allow 1-step time drift tolerance; sync server time |
| Users losing access without backup codes | Medium | High | Require backup code download; admin override process |
| QR code interception during setup | Low | High | HTTPS only; short expiry on setup tokens |
| MFA fatigue attacks | Low | Medium | Rate limiting; push notification limits |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | TOTP generation and encryption | M | story-03-jwt-implementation |
| slice-b | MFA setup and QR code | M | slice-a |
| slice-c | MFA verification endpoint | M | slice-a |
| slice-d | Backup codes and recovery | S | slice-c |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — TOTP secret generation and encryption
- [slice-b.md](./slice-b.md) — MFA setup flow with QR codes
- [slice-c.md](./slice-c.md) — MFA verification endpoint
- [slice-d.md](./slice-d.md) — Backup codes and recovery flow
