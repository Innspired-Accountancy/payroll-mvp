# Story: Student Loan Deductions

**Epic:** epic-01-core-payroll
**Priority:** 6 of 11
**Sprint:** Month 2
**Dependencies:** story-03-tax-engine

---

## Goal

Implement student loan deduction calculations for all UK plans (Plan 1, Plan 2, Plan 4, Plan 5) and Postgraduate Loan (PGL). Handle plan type identification, threshold checks, percentage deductions, and year-to-date threshold tracking.

---

## Acceptance Criteria

- [ ] Deduct Plan 1 at 9% on earnings above annual threshold (£24,990 for 2026-27)
- [ ] Deduct Plan 2 at 9% on earnings above annual threshold (£27,295 for 2026-27)
- [ ] Deduct Plan 4 at 9% on earnings above annual threshold (£31,395 for 2026-27)
- [ ] Deduct Plan 5 at 9% on earnings above annual threshold (£25,000 for 2026-27)
- [ ] Deduct PGL at 6% on earnings above threshold (£21,000 for 2026-27)
- [ ] Handle multiple loan types (plan + PGL) concurrently
- [ ] Pro-rata thresholds for weekly/monthly pay periods
- [ ] Stop deductions when SLC sends stop notice (SD1, SD2, etc.)
- [ ] Report loan deductions on FPS with correct plan type
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Annual threshold changes | High | Medium | Config-driven thresholds, annual review process |
| Plan type misidentification | Medium | High | Employee self-declaration + SLC verification |
| Stop notice handling | Low | Medium | SLC notification integration, manual override |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Student loan plan configuration | S | None |
| slice-b | Plan 1/2/4/5 deduction calculator | M | slice-a |
| slice-c | Postgraduate loan calculator | S | slice-a |
| slice-d | Stop notice handling | S | slice-b, slice-c |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Student loan plan configuration and thresholds
- [slice-b.md](./slice-b.md) — Undergraduate plan deduction calculator
- [slice-c.md](./slice-c.md) — Postgraduate loan calculator
- [slice-d.md](./slice-d.md) — SLC stop notice handling
