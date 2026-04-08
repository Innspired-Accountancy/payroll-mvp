# Story: FPS Corrections and Resubmission

**Epic:** epic-02-hmrc-submissions
**Priority:** 8 of 9
**Sprint:** Month 3
**Dependencies:** story-05-error-handling

---

## Goal

Support FPS corrections for errors discovered after submission. Generate correction FPS with proper late reporting reason codes, maintain submission chain linking original to correction, and handle both current and prior tax year corrections per HMRC rules.

---

## Acceptance Criteria

- [ ] Identify correctable FPS submissions (not yet accepted or current tax year)
- [ ] Generate correction FPS with incremented version number
- [ ] Include late reporting reason code (H-M) for corrections after 19th
- [ ] Link correction to original submission via parent_submission_id
- [ ] Copy all employees from original with corrections applied
- [ ] Support prior tax year corrections (6 April onwards)
- [ ] Display correction chain in UI (original → corrections)
- [ ] Validate correction FPS before submission
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| HMRC correction rules complexity | Medium | High | Clear documentation, HMRC guidance reference, automated rule checks |
| Prior year correction timing | Low | High | Tax year awareness, 6 April validation, clear user messaging |
| Correction chain integrity | Medium | High | Database constraints, audit logging, immutable original records |
| Late reason code selection | Medium | Medium | Auto-suggest based on date, manual override with validation |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Correction eligibility and chain management | M | None |
| slice-b | Correction FPS generation logic | M | slice-a |
| slice-c | Correction UI and workflow | M | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Correction eligibility engine and chain tracking
- [slice-b.md](./slice-b.md) — Correction FPS generation with late reason codes
- [slice-c.md](./slice-c.md) — Correction workflow UI and history view
