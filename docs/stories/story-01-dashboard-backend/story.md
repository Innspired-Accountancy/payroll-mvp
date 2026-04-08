# Story: Dashboard Backend - Status Aggregation API

**Epic:** epic-04-bureau-operations
**Priority:** 1 of 8
**Sprint:** Month 4
**Dependencies:** epic-01-core-payroll, epic-02-hmrc-submissions, epic-03-pensions

---

## Goal

Build the backend API that aggregates payroll status across all clients for bureau dashboard views. This provides real-time visibility into client portfolio status, upcoming deadlines, exceptions, and workload distribution. This is the foundation for all bureau operations features.

---

## Acceptance Criteria

- [ ] Dashboard summary API returns aggregated counts (total clients, payrolls this week, overdue items, critical exceptions)
- [ ] Client list API supports filtering by status, assigned processor, and overdue flag
- [ ] Status breakdown endpoint returns counts per payroll state (not_started through complete)
- [ ] Upcoming deadlines endpoint returns deadlines with days remaining and urgency status
- [ ] Exceptions list endpoint returns active exceptions with severity and assignment info
- [ ] All endpoints enforce bureau-level tenant isolation
- [ ] API responses complete within 500ms for 1000 clients
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Performance degradation with large client portfolios | Medium | High | Database indexing, query optimization, implement caching for summary data |
| Stale status data from event integration failures | Medium | Medium | Retry logic, dead letter queue, manual refresh endpoint |
| Race conditions in concurrent status updates | Low | High | Optimistic locking, atomic updates, proper transaction boundaries |
| Data aggregation accuracy across multiple tables | Medium | High | Comprehensive test suite, data consistency checks, audit logging |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Client payroll status data model and views | M | none |
| slice-b | Dashboard aggregation API endpoints | M | slice-a |
| slice-c | Real-time status update events | M | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Client payroll status data model and database views
- [slice-b.md](./slice-b.md) — Dashboard aggregation tRPC procedures
- [slice-c.md](./slice-c.md) — Real-time status updates via events
