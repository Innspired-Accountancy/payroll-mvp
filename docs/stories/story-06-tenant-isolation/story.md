# Story: Tenant Isolation

**Epic:** epic-05-identity-access
**Priority:** 6 of 8
**Sprint:** Sprint 2
**Dependencies:** story-05-rbac

## Goal

Implement multi-tenant data isolation ensuring users can only access data within their assigned tenant scope (bureau, employer, or employee). Enforce tenant filtering at the database query level for all data access operations.

## Acceptance Criteria

- [ ] Tenant context extraction from JWT token on each request
- [ ] Automatic tenant filtering applied to all database queries
- [ ] Row-level security policies for PostgreSQL (RLS) as defense in depth
- [ ] Cross-tenant access prevention verified in all queries
- [ ] Tenant switch validation (users with multi-tenant access)
- [ ] Tenant-aware query builder helpers for Drizzle ORM
- [ ] Tenant context middleware for tRPC context
- [ ] Audit logging of tenant context for all data access
- [ ] Error handling for unauthorized tenant access attempts
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Missing tenant filter in manual SQL | Medium | Critical | Code review checklist; lint rules; RLS as backup |
| Tenant context not set in async contexts | Medium | High | Use AsyncLocalStorage; validate context exists |
| Performance impact of tenant filtering | Low | Medium | Benchmark; add indexes on tenant_id columns |
| RLS policy maintenance overhead | Medium | Medium | Generate policies from schema; test coverage |
| Tenant identification confusion (bureau vs employer) | Medium | High | Clear naming; type safety; validation layers |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Tenant context middleware | M | story-05-rbac |
| slice-b | Database query tenant filtering | M | slice-a |
| slice-c | PostgreSQL RLS policies | M | slice-b |
| slice-d | Tenant switch and validation | S | slice-a |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Tenant context extraction middleware
- [slice-b.md](./slice-b.md) — Query-level tenant filtering
- [slice-c.md](./slice-c.md) — PostgreSQL RLS policies
- [slice-d.md](./slice-d.md) — Multi-tenant user switching
