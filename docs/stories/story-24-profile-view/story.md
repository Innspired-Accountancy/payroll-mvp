# Story: Profile View

**Epic:** epic-06-employee-portal
**Priority:** 7 of 8
**Sprint:** Month 4
**Dependencies:** story-18-portal-auth (authentication), epic-01-core-payroll (employee data)

---

## Goal

Enable employees to view their personal details and employment information through the portal. This includes contact information, address, bank details (masked), emergency contacts, and employment terms. This is read-only viewing; changes require the change request workflow.

---

## Acceptance Criteria

- [ ] Employee sees personal details: name, DOB, NI number, contact info
- [ ] Address displayed in readable format with postcode
- [ ] Bank details displayed with account number masked (****5678)
- [ ] Sort code displayed in standard format (12-34-56)
- [ ] Emergency contact name and phone visible
- [ ] Employment details: start date, job title, department, employment type
- [ ] Tax code and pension scheme visible
- [ ] All data is read-only (no inline editing)
- [ ] "Request Change" buttons next to editable fields
- [ ] Last updated timestamp shown per section
- [ ] Mobile-responsive layout with collapsible sections
- [ ] Strict data isolation - only own profile accessible
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| PII data exposure | Low | Critical | Mask bank details, require auth for all access, audit logging |
| Outdated information confusion | Medium | Medium | Clear "last updated" timestamps, pending change indicators |
| Mobile layout complexity | Low | Low | Collapsible accordion sections, test on multiple devices |
| Data accuracy from source | Medium | Medium | Single source of truth from employee table, real-time queries |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Profile API and personal details view | S | story-18-portal-auth |
| slice-b | Employment details and bank details view | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Personal details view with masked sensitive data
- [slice-b.md](./slice-b.md) — Employment details and bank details display
