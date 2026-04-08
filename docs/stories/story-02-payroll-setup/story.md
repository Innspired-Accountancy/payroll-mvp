# Story: Payroll Setup

**Epic:** epic-01-core-payroll
**Priority:** 2 of 11
**Sprint:** Month 1
**Dependencies:** story-01-employee-mgmt

---

## Goal

Enable configuration of payroll schedules (weekly, monthly, lunar, quarterly), pay period generation, and tax year calendar management. This is the foundation for running payroll calculations and ensuring correct period boundaries and tax period mappings.

---

## Acceptance Criteria

- [ ] Create payroll schedules with frequency (weekly/monthly/lunar/quarterly), anchor date, and pay day offset
- [ ] Auto-generate pay periods for a tax year with correct start/end dates and tax period numbers
- [ ] Handle period 12/13 splits for monthly and weekly schedules per HMRC rules
- [ ] Support custom pay dates with validation (must be after period end)
- [ ] Prevent overlapping periods within the same schedule
- [ ] Display tax year calendar with all periods visualized
- [ ] Allow period deletion only when no pay runs attached
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Tax period mapping errors | Medium | Critical | HMRC CWG2 reference, automated tests for period boundaries |
| Week 53 edge case handling | Medium | High | Explicit Week 53 logic, HMRC compliance tests |
| Schedule changes with existing pay runs | Medium | High | Block schedule changes when pay runs exist, migration workflow |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Payroll schedule data model and API | M | None |
| slice-b | Pay period generation engine | M | slice-a |
| slice-c | Tax period mapping and Week 53 handling | S | slice-b |
| slice-d | Payroll calendar UI | M | slice-a, slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Payroll schedule model and CRUD API
- [slice-b.md](./slice-b.md) — Pay period generation with date calculations
- [slice-c.md](./slice-c.md) — HMRC tax period mapping and Week 53 handling
- [slice-d.md](./slice-d.md) — Payroll calendar visualization UI
