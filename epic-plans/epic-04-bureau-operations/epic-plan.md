# Epic: Bureau Operations Centre

**Date:** 2026-04-08
**Sprint(s):** Months 5-6
**Dependencies:** epic-01-core-payroll, epic-02-hmrc-submissions, epic-03-pension-ae

---

## Scope & Deliverables

Multi-client dashboard, exception queue management, task tracking, automated client chasing, and deadline calendar. The key differentiator module.

### In Scope
- Multi-client dashboard with status views
- Exception queue (HMRC failures, pension rejections, missing data)
- Task/deadline tracking
- Automated client reminders
- Client communication logging
- Batch operations

### Out of Scope
- Advanced SLA tracking (phase 2)
- Capacity planning algorithms (phase 2)
- AI-powered anomaly detection (phase 2)

---

## Decisions

### Libraries & Packages

| Package | Version | Rationale | License |
|---------|---------|-----------|---------|
| celery | 5.3.x | Background task processing | BSD |
| celery-beat | 5.3.x | Scheduled tasks (reminders) | BSD |
| sendgrid | 6.x | Email delivery | MIT |

### Real-Time Updates

| Aspect | Decision | Details |
|--------|----------|---------|
| Transport | WebSocket | Socket.io fallback |
| Events | Server-Sent Events | For dashboard updates |
| State | React Query + Events | Optimistic updates |

### Exception Types

| Type | Source | Severity |
|------|--------|----------|
| HMRC_FAILURE | HMRC module | Critical |
| PENSION_REJECTED | Pension module | High |
| PAYMENT_FAILED | Payments module | High |
| MISSING_DATA | Bureau operations | Medium |
| APPROVAL_PENDING | Workflow | Low |

### Automation Rules

| Rule | Trigger | Action |
|------|---------|--------|
| Cut-off reminder | 2 days before cut-off + missing data | Email client |
| Escalation | 1 day overdue | Notify manager |
| Chase | Data still missing 24h after reminder | Flag for manual |

---

## Build Order (Story Sequence)

| # | Story | Description | Effort | Dependencies |
|---|-------|-------------|--------|--------------|
| 1 | story-01-dashboard-backend | Status aggregation API | M | epic-01, epic-02, epic-03 |
| 2 | story-02-dashboard-ui | Dashboard views, filters | M | story-01 |
| 3 | story-03-exception-queue | Exception creation, management | L | story-01 |
| 4 | story-04-task-tracking | Task assignment, due dates | M | story-02 |
| 5 | story-05-client-chasing | Automated reminders | M | story-04 |
| 6 | story-06-communications | Email logging, history | S | story-05 |
| 7 | story-07-calendar | Deadline calendar view | S | story-02 |
| 8 | story-08-batch-ops | Multi-client batch actions | M | story-02 |

---

## Decision Completeness Checklist

- [x] All third-party libraries named with versions
- [x] Real-time update approach defined (WebSocket/SSE)
- [x] Exception categories defined
- [x] Automation rules specified
- [x] Task scheduling approach defined (Celery Beat)
- [x] No "TBD", slash-notation, or placeholder text remaining
