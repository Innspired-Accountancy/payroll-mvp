# Story: Personal Detail Change Requests

**Epic:** epic-06-employee-portal
**Priority:** 8 of 8
**Sprint:** Month 4
**Dependencies:** story-24-profile-view (profile viewing)

---

## Goal

Enable employees to request changes to their personal details (address, phone, bank details) through the portal with an approval workflow. Changes to sensitive data like bank details require employer/bureau approval for security and compliance.

---

## Acceptance Criteria

- [ ] Employee can initiate change request from profile page
- [ ] Supported change types: home address, phone number, bank details
- [ ] Form shows current value and allows entering new value
- [ ] Bank detail changes include: account name, sort code (validated format), account number (validated length)
- [ ] Address changes include: line1, line2, city, postcode (postcode validated)
- [ ] Phone changes validate UK mobile format
- [ ] Submitting creates change request with "pending_approval" status
- [ ] Employee sees pending status on profile with option to cancel
- [ ] Employer/bureau approver receives notification of pending change
- [ ] Upon approval, employee record is updated and employee is notified
- [ ] Upon rejection, employee sees rejection reason
- [ ] All change requests logged with before/after values for audit
- [ ] Employee cannot submit new change for same field while one is pending
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Fraudulent bank detail changes | Low | Critical | Mandatory approval workflow, notification to employer, audit trail |
| Invalid sort code/account numbers | Medium | Medium | Modulus check validation on sort code/account number combination |
| Change request spam | Low | Low | Rate limiting (max 3 pending requests), cancellation option |
| Approver delay | Medium | Medium | Escalation after 5 business days, reminder notifications |
| Concurrent conflicting changes | Low | Medium | Row-level locking or optimistic concurrency on approval |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Change request form and submission | M | story-24-profile-view |
| slice-b | Approval workflow and status tracking | M | slice-a |
| slice-c | Bank detail validation and notification | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Change request form with validation
- [slice-b.md](./slice-b.md) — Approval workflow and status management
- [slice-c.md](./slice-c.md) — Bank validation modulus check and notifications
