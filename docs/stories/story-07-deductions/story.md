# Story: Deductions Engine

**Epic:** epic-01-core-payroll
**Priority:** 7 of 11
**Sprint:** Month 3
**Dependencies:** story-03-tax-calculator, story-04-nic-calculator

## Goal

Implement pre-tax and post-tax deduction calculations including salary sacrifice schemes and basic Attachment of Earnings Orders (AEOs). These deductions reduce taxable pay (salary sacrifice) or net pay (AEOs) and must integrate seamlessly with the tax and NIC calculation pipeline to produce accurate net pay.

What's in scope: Pension salary sacrifice, cycle-to-work schemes, childcare vouchers, and basic AEOs (non-priority and priority). What's out of scope: Complex AEO edge cases (consolidated orders, council tax variations), court orders with protected earnings override, and net-to-gross calculations.

## Acceptance Criteria

- [ ] Salary sacrifice deductions correctly reduce taxable gross before PAYE and NIC calculations
- [ ] Pre-tax deductions are applied in the correct order: pension → other salary sacrifice → tax calc
- [ ] AEOs are calculated on post-tax net pay with protected earnings thresholds enforced
- [ ] Multiple AEOs are prioritized correctly: CSA/CMS → court fines → council tax
- [ ] Deduction totals appear on payslips with clear categorisation
- [ ] YTD deduction tracking maintains accurate cumulative totals per deduction type
- [ ] All tests pass (unit + integration as applicable)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| AEO protected earnings calculations conflict with minimum wage requirements | Medium | High | Implement validation that checks NMW compliance before applying deductions |
| Multiple concurrent AEOs exceed available net pay | Medium | Medium | Implement proportional distribution logic when total AEOs exceed available funds |
| Salary sacrifice reduces pay below NMW threshold | Medium | High | Add pre-calculation validation with employer warning workflow |
| Complex AEO types (consolidated, offset) requested mid-sprint | Low | Medium | Clearly document "basic AEO" scope boundary in user-facing docs |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Deduction types data model and employee assignment | M | none |
| slice-b | Salary sacrifice calculation and tax/NIC integration | M | slice-a |
| slice-c | Basic AEO calculation engine | M | slice-a |
| slice-d | Payslip integration and YTD tracking | S | slice-b, slice-c |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Deduction types data model and employee deduction assignment
- [slice-b.md](./slice-b.md) — Salary sacrifice calculation with tax/NIC reduction integration
- [slice-c.md](./slice-c.md) — Basic AEO calculation engine with protected earnings
- [slice-d.md](./slice-d.md) — Payslip integration and YTD deduction tracking
