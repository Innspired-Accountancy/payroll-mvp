# Story: Payslip Download

**Epic:** epic-06-employee-portal
**Priority:** 3 of 8
**Sprint:** Month 4
**Dependencies:** story-19-payslip-view (payslip viewing infrastructure)

---

## Goal

Enable employees to download their payslips as PDF documents for record-keeping and external use. Downloads must be secure, audited, and generate PDFs on-demand or serve pre-generated files with integrity verification.

---

## Acceptance Criteria

- [ ] Employee can download any visible payslip as PDF
- [ ] Download button available on both list and detail views
- [ ] PDF generation uses existing payslip template (consistent with email PDFs)
- [ ] Download is logged with timestamp, IP, and payslip ID for audit
- [ ] PDF filename includes employee number and pay date (e.g., "EMP001-2026-04-30-payslip.pdf")
- [ ] Download completes within 5 seconds (target: <2 seconds)
- [ ] Failed downloads show user-friendly error message
- [ ] Strict data isolation - only employee's own payslips downloadable
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| PDF generation performance | Medium | Medium | Pre-generate PDFs during payroll finalization, serve cached files |
| Storage costs for PDFs | Low | Low | VPS disk storage (sufficient for MVP), compress PDFs, S3 migration path |
| Direct URL access bypass | Low | High | Signed URLs with expiry (15 minutes), auth middleware on download endpoint |
| Browser compatibility | Low | Low | Standard PDF MIME type, test in Chrome/Firefox/Safari/Edge |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | PDF download endpoint with audit logging | S | story-19-payslip-view |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Secure PDF download with audit logging
