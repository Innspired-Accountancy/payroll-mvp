# Story: Payroll Review and Approval

**Epic:** epic-07-employer-portal
**Priority:** 4 of 8
**Sprint:** Sprint 4
**Dependencies:** story-28-variable-pay-form, story-08-pay-run-lifecycle

## Goal

Enable authorized client users (admins and managers with approval permission) to review payroll summaries, compare variances against previous periods, and approve or reject payrolls with comments before final bureau processing.

## Acceptance Criteria

- [ ] Payroll summary view with totals and employee breakdown
- [ ] Variance comparison vs previous pay period
- [ ] Employee-level summary with gross/net/tax figures
- [ ] Exception highlighting (unusual amounts, new employees)
- [ ] Approve action with optional comments
- [ ] Reject action with required comments
- [ ] Approval/rejection notifications to bureau
- [ ] Audit trail of all approval decisions
- [ ] Prevention of duplicate approvals
- [ ] Segregation check (approver cannot be submitter)
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Unauthorized approval by non-approver | Low | Critical | Permission checks; role validation |
| Approver conflict (same person submitted) | Medium | High | SoD enforcement; block and flag |
| Duplicate approvals creating confusion | Medium | Medium | Idempotency keys; status checks |
| Delayed approval missing deadline | Medium | High | Deadline reminders; escalation alerts |
| Misunderstanding variance figures | Medium | Medium | Clear labeling; help text; comparison tooltips |
| Mobile approval UX issues | Medium | Low | Simplified mobile view; confirmation modals |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Payroll approval API and workflow | M | story-28-variable-pay-form |
| slice-b | Approval UI with variance display | M | slice-a |
| slice-c | Notifications and audit logging | S | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Payroll approval API with SoD checks
- [slice-b.md](./slice-b.md) — Approval UI with variance comparison
- [slice-c.md](./slice-c.md) — Notifications and audit logging
