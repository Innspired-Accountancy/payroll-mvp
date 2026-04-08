# Story: Worker Assessment Engine

**Epic:** epic-03-pension-ae
**Priority:** 2 of 8
**Sprint:** Month 3
**Dependencies:** story-01-pension-schemes (schemes exist), epic-01-core-payroll (employee, earnings data)

---

## Goal

Implement the worker categorization engine that assesses employees against age and earnings criteria to determine auto-enrolment eligibility per The Pensions Regulator requirements. This categorizes workers as Eligible Jobholder, Non-Eligible Jobholder, or Entitled Worker.

---

## Acceptance Criteria

- [ ] Assess worker age on assessment date (22 to State Pension Age)
- [ ] Calculate qualifying earnings (earnings between £6,240 and £50,270 annually for 2026-27)
- [ ] Categorize workers: Eligible Jobholder (auto-enrol), Non-Eligible Jobholder (can opt-in), Entitled Worker (can join)
- [ ] Handle multiple assessment triggers (new employee, periodic, postponement end)
- [ ] Record assessment with immutable audit trail
- [ ] Assess all workers every pay reference period
- [ ] Support earnings from multiple employments under same employer
- [ ] All tests pass including edge cases around age boundaries
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Age calculation errors at boundaries | Medium | High | Use date-fns for precise age calc, boundary tests |
| Qualifying earnings band changes annually | High | Medium | Config-driven bands per tax year |
| State Pension Age lookup incorrect | Low | High | HMRC state pension age API integration |
| Assessment timing across tax year boundary | Medium | Medium | Explicit tax year context in assessment |
| Multiple employments aggregation | Medium | Medium | Sum all employments before assessment |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Assessment engine core logic | M | story-01-pension-schemes |
| slice-b | Assessment trigger integration | S | slice-a |
| slice-c | Age and earnings calculation utilities | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Core assessment categorization logic
- [slice-b.md](./slice-b.md) — Assessment trigger integration with payroll
- [slice-c.md](./slice-c.md) — Age calculation and earnings utilities
