# Story: Submission Status Polling

**Epic:** epic-02-hmrc-submissions
**Priority:** 4 of 9
**Sprint:** Month 2
**Dependencies:** story-03-hmrc-gateway

---

## Goal

Implement a state machine to track submission status from "submitted" through to final "accepted" or "rejected". Poll HMRC gateway for acknowledgements, update submission records, and trigger downstream actions (notifications, error handling, evidence storage).

---

## Acceptance Criteria

- [ ] State machine with states: draft → validated → submitted → acknowledged → accepted/rejected
- [ ] Poll HMRC gateway every 30 seconds for pending submissions
- [ ] Stop polling after 24 hours or final state reached
- [ ] Parse acknowledgement XML responses from HMRC
- [ ] Update submission status and timestamps in database
- [ ] Store HMRC correlation ID and acknowledgement ID
- [ ] Trigger notification on status change to accepted/rejected
- [ ] Handle timeout (no response within 24 hours) as error condition
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| HMRC response delays | Medium | Medium | 24-hour timeout window, manual check option, alerting |
| State machine complexity | Medium | Medium | Clear state transitions, audit logging, prevent invalid transitions |
| Polling overhead | Medium | Low | Efficient queries, index on status+submitted_at, rate limiting |
| Duplicate acknowledgements | Low | Medium | Idempotent status updates, correlation ID deduplication |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | State machine implementation | M | None |
| slice-b | Polling service with scheduler | M | slice-a |
| slice-c | Acknowledgement response parser | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — State machine with XState library
- [slice-b.md](./slice-b.md) — Polling scheduler and HMRC status endpoint
- [slice-c.md](./slice-c.md) — Acknowledgement XML parser and status updates
