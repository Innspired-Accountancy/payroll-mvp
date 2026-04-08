# Story: FPS XML Generator

**Epic:** epic-02-hmrc-submissions
**Priority:** 1 of 9
**Sprint:** Month 2
**Dependencies:** epic-01-core-payroll (pay run calculation complete)

---

## Goal

Generate valid RTI Full Payment Submission (FPS) XML documents from approved pay run data. The FPS captures all employee payments, deductions, and statutory items for a specific pay period and PAYE scheme, formatted according to HMRC RTI schema v2025-26.

---

## Acceptance Criteria

- [ ] Generate FPS XML from pay run data with all required RTI elements
- [ ] Include correct Employer PAYE Reference, Accounts Office Reference, and Contact details
- [ ] Include all employees with taxable payments in the pay period
- [ ] Calculate and include correct totals (taxable pay, tax deducted, NICs, student loans)
- [ ] Format dates and monetary values per HMRC specification
- [ ] Generate valid XML structure matching HMRC RTI schema v2025-26
- [ ] Store generated FPS in database with draft status
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| HMRC schema changes | Low | High | Monitor HMRC schema updates, make schema version configurable |
| Missing employee data fields | Medium | High | Validate pay run completeness before generation, clear error messages |
| Date formatting edge cases | Medium | Medium | Use XML date types, comprehensive test cases for fiscal year boundaries |
| Large pay runs causing timeouts | Medium | Medium | Implement streaming generation, progress tracking for 500+ employees |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | FPS data model and database schema | S | None |
| slice-b | XML generation engine with templates | L | slice-a |
| slice-c | Pay run data aggregation logic | M | slice-b |
| slice-d | FPS generation API endpoint | M | slice-c |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — HmrcSubmission and related data models
- [slice-b.md](./slice-b.md) — XML template engine and FPS structure builder
- [slice-c.md](./slice-c.md) — Pay run aggregation and calculation logic
- [slice-d.md](./slice-d.md) — tRPC procedures for FPS generation
