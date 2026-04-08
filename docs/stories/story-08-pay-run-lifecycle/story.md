# Story: Pay Run Lifecycle

**Epic:** epic-01-core-payroll
**Priority:** 8 of 11
**Sprint:** Month 3
**Dependencies:** story-03-tax-engine, story-04-nic-calculator

---

## Goal

Implement the complete pay run workflow from draft creation through calculation, review, approval, and finalization. Support variance reporting, approval workflows, and calculation versioning with immutable snapshots on approval.

---

## Acceptance Criteria

- [ ] Create draft pay run from pay period with all active employees
- [ ] Calculate payroll with tax, NIC, statutory payments, deductions
- [ ] Generate variance report against previous period
- [ ] Display warnings (NMW check, negative net, missing data)
- [ ] Support review state with comment/annotation
- [ ] Approve pay run with snapshot hash and immutability
- [ ] Reopen approved pay run with authorization and reason capture
- [ ] Create calculation versions on reopen (preserve original)
- [ ] Trigger downstream events on approval (FPS, pension, payments)
- [ ] All tests pass

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Data corruption on reopen | Low | Critical | Immutable snapshots, versioned calculations |
| Concurrent modification | Medium | High | Row-level locking, optimistic concurrency |
| Downstream dependency failure | Medium | High | Event retry, dead letter queue, manual retry |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Pay run data model and state machine | M | None |
| slice-b | Payroll calculation orchestrator | L | slice-a |
| slice-c | Variance reporting engine | M | slice-b |
| slice-d | Approval workflow and snapshots | M | slice-b |
| slice-e | Reopen and versioning | M | slice-d |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Pay run model and status state machine
- [slice-b.md](./slice-b.md) — Payroll calculation orchestration
- [slice-c.md](./slice-c.md) — Variance reporting engine
- [slice-d.md](./slice-d.md) — Approval workflow with snapshots
- [slice-e.md](./slice-e.md) — Reopen authorization and versioning
