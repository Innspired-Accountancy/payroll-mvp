# Story: User and Role Models

**Epic:** epic-05-identity-access
**Priority:** 1 of 8
**Sprint:** Sprint 1
**Dependencies:** none

## Goal

Establish the foundational database schema for identity and access management by creating User, Role, UserRoleAssignment, and Permission entities with Drizzle ORM. This story delivers the data layer that all subsequent authentication and authorization features depend on.

## Acceptance Criteria

- [ ] User table created with all fields (id, email, password_hash, first_name, last_name, user_type, bureau_id, employer_id, employee_id, status, mfa_enabled, mfa_secret, last_login, password_changed_at, failed_login_attempts, locked_until)
- [ ] Role table created with system and custom role support
- [ ] UserRoleAssignment table created for many-to-many user-role relationships with scoping
- [ ] Permission table created with predefined permission codes
- [ ] All Drizzle ORM schemas properly typed with TypeScript
- [ ] Database migrations generated and tested
- [ ] Seed data for system roles (Platform Admin, Bureau Owner, Payroll Manager, etc.)
- [ ] All tables have proper indexes for query performance
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Schema changes needed after downstream stories start | Medium | Medium | Lock schema after review; version migrations properly |
| UUID vs auto-increment ID conflicts | Low | High | Follow project convention; use UUID consistently |
| JSONB permissions field performance issues | Low | Medium | Index JSONB fields; monitor query performance |
| Soft delete vs hard delete decision affects relationships | Medium | Medium | Define policy in this story; use status field consistently |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | User entity schema and migrations | S | none |
| slice-b | Role and Permission entities | S | slice-a |
| slice-c | UserRoleAssignment with scoping | S | slice-b |
| slice-d | Seed data and indexes | XS | slice-c |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — User table schema with Drizzle ORM
- [slice-b.md](./slice-b.md) — Role and Permission tables
- [slice-c.md](./slice-c.md) — UserRoleAssignment with multi-tenant scoping
- [slice-d.md](./slice-d.md) — Seed data and database indexes
