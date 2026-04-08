# Story: Employee Leaver Process

**Epic:** epic-07-employer-portal
**Priority:** 6 of 8
**Sprint:** Sprint 5
**Dependencies:** story-26-client-auth, story-01-employee-mgmt

## Goal

Enable employer admins to process employee departures by capturing leaving date, final payment details, and P45 requirements. Ensure proper handling of final pay calculations and HMRC reporting obligations.

## Acceptance Criteria

- [ ] Leaver form with employee selection
- [ ] Leaving date capture
- [ ] Final pay period determination
- [ ] Outstanding holiday/pay capture
- [ ] P45 required checkbox
- [ ] Reason for leaving (optional)
- [ ] Final payment instructions
- [ ] Validation for past/future dates
- [ ] Confirmation of final payroll inclusion
- [ ] Notification to bureau for P45 generation
- [ ] Employee status update to "leaving"
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Wrong leaving date affecting tax year | Medium | High | Date validation; tax year boundary warnings |
| Missing final pay adjustments | Medium | Medium | Checklist; holiday pay calculator hint |
| Duplicate leaver submissions | Low | Medium | Idempotency; status checking |
| Leaving date before hire date | Low | High | Validation rule; error message |
| P45 not generated in time | Medium | Medium | Automated bureau notification; SLA tracking |
| Access revocation timing | Medium | Medium | Coordination with employee portal deactivation |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Leaver form API and processing | S | story-26-client-auth |
| slice-b | Leaver form UI and validation | S | slice-a |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Leaver form API and final pay handling
- [slice-b.md](./slice-b.md) — Leaver form UI components
