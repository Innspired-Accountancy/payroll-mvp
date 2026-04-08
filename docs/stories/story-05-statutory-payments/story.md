# Story: Statutory Payments (SSP, SMP, SPP)

**Epic:** epic-01-core-payroll
**Priority:** 5 of 11
**Sprint:** Month 2
**Dependencies:** story-03-tax-engine, story-04-nic-calculator

---

## Goal

Implement Statutory Sick Pay (SSP), Statutory Maternity Pay (SMP), and Statutory Paternity Pay (SPP) calculations including qualifying conditions, weekly rates, recovery calculations, and NIC treatment. Handle linked periods, average weekly earnings (AWE) calculations, and leave management.

---

## Acceptance Criteria

- [ ] Calculate SSP with PIW (Period of Incapacity for Work), qualifying days, and waiting days
- [ ] Calculate SMP with 90% for 6 weeks + statutory rate for 33 weeks
- [ ] Calculate SPP with statutory rate for 1-2 weeks
- [ ] AWE calculation using relevant period rules for each payment type
- [ ] Handle linked PIWs for SSP (no re-qualifying within 8 weeks)
- [ ] Calculate NIC on statutory payments correctly
- [ ] Calculate tax on statutory payments
- [ ] Track leave balances and payment history
- [ ] Generate recovery amounts for small employer relief (SSP only)
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| AWE calculation complexity | Medium | High | HMRC guidance compliance, reference scenarios |
| Linked PIW edge cases | Medium | Medium | Comprehensive state management, date arithmetic tests |
| SMP 90% vs flat rate boundaries | Low | Medium | Automated threshold calculations, boundary tests |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | SSP calculation engine | L | None |
| slice-b | SMP calculation engine | L | None |
| slice-c | SPP calculation engine | M | None |
| slice-d | AWE calculation utility | M | slice-a, slice-b, slice-c |
| slice-e | Leave tracking and history | M | slice-a, slice-b, slice-c |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — SSP calculation with PIW and qualifying days
- [slice-b.md](./slice-b.md) — SMP calculation with AWE and rate phases
- [slice-c.md](./slice-c.md) — SPP calculation engine
- [slice-d.md](./slice-d.md) — Average Weekly Earnings calculation utility
- [slice-e.md](./slice-e.md) — Leave tracking and payment history
