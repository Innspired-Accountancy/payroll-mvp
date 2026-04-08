# Story: Pension Contributions Calculation

**Epic:** epic-03-pension-ae
**Priority:** 4 of 8
**Sprint:** Month 4
**Dependencies:** story-02-assessment-engine (qualifying earnings), story-01-pension-schemes (scheme config)

---

## Goal

Implement pension contribution calculation including qualifying earnings determination, employee and employer contribution computation, and tax relief application. Support both relief at source and net pay arrangements per scheme configuration.

---

## Acceptance Criteria

- [ ] Calculate qualifying earnings using banded thresholds
- [ ] Calculate employee contribution per scheme rate (percentage or fixed)
- [ ] Calculate employer contribution per scheme rate
- [ ] Apply relief at source (RAS) - add basic rate tax relief to employee contribution
- [ ] Apply net pay arrangement - reduce taxable pay before tax calculation
- [ ] Handle certified schemes with different earnings bases
- [ ] Support salary sacrifice arrangements (optional)
- [ ] Record all contributions with full audit trail
- [ ] Include contributions in payslip generation
- [ ] All tests pass with HMRC reference calculations
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Relief method confusion (RAS vs net pay) | Medium | High | Clear scheme configuration validation, test cases |
| Contribution rate changes mid-period | Low | Medium | Effective dating for rate changes |
| Rounding errors in totals | Medium | Medium | Decimal.js with 4dp precision, consistent rounding |
| Salary sacrifice impact on other calculations | Low | High | Integration tests with full payroll calculation |
| Certified scheme complexity | Low | Medium | Separate calculation path for certified schemes |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Contribution calculation engine | M | story-02-assessment-engine |
| slice-b | Relief method application | S | slice-a |
| slice-c | Payslip integration | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Core contribution calculation logic
- [slice-b.md](./slice-b.md) — Tax relief method implementation
- [slice-c.md](./slice-c.md) — Payslip integration and display
