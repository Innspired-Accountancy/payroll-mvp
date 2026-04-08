# Story: Leave Balance and History View

**Epic:** epic-06-employee-portal
**Priority:** 5 of 8
**Sprint:** Month 4
**Dependencies:** story-18-portal-auth (authentication), Leave Module (balance calculation)

---

## Goal

Enable employees to view their current leave balances and historical leave requests through the portal. Employees can see annual leave entitlement, days taken, days remaining, and a history of past and upcoming leave requests with their approval status.

---

## Acceptance Criteria

- [ ] Employee sees current leave year balance summary (annual leave, sick leave, other types)
- [ ] Balance shows: entitlement, taken, booked (pending), remaining
- [ ] Leave history table shows all requests with dates, type, days, status
- [ ] History is sortable by date (default: newest first)
- [ ] Status values: pending, approved, rejected, cancelled
- [ ] Clicking a request shows full details including approver notes
- [ ] Color-coded status indicators (green=approved, yellow=pending, red=rejected)
- [ ] Mobile-responsive card view for mobile screens
- [ ] Real-time balance calculation from Leave Module
- [ ] Strict data isolation - only employee's own leave data visible
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Leave balance accuracy | Medium | High | Real-time calculation from Leave Module API, not cached values |
| Accrual complexity | Medium | Medium | Display breakdown showing opening balance + accruals - taken |
| Multiple leave types | Low | Low | Tabbed or expandable sections for each leave type |
| Carry-over calculation | Medium | Medium | Clear display of carry-over days separate from current year entitlement |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Leave balance summary API and UI | M | story-18-portal-auth, Leave Module |
| slice-b | Leave request history list | M | slice-a |
| slice-c | Leave request detail view | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Leave balance summary with real-time calculation
- [slice-b.md](./slice-b.md) — Leave request history list with status filtering
- [slice-c.md](./slice-c.md) — Leave request detail view with approver notes
