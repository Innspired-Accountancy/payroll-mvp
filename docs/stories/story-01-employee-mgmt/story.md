# Story: Employee Management

**Epic:** epic-01-core-payroll
**Priority:** 1 of 11
**Sprint:** Month 1
**Dependencies:** epic-05-identity-access (user authentication)

---

## Goal

Enable CRUD operations for employees and their employment records with effective dating for all payroll-relevant changes. This is the foundation for all payroll processing.

---

## Acceptance Criteria

- [ ] Create employee with all required fields (name, DOB, NI, address, start date)
- [ ] Support multiple employments per person under same employer
- [ ] Edit employee with before/after audit trail
- [ ] Terminate employee with leaving date and P45 generation trigger
- [ ] View employee history (all changes over time)
- [ ] All changes audit-logged with timestamp and user
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| NI number validation edge cases | Medium | Medium | HMRC validation rules research, comprehensive test cases |
| Duplicate employee detection | Medium | High | Matching algorithm, manual merge workflow |
| Historical data import issues | Medium | Medium | Validation on import, error reporting |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Employee data model and API | M | None |
| slice-b | Employment records and effective dating | M | slice-a |
| slice-c | Audit logging for all changes | S | slice-a |
| slice-d | Employee UI (list, create, edit) | M | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Employee data model and REST API
- [slice-b.md](./slice-b.md) — Employment records with effective dating
- [slice-c.md](./slice-c.md) — Audit logging integration
- [slice-d.md](./slice-d.md) — Employee management UI
