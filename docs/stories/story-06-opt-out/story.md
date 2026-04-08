# Story: Opt-Out Workflow

**Epic:** epic-03-pension-ae
**Priority:** 6 of 8
**Sprint:** Month 4
**Dependencies:** story-03-enrolment-workflow (enrolment records exist)

---

## Goal

Implement the opt-out workflow that allows employees to opt out of pension enrolment within the 1-month window, processes refunds of employee contributions, and maintains compliance records per The Pensions Regulator requirements.

---

## Acceptance Criteria

- [ ] Record opt-out request within 1-month opt-out window
- [ ] Validate opt-out window is still open
- [ ] Update enrolment status to "opted_out"
- [ ] Calculate refund amount (employee contributions only)
- [ ] Process refund via next payroll run
- [ ] Notify NEST of opt-out (via API or file)
- [ ] Retain opt-out records for 4 years (compliance)
- [ ] Handle opt-out via online portal or paper form
- [ ] Generate opt-out confirmation letter
- [ ] Track 12-month exclusion from re-auto-enrolment
- [ ] All tests pass
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Opt-out after window closes | Medium | High | Window validation, clear user messaging |
| Refund calculation errors | Low | High | Unit tests, audit trail |
| Missed NEST notification | Low | Medium | Queue and retry, status tracking |
| Lost opt-out evidence | Low | Critical | Immutable records, 4-year retention |
| Employee confusion about re-enrolment | Medium | Medium | Clear communication, 12-month tracking |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Opt-out recording and validation | M | story-03-enrolment-workflow |
| slice-b | Refund calculation and processing | M | slice-a |
| slice-c | NEST opt-out notification | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Opt-out recording and window validation
- [slice-b.md](./slice-b.md) — Refund calculation and payroll processing
- [slice-c.md](./slice-c.md) — NEST opt-out notification
