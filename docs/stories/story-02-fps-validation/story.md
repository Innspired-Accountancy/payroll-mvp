# Story: FPS Validation

**Epic:** epic-02-hmrc-submissions
**Priority:** 2 of 9
**Sprint:** Month 2
**Dependencies:** story-01-fps-generator

---

## Goal

Validate generated FPS XML against HMRC XSD schema and enforce business rules to catch errors before submission. Distinguish between blocking errors (prevent submission) and warnings (allow with acknowledgment). Provide clear, actionable feedback to payroll managers.

---

## Acceptance Criteria

- [ ] Validate FPS XML against HMRC RTI schema v2025-26 XSD
- [ ] Implement business rule validation (NI number format, tax code validity)
- [ ] Categorize issues as blockers (prevent submission) or warnings (allow)
- [ ] Link validation errors to specific employees and fields
- [ ] Display validation results in UI with clear error messages
- [ ] Prevent submission when blockers exist
- [ ] Log all validation results for audit trail
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| XSD validation performance | Medium | Medium | Implement streaming validation, caching for repeated validations |
| False positive warnings | Medium | Medium | Calibrate business rules with real data, allow suppression |
| Complex rule interdependencies | Medium | Medium | Document rules clearly, unit test each independently |
| HMRC schema ambiguity | Low | High | Reference HMRC guidance documents, implement conservative validation |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | XSD schema validation engine | M | None |
| slice-b | Business rules validation layer | M | slice-a |
| slice-c | Validation results API and UI | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — XSD validation with libxml2
- [slice-b.md](./slice-b.md) — Business rules engine and error classification
- [slice-c.md](./slice-c.md) — Validation results endpoint and display components
