# Story: Year End Processing

**Epic:** epic-01-core-payroll
**Priority:** 11 of 11
**Sprint:** Month 4
**Dependencies:** story-10-payslip-generation

---

## Goal

Implement tax year-end processing including P60 generation, final FPS submission, YTD carry forward to new tax year, and year-end reconciliation reports. Ensure clean transition between tax years with data integrity.

---

## Acceptance Criteria

- [ ] Generate P60 forms for all employees with complete year data
- [ ] Produce P60 PDFs with HMRC-compliant format
- [ ] Generate final FPS with year-end indicators
- [ ] Create year-end reconciliation report (expected vs actual)
- [ ] Carry forward employee YTD balances to new tax year
- [ ] Reset tax codes per HMRC guidance (P9X updates)
- [ ] Archive completed tax year data with retention compliance
- [ ] Support mid-year employer registration (pro-rata thresholds)
- [ ] Handle leavers with P45 already issued (no P60)
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Year-end deadline pressure | High | High | Early testing, automated checks, rollback plan |
| Data migration errors | Low | Critical | Dry-run testing, backup verification, reconciliation |
| P60 corrections after issue | Medium | Medium | Amendment process, audit trail, communication |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Year-end data aggregation | M | None |
| slice-b | P60 generation (data and PDF) | M | slice-a |
| slice-c | Final FPS preparation | S | slice-a |
| slice-d | YTD carry forward | M | slice-a |
| slice-e | Year-end reconciliation reports | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Year-end data aggregation and validation
- [slice-b.md](./slice-b.md) — P60 form generation and PDF
- [slice-c.md](./slice-c.md) — Final FPS with year-end indicators
- [slice-d.md](./slice-d.md) — YTD carry forward to new tax year
- [slice-e.md](./slice-e.md) — Year-end reconciliation reports
