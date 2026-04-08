# Story: Task Tracking - Task Assignment and Due Dates

**Epic:** epic-04-bureau-operations
**Priority:** 4 of 8
**Sprint:** Month 4
**Dependencies:** story-02-dashboard-ui

---

## Goal

Implement task tracking for payroll processing workflows. Enable explicit task assignment with due dates, track task status through completion, and provide visibility into team workload and individual task queues. Tasks are linked to payroll periods and client records.

---

## Acceptance Criteria

- [ ] Task data model supports payroll-related tasks with due dates and assignments
- [ ] Tasks can be created manually by bureau staff or automatically by the system
- [ ] Task types include: data collection, review, approval, submission, follow-up
- [ ] Tasks display in assignee's personal queue with priority and due date
- [ ] Tasks support status workflow: pending, in_progress, blocked, complete
- [ ] Overdue tasks are highlighted and trigger notifications
- [ ] Tasks can be reassigned between team members
- [ ] Task completion updates related payroll status
- [ ] Task history is retained for audit and performance tracking
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Task proliferation overwhelming users | Medium | Medium | Auto-cleanup, bulk actions, smart defaults |
| Due date misalignment with client pay schedules | Medium | Medium | Configurable rules, cut-off date integration, validation |
| Task/exception duplication causing confusion | Medium | Medium | Clear separation of concerns, linking mechanism |
| Notification fatigue from task alerts | Low | Medium | Batched notifications, digest mode, preference settings |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Task data model and API | M | story-02-dashboard-ui |
| slice-b | Task queue UI and management | M | slice-a |
| slice-c | Automatic task creation rules | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Task data model and CRUD API
- [slice-b.md](./slice-b.md) — Task queue UI and assignment workflow
- [slice-c.md](./slice-c.md) — Automatic task creation from payroll events
