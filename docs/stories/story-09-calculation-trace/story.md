# Story: Calculation Trace and Explainability

**Epic:** epic-01-core-payroll
**Priority:** 9 of 11
**Sprint:** Month 3
**Dependencies:** story-08-pay-run-lifecycle

---

## Goal

Provide full transparency into payroll calculations with step-by-step trace showing all inputs, formulas, intermediate values, and final results. Enable "explain this calculation" feature for payslips with user-friendly display.

---

## Acceptance Criteria

- [ ] Capture every calculation step with formula, inputs, and outputs
- [ ] Store complete trace in JSONB with structured format
- [ ] Display trace in hierarchical view (sections: earnings, tax, NIC, deductions)
- [ ] Show threshold values and how they were applied
- [ ] Link trace values to source data (employee, pay elements, YTD)
- [ ] Export trace as PDF for customer support
- [ ] Query trace via API for integration testing
- [ ] Performance: trace generation adds <10% to calculation time
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Trace size bloat | Medium | Medium | Compression, selective storage, retention policy |
| Performance overhead | Low | Medium | Benchmark testing, async trace generation option |
| Complex nested calculation display | Medium | Low | Hierarchical UI, collapsible sections |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Trace data structure and capture | S | None |
| slice-b | Trace generation in calculators | S | slice-a |
| slice-c | Trace API and storage | S | slice-a |
| slice-d | Explain calculation UI | M | slice-b, slice-c |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Trace data structure definition
- [slice-b.md](./slice-b.md) — Trace generation in calculation engines
- [slice-c.md](./slice-c.md) — Trace storage and retrieval API
- [slice-d.md](./slice-d.md) — Explain calculation UI component
