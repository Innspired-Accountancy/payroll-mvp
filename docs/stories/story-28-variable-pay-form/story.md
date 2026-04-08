# Story: Variable Pay Data Input Form

**Epic:** epic-07-employer-portal
**Priority:** 3 of 8
**Sprint:** Sprint 4
**Dependencies:** story-26-client-auth, story-02-payroll-setup

## Goal

Enable employer admins to submit variable pay data (hours, overtime, bonuses) for employees in an open pay period. Include validation for National Minimum Wage compliance and reasonable limits, with save-as-draft and submit functionality.

## Acceptance Criteria

- [ ] Form displays all employees for the employer in the pay period
- [ ] Input fields: hours worked, overtime hours, bonus amount per employee
- [ ] Real-time validation with error messages
- [ ] National Minimum Wage compliance checking
- [ ] Save as draft functionality
- [ ] Submit final data with confirmation
- [ ] Bulk import from CSV/Excel option
- [ ] Visual indicators for incomplete entries
- [ ] Audit trail of who submitted and when
- [ ] Notification to bureau on submission
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| NMW compliance violations missed | Medium | High | Automated validation; flag below-minimum rates |
| Data loss during entry | Medium | High | Auto-save drafts; local storage backup |
| Concurrent edits by multiple users | Medium | Medium | Optimistic locking; last-write-wins with warning |
| Invalid data formats (text in number fields) | Low | Low | Input validation; type coercion with warnings |
| CSV import parsing errors | Medium | Medium | Template download; validation reports; row-level errors |
| Large employer performance issues | Medium | Medium | Virtual scrolling; pagination; debounced saves |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Variable pay API and validation | M | story-26-client-auth |
| slice-b | Data entry form UI | M | slice-a |
| slice-c | CSV import and bulk operations | M | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Variable pay submission API with validation
- [slice-b.md](./slice-b.md) — Data entry form components
- [slice-c.md](./slice-c.md) — CSV import and bulk operations
