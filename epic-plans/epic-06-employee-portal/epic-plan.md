# Epic: Employee Portal

**Date:** 2026-04-08
**Sprint(s):** Months 5-6
**Dependencies:** epic-01-core-payroll (payslip data), epic-05-identity-access (auth)

---

## Scope & Deliverables

Employee self-service portal for payslip access, P60s, leave requests, and personal detail updates with strict self-only access.

### In Scope
- Payslip viewing and download
- P60/P45 access
- Leave requests and balance viewing
- Personal detail update requests (with approval)
- Bank detail change requests (with approval)
- MFA for portal access

### Out of Scope
- Mobile app (responsive web only)
- Push notifications (phase 2)

---

## Decisions

### UI/UX

| Aspect | Decision | Details |
|--------|----------|---------|
| Design | Mobile-first | Most employees use phones |
| PDF viewer | Browser native | Download for offline |
| Theme | White-label ready | Employer branding |

### Security

| Aspect | Decision | Details |
|--------|----------|---------|
| Self-access | Strict enforcement | Can only see own data |
| Change approval | Employer/bureau | Bank details require approval |
| Session | 30min timeout | Auto-logout |

---

## Build Order (Story Sequence)

| # | Story | Description | Effort | Dependencies |
|---|-------|-------------|--------|--------------|
| 1 | story-01-portal-auth | Employee login, MFA | M | epic-05 |
| 2 | story-02-payslip-view | Payslip list, detail view | M | epic-01 |
| 3 | story-03-payslip-download | PDF download | S | story-02 |
| 4 | story-04-p60-access | Year-end document access | S | epic-01 |
| 5 | story-05-leave-view | Balance, history | M | Leave module |
| 6 | story-06-leave-request | Request workflow | M | story-05 |
| 7 | story-07-profile-view | Personal details | S | epic-01 |
| 8 | story-08-change-requests | Update requests with approval | M | story-07 |

---

## Decision Completeness Checklist

- [x] Mobile-first design specified
- [x] Self-access enforcement defined
- [x] Change approval workflow defined
- [x] Session timeout specified
- [x] No "TBD", slash-notation, or placeholder text remaining
