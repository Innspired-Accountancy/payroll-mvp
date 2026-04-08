# Story: P60 Document Access

**Epic:** epic-06-employee-portal
**Priority:** 4 of 8
**Sprint:** Month 4
**Dependencies:** epic-01-core-payroll (year-end processing), story-18-portal-auth (authentication)

---

## Goal

Enable employees to access their P60 year-end tax documents through the portal. P60s are legally required documents showing total pay and deductions for the tax year, and employees must be able to view and download them securely.

---

## Acceptance Criteria

- [ ] Employee sees list of available P60s by tax year (e.g., "2025-2026")
- [ ] Only P60s for completed tax years where employee was employed on 5 April are shown
- [ ] P60 displays: employer name, employer PAYE reference, employee name, NI number, tax code, total pay, total tax, total NICs
- [ ] Employee can download P60 as PDF
- [ ] Download is logged with timestamp, IP, and document ID
- [ ] P60s available from year-end processing completion date
- [ ] Strict data isolation - employee only sees their own P60s
- [ ] Mobile-responsive view for P60 display
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| P60 data accuracy | Low | Critical | Source from final FPS submission data, validate against payroll totals |
| Year-end timing confusion | Medium | Medium | Clear messaging: "P60s available after 31 May following tax year end" |
| Multiple employments | Medium | Medium | Show separate P60s per employment, clearly labeled by employer |
| Historical P60 migration | Medium | Low | Import from previous system during onboarding, mark as "archived" |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | P60 list and view API/UI | M | epic-01-core-payroll, story-18-portal-auth |
| slice-b | P60 PDF download with audit | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — P60 list and HTML view with tax year filtering
- [slice-b.md](./slice-b.md) — P60 PDF download and audit logging
