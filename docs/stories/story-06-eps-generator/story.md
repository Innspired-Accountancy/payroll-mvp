# Story: EPS Generator

**Epic:** epic-02-hmrc-submissions
**Priority:** 6 of 9
**Sprint:** Month 2
**Dependencies:** story-01-fps-generator (shared XML infrastructure)

---

## Goal

Generate Employer Payment Summary (EPS) XML for statutory recovery claims, Employment Allowance declarations, and other PAYE adjustments that cannot be included in FPS. Support monthly EPS submissions by the 19th deadline.

---

## Acceptance Criteria

- [ ] Generate EPS XML for statutory payments recovery (SSP, SMP, SPP, SAP)
- [ ] Generate EPS for NIC compensation claims
- [ ] Generate EPS for Employment Allowance claims
- [ ] Generate EPS for CIS deductions suffered
- [ ] Include correct tax year and tax month identifiers
- [ ] Calculate recovery amounts from payroll data
- [ ] Support "no payment to declare" submissions
- [ ] Validate EPS against HMRC schema
- [ ] Store EPS with draft status before submission
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Recovery calculation errors | Medium | High | Cross-validate against statutory payment records, audit trail |
| Employment Allowance eligibility | Medium | High | Eligibility checks before generation, warnings for ineligible claims |
| Monthly vs annual recovery confusion | Medium | Medium | Clear UI labeling, tax month auto-detection, validation rules |
| EPS/FPS overlap rules | Low | High | HMRC rules implementation, prevent duplicate recovery claims |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | EPS data structures and recovery calculations | M | None |
| slice-b | EPS XML generation engine | L | slice-a |
| slice-c | EPS validation and API | M | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — EPS data model and recovery calculations
- [slice-b.md](./slice-b.md) — EPS XML template and generation engine
- [slice-c.md](./slice-c.md) — EPS validation and submission API
