# Story: Pension Scheme Setup and Configuration

**Epic:** epic-03-pension-ae
**Priority:** 1 of 8
**Sprint:** Month 3
**Dependencies:** epic-01-core-payroll (employer, employee data)

---

## Goal

Enable bureau administrators to configure pension schemes per employer with provider-specific settings (NEST initially), contribution rates, earnings basis, and relief method. This is the foundation for all auto-enrolment functionality.

---

## Acceptance Criteria

- [ ] Create pension scheme with provider selection (NEST supported first)
- [ ] Configure employer and employee contribution rates (percentage or fixed amount)
- [ ] Set earnings basis (qualifying earnings, banded earnings, or total earnings)
- [ ] Configure tax relief method (relief at source or net pay arrangement)
- [ ] Store provider-specific credentials and API configuration securely
- [ ] Edit scheme configuration with audit trail
- [ ] Associate multiple schemes per employer (for different worker groups)
- [ ] Set default scheme flag per employer
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| NEST API credentials exposure | Low | Critical | Store in encrypted JSONB, env-based decryption keys |
| Invalid contribution rate combinations | Medium | High | Validation rules, min/max bounds enforcement |
| Multiple default schemes per employer | Low | Medium | Database constraint + application validation |
| Provider API schema changes | Medium | Medium | Versioned API config schema, migration strategy |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Pension scheme data model and API | S | None |
| slice-b | Scheme configuration UI | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Pension scheme data model and tRPC API
- [slice-b.md](./slice-b.md) — Scheme configuration UI components
