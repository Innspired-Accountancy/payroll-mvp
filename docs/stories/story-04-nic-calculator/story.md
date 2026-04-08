# Story: NIC Calculation Engine

**Epic:** epic-01-core-payroll
**Priority:** 4 of 11
**Sprint:** Month 2
**Dependencies:** story-02-payroll-setup, story-03-tax-engine

---

## Goal

Implement HMRC-compliant National Insurance Contributions calculation for all employee categories (A, B, C, H, J, M, Z, X), supporting both employee and employer NIC with correct LEL, PT, UEL thresholds and rates per tax year.

---

## Acceptance Criteria

- [ ] Calculate employee NIC for categories A, B, C, H, J, M, Z, X using exact HMRC tables
- [ ] Calculate employer NIC for all categories with correct secondary thresholds
- [ ] Handle pro-rata thresholds for weekly, monthly, and irregular pay periods
- [ ] Support director annual calculation method with election trigger
- [ ] Handle multiple employments with correct LEL treatment
- [ ] Calculate NIC on statutory payments (SSP, SMP, etc.) where applicable
- [ ] Support contracted-out calculations (if applicable for historical data)
- [ ] 100% match HMRC NIC reference calculations
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Category rate changes mid-year | Low | High | Config-driven rates, versioned tax year config |
| Director annual calculation complexity | Medium | Medium | Separate calculation path, explicit election handling |
| Multiple employments LEL aggregation | Medium | Medium | HMRC guidance verification, edge case testing |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | NIC category config and threshold tables | S | None |
| slice-b | Employee NIC calculator core | L | slice-a |
| slice-c | Employer NIC calculator | M | slice-b |
| slice-d | Director annual calculation | M | slice-b |
| slice-e | Pro-rata threshold calculations | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — NIC category configuration and threshold tables
- [slice-b.md](./slice-b.md) — Core employee NIC calculator
- [slice-c.md](./slice-c.md) — Employer NIC calculation
- [slice-d.md](./slice-d.md) — Director annual calculation method
- [slice-e.md](./slice-e.md) — Pro-rata thresholds for different frequencies
