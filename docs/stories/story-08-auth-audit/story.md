# Story: Authentication Audit Logging

**Epic:** epic-05-identity-access
**Priority:** 8 of 8
**Sprint:** Sprint 3
**Dependencies:** story-02-password-auth

## Goal

Implement comprehensive audit logging for all authentication events including successful/failed logins, password resets, MFA events, permission changes, and session activities. Ensure compliance requirements are met with immutable, queryable audit records.

## Acceptance Criteria

- [ ] Audit log table schema (id, timestamp, user_id, event_type, ip_address, user_agent, details, success)
- [ ] Login success/failure logging with reason codes
- [ ] Password reset request/completion logging
- [ ] MFA setup/enable/disable/verification logging
- [ ] Permission grant/revoke logging with actor and target
- [ ] Session create/terminate/timeout logging
- [ ] Audit log query endpoints for admin (with filtering, pagination)
- [ ] Audit log retention policy (90 days hot, 1 year cold archive)
- [ ] Tamper-evident audit log (hash chain or digital signatures)
- [ ] Export capability (CSV/JSON) for compliance reports
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Audit log volume overwhelming database | Medium | High | Async logging; batch inserts; separate audit database |
| PII in audit logs compliance issues | Medium | High | Exclude sensitive data; hash identifiers; retention policy |
| Audit log tampering by privileged users | Low | Critical | Immutable storage; separate audit service; WORM storage |
| Performance impact of synchronous audit logging | Medium | Medium | Async logging queue; fire-and-forget pattern |
| Missing critical security events | Medium | High | Comprehensive event catalog; code review checklist |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Audit log schema and storage | S | story-02-password-auth |
| slice-b | Authentication event logging | S | slice-a |
| slice-c | Permission change logging | S | slice-a |
| slice-d | Audit log query and export | M | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Audit log database schema
- [slice-b.md](./slice-b.md) — Authentication event logging
- [slice-c.md](./slice-c.md) — Permission and session logging
- [slice-d.md](./slice-d.md) — Audit log query and export endpoints
