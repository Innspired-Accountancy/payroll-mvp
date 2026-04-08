# Story: Payslip View

**Epic:** epic-06-employee-portal
**Priority:** 2 of 8
**Sprint:** Month 4
**Dependencies:** story-18-portal-auth (authentication), epic-01-core-payroll (payslip generation)

---

## Goal

Enable employees to view their payslip history and detailed payslip breakdowns through the portal. Employees must only see their own payslips with year-to-date summaries and itemized earnings/deductions. This is a core self-service feature required for V1.

---

## Acceptance Criteria

- [ ] Employee sees list of all their payslips sorted by date (newest first)
- [ ] List view shows pay date, period, gross pay, and net pay per payslip
- [ ] Clicking a payslip shows detailed view with itemized breakdown
- [ ] Detailed view includes: earnings (basic, bonus, etc.), deductions (tax, NI, pension), employer contributions
- [ ] Year-to-date totals displayed (taxable pay, tax paid, NIC paid)
- [ ] Tax code and NI number visible on payslip detail
- [ ] Employer name and pay date clearly displayed
- [ ] Strict data isolation - employee can only view their own payslips
- [ ] All access logged with timestamp and IP address
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Data isolation failure (IDOR) | Low | Critical | tRPC middleware enforcing employee_id filter on all queries |
| Performance with many payslips | Low | Medium | Pagination on list view (10 payslips per page) |
| Historical payslip data incomplete | Medium | Medium | Graceful handling of missing data, show "data not available" message |
| Mobile responsiveness issues | Medium | Medium | Responsive table design, test on multiple screen sizes |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Payslip list API and UI | M | story-18-portal-auth |
| slice-b | Payslip detail view with itemized breakdown | M | slice-a |
| slice-c | YTD summary and access logging | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Payslip list API with pagination and list UI
- [slice-b.md](./slice-b.md) — Payslip detail view with full itemization
- [slice-c.md](./slice-c.md) — YTD summary calculation and access audit logging
