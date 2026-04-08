# Story: New Employee Starter Form

**Epic:** epic-07-employer-portal
**Priority:** 5 of 8
**Sprint:** Sprint 5
**Dependencies:** story-26-client-auth, story-01-employee-mgmt

## Goal

Provide a digital HMRC starter checklist form for employer admins to capture new employee details, including personal information, P45 data when available, and starter declaration. Ensure all required fields for payroll onboarding are collected.

## Acceptance Criteria

- [ ] Digital HMRC starter checklist form
- [ ] Personal details: name, address, DOB, NI number
- [ ] Contact details: email, phone
- [ ] Employment details: start date, job title, department
- [ ] Payment details: bank account (pending bureau approval)
- [ ] P45 data capture when available (tax code, previous earnings)
- [ ] Starter declaration (A, B, C options)
- [ ] Student loan status capture
- [ ] Form validation with clear error messages
- [ ] Submission notification to bureau
- [ ] New employee queued for next payroll
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Incorrect starter declaration selected | Medium | High | Clear guidance; help text; validation |
| Invalid NI number format | Medium | Medium | NI number validation; checksum verification |
| Missing required fields delaying payroll | Medium | High | Field validation; required indicators |
| Duplicate employee creation | Medium | Medium | Duplicate detection; NI number matching |
| P45 data entry errors | Medium | Medium | Field-level validation; range checks |
| Bank detail changes without approval | Low | High | Flag for bureau review; separate approval flow |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Starter form API and validation | M | story-26-client-auth |
| slice-b | Starter form UI with HMRC layout | M | slice-a |
| slice-c | Bank details capture and approval flag | S | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Starter form API with validation
- [slice-b.md](./slice-b.md) — Starter form UI components
- [slice-c.md](./slice-c.md) — Bank details and bureau notification
