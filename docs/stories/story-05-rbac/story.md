# Story: Role-Based Access Control (RBAC)

**Epic:** epic-05-identity-access
**Priority:** 5 of 8
**Sprint:** Sprint 2
**Dependencies:** story-01-user-model

## Goal

Implement a comprehensive RBAC system with permission definitions, role management, user-role assignments, and permission checking utilities. Support both system-defined roles and custom bureau-specific roles with fine-grained permission scoping.

## Acceptance Criteria

- [ ] Permission registry with all system permissions defined
- [ ] Role CRUD endpoints (create, read, update, delete custom roles)
- [ ] User-role assignment endpoints with scope specification
- [ ] Permission checking utilities for route protection
- [ ] Permission middleware for tRPC procedures
- [ ] Role hierarchy support (inheritance where applicable)
- [ ] Segregation of duties validation (prevent conflicting roles)
- [ ] Permission cache for performance (Redis, 5-minute TTL)
- [ ] Admin UI endpoints for role management
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Permission cache stale data | Medium | High | Short TTL; cache invalidation on role change |
| Role hierarchy complexity | Medium | Medium | Keep hierarchy simple (max 2 levels); document clearly |
| Permission enumeration attacks | Low | Medium | Return 403 without revealing which permission missing |
| Segregation of duties edge cases | Medium | High | Comprehensive SoD matrix; audit all combinations |
| Performance on permission checks | Medium | Medium | Benchmark; optimize queries; use materialized views |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Permission registry and definitions | M | story-01-user-model |
| slice-b | Role management CRUD | M | slice-a |
| slice-c | User-role assignment | M | slice-b |
| slice-d | Permission checking middleware | L | slice-a |
| slice-e | Segregation of duties | M | slice-d |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Permission registry with all system permissions
- [slice-b.md](./slice-b.md) — Role management CRUD endpoints
- [slice-c.md](./slice-c.md) — User-role assignment with scoping
- [slice-d.md](./slice-d.md) — Permission checking middleware for tRPC
- [slice-e.md](./slice-e.md) — Segregation of duties enforcement
