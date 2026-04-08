# Story: Error Handling and User Guidance

**Epic:** epic-02-hmrc-submissions
**Priority:** 5 of 9
**Sprint:** Month 2
**Dependencies:** story-04-polling

---

## Goal

Parse HMRC rejection responses, map error codes to specific employees and fields, and provide actionable guidance to payroll managers for correcting issues. Maintain submission chain for audit and support resubmission workflows.

---

## Acceptance Criteria

- [ ] Parse HMRC rejection XML and extract error codes
- [ ] Map errors to specific employees (where applicable)
- [ ] Map errors to specific fields using XPath references
- [ ] Classify errors as correctable vs permanent rejection
- [ ] Provide actionable correction guidance for each error type
- [ ] Display errors in UI linked to employee records
- [ ] Link rejection to original submission for audit trail
- [ ] Support correction workflow (update data, regenerate, resubmit)
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Unknown HMRC error codes | Medium | Medium | Generic fallback handling, manual mapping process, HMRC documentation |
| Complex error-to-field mapping | Medium | Medium | Maintain mapping table, XPath library, manual override capability |
| User confusion on corrections | Medium | High | Clear UI guidance, step-by-step correction wizard, help text |
| Error state persistence | Low | High | Atomic updates, transaction wrapping, audit logging |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Error parser and classification engine | M | None |
| slice-b | Error-to-employee/field mapping | M | slice-a |
| slice-c | Correction workflow UI and API | M | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — HMRC error parser and classification
- [slice-b.md](./slice-b.md) — Error mapping to payroll data fields
- [slice-c.md](./slice-c.md) — Correction workflow and resubmission logic
