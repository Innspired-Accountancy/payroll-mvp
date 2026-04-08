# Story: Batch Operations - Multi-Client Batch Actions

**Epic:** epic-04-bureau-operations
**Priority:** 8 of 8
**Sprint:** Month 4
**Dependencies:** story-02-dashboard-ui

---

## Goal

Enable bureau staff to perform actions on multiple clients simultaneously to improve efficiency. Support bulk reassignment of payrolls, batch sending of reminders, mass status updates, and bulk exports. Include confirmation dialogs and progress tracking for long-running operations.

---

## Acceptance Criteria

- [ ] Batch selection supports multi-select in client table with select-all option
- [ ] Bulk reassign allows changing assigned processor for multiple clients
- [ ] Batch reminder sends chasing emails to selected clients
- [ ] Bulk status update changes payroll status for selected clients
- [ ] Batch export generates CSV/Excel with selected client data
- [ ] Confirmation dialog shows action summary and requires explicit confirmation
- [ ] Progress indicator displays for long-running batch operations
- [ ] Partial failure handling shows which items succeeded/failed
- [ ] Batch operation audit log records who performed what action when
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Accidental bulk actions affecting many clients | Medium | High | Confirmation dialogs, undo capability, action logging |
| Batch operations timing out for large selections | Medium | High | Async processing, background jobs, progress tracking |
| Partial failures leaving inconsistent state | Medium | High | Transaction boundaries, compensation logic, retry mechanism |
| Performance impact on system during batch operations | Medium | Medium | Rate limiting, off-peak scheduling, resource throttling |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Batch selection and UI framework | M | story-02-dashboard-ui |
| slice-b | Bulk reassign and status update | M | slice-a |
| slice-c | Batch reminders and exports | M | slice-a |
| slice-d | Progress tracking and error handling | S | slice-b, slice-c |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Batch selection UI and framework
- [slice-b.md](./slice-b.md) — Bulk reassign and status update operations
- [slice-c.md](./slice-c.md) — Batch reminders and data exports
- [slice-d.md](./slice-d.md) — Progress tracking and partial failure handling
