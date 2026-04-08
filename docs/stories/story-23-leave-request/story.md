# Story: Leave Request Workflow

**Epic:** epic-06-employee-portal
**Priority:** 6 of 8
**Sprint:** Month 4
**Dependencies:** story-22-leave-view (leave viewing), Leave Module (request processing)

---

## Goal

Enable employees to submit leave requests through the portal with date selection, leave type choice, and reason input. Requests must trigger approval workflows, check for conflicts, and notify both employee and approver of status changes.

---

## Acceptance Criteria

- [ ] Employee can initiate new leave request from leave page
- [ ] Request form includes: leave type dropdown, start date, end date, days (auto-calculated), reason textarea
- [ ] System validates dates (start <= end, not in past beyond 30 days)
- [ ] System checks for existing approved leave conflicts
- [ ] System validates sufficient balance before allowing submission
- [ ] Submitting creates request with "pending" status
- [ ] Employee receives confirmation notification (in-app + email)
- [ ] Approver receives notification of pending request
- [ ] Employee can cancel own pending requests
- [ ] Cannot edit submitted requests (cancel and resubmit instead)
- [ ] Real-time balance preview showing remaining after request
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Double-booking/overlap | Medium | High | Exclusion constraint or application check for date overlaps |
| Insufficient balance approval | Low | Medium | Pre-submission balance validation, warning if overdrawn |
| Approver notification failure | Medium | High | Retry queue for failed notifications, in-app notification fallback |
| Backdated requests abuse | Low | Medium | Block requests >30 days in past, require admin override |
| Half-day complexity | Medium | Medium | Support half-day selection (AM/PM), calculate days accordingly |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Leave request form with validation | M | story-22-leave-view |
| slice-b | Conflict checking and submission | M | slice-a |
| slice-c | Notification integration and cancel functionality | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Leave request form with date picker and validation
- [slice-b.md](./slice-b.md) — Conflict detection and submission workflow
- [slice-c.md](./slice-c.md) — Email notifications and request cancellation
