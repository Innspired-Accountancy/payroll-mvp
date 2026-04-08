# Story: Payslip Generation

**Epic:** epic-01-core-payroll
**Priority:** 10 of 11
**Sprint:** Month 3
**Dependencies:** story-08-pay-run-lifecycle

---

## Goal

Generate compliant UK payslips in both HTML and PDF formats with all required fields (gross pay, deductions, net pay, YTD totals, tax code, NI category). Support bulk generation, email distribution, and employee portal access.

---

## Acceptance Criteria

- [ ] Generate payslips with all statutory required fields
- [ ] Display earnings breakdown (basic, overtime, bonus, statutory payments)
- [ ] Display deductions breakdown (tax, NIC, pension, student loan, other)
- [ ] Show YTD totals for taxable pay, tax, NIC, pension
- [ ] Include employer info, employee name, NI number, tax code
- [ ] Show pay period dates, pay date, tax period
- [ ] Generate PDF with professional formatting
- [ ] Support bulk PDF generation for pay run
- [ ] Store payslip PDFs securely with encryption
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| PDF generation performance | Medium | Medium | Streaming generation, caching, worker processes |
| PDF accessibility compliance | Medium | Medium | PDF/UA standard, screen reader testing |
| Storage costs for PDFs | Medium | Low | Compression, retention policies, tiered storage |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Payslip data model and aggregation | M | None |
| slice-b | Payslip HTML template | M | slice-a |
| slice-c | PDF generation service | M | slice-b |
| slice-d | Bulk generation and storage | M | slice-c |
| slice-e | Employee portal payslip view | M | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Payslip data aggregation from calculation
- [slice-b.md](./slice-b.md) — Payslip HTML template component
- [slice-c.md](./slice-c.md) — PDF generation service
- [slice-d.md](./slice-d.md) — Bulk generation and secure storage
- [slice-e.md](./slice-e.md) — Employee portal payslip view
