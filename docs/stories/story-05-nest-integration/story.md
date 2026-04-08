# Story: NEST Integration

**Epic:** epic-03-pension-ae
**Priority:** 5 of 8
**Sprint:** Month 4
**Dependencies:** story-03-enrolment-workflow, story-04-contributions

---

## Goal

Implement NEST (National Employment Savings Trust) API integration for submitting enrolments, contributions, and opt-outs. Support both real-time API submission and file-based fallback for compliance.

---

## Acceptance Criteria

- [ ] Authenticate with NEST Web Services using employer credentials
- [ ] Submit enrolment data for newly enrolled employees
- [ ] Submit contribution schedules per pay period
- [ ] Submit opt-out notifications
- [ ] Poll for submission status and handle responses
- [ ] Reconcile submitted vs accepted records
- [ ] Handle NEST API errors with retry logic
- [ ] Support file-based submission fallback (CSV/CSV)
- [ ] Log all submissions with provider reference
- [ ] Generate submission reports for audit
- [ ] All tests pass including error scenarios
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| NEST API downtime | Medium | High | File-based fallback, queue and retry |
| API schema changes | Medium | High | Versioned API client, monitoring |
| Authentication failures | Low | High | Credential monitoring, automatic retry |
| Submission rejections | Medium | Medium | Validation before submit, error reporting |
| Rate limiting | Low | Medium | Exponential backoff, queue management |
| Data mismatch on reconciliation | Low | High | Detailed logging, manual review workflow |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | NEST API client and authentication | M | story-03-enrolment-workflow, story-04-contributions |
| slice-b | Enrolment submission | M | slice-a |
| slice-c | Contribution schedule submission | L | slice-a |
| slice-d | File-based fallback | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — NEST API client and authentication
- [slice-b.md](./slice-b.md) — Enrolment submission to NEST
- [slice-c.md](./slice-c.md) — Contribution schedule submission
- [slice-d.md](./slice-d.md) — File-based fallback mechanism
