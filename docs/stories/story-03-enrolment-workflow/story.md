# Story: Auto-Enrolment Workflow

**Epic:** epic-03-pension-ae
**Priority:** 3 of 8
**Sprint:** Month 3
**Dependencies:** story-02-assessment-engine (eligible jobholders identified)

---

## Goal

Implement the automatic enrolment workflow that creates pension enrolment records for eligible jobholders, establishes opt-out windows, and triggers statutory communications. This ensures compliance with The Pensions Regulator's 6-week communication requirement.

---

## Acceptance Criteria

- [ ] Auto-enrol eligible jobholders identified by assessment engine
- [ ] Create PensionEnrolment record with enrolment date and opt-out deadline (1 month)
- [ ] Handle different enrolment reasons: automatic, opt_in, join_request
- [ ] Skip enrolment if employee already enrolled in same scheme
- [ ] Skip enrolment if employee opted out within last 12 months (re-enrolment rules)
- [ ] Trigger statutory communication generation within 6 weeks
- [ ] Include employee in pension contribution calculation from enrolment date
- [ ] Handle postponement correctly (3-month delay option)
- [ ] All tests pass
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Duplicate enrolments | Medium | High | Unique constraint on employee+scheme+active status |
| Missed 6-week communication deadline | Low | Critical | Immediate queue trigger, SLA monitoring alerts |
| Opt-out window calculation errors | Medium | High | Date-fns addMonths(1), explicit business day logic |
| Re-enrolment timing (12-month rule) | Medium | Medium | Check opt-out history before auto-enrol |
| Multiple eligible schemes | Low | Medium | Enrol in default scheme first |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Enrolment workflow engine | M | story-02-assessment-engine |
| slice-b | Enrolment status management | S | slice-a |
| slice-c | Communication trigger integration | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Core enrolment workflow logic
- [slice-b.md](./slice-b.md) — Enrolment status lifecycle management
- [slice-c.md](./slice-c.md) — Communication service integration
