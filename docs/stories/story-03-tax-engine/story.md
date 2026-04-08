# Story: PAYE Tax Calculation Engine

**Epic:** epic-01-core-payroll
**Priority:** 3 of 11
**Sprint:** Month 2
**Dependencies:** story-01-employee-mgmt, story-02-payroll-setup

---

## Goal

Implement HMRC-compliant PAYE tax calculation including cumulative basis, Week 1/Month 1, Scottish/Welsh rates, K-codes, and emergency tax codes.

---

## Acceptance Criteria

- [ ] Calculate tax using HMRC exact percentage method
- [ ] Support cumulative and non-cumulative tax bases
- [ ] Handle all standard tax codes (1257L, BR, D0, D1, 0T, NT, K-codes)
- [ ] Support Scottish (S-prefix) and Welsh (C-prefix) rates
- [ ] Handle K-code negative tax correctly (max 50% of pay)
- [ ] Track cumulative taxable pay and tax per employee
- [ ] Handle mid-year tax code changes with recalculation
- [ ] 100% match HMRC reference calculations for test scenarios
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Calculation errors | Low | Critical | HMRC test packs, reference data validation |
| Tax year boundary issues | Medium | High | Comprehensive testing around April 6 |
| Scottish/Welsh rate changes | Low | Medium | Config-driven rates, annual review |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Tax code parser and validator | S | None |
| slice-b | Tax calculator core (cumulative) | L | slice-a |
| slice-c | Scottish and Welsh rates | S | slice-b |
| slice-d | K-codes and special cases | M | slice-b |
| slice-e | Tax year configuration | M | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Tax code parser
- [slice-b.md](./slice-b.md) — Core tax calculator
- [slice-c.md](./slice-c.md) — Scottish and Welsh rates
- [slice-d.md](./slice-d.md) — K-codes and special handling
- [slice-e.md](./slice-e.md) — Tax year configuration
