# Slice a: Pay Run Model and State Machine

**Story:** story-08-pay-run-lifecycle
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** None

---

## Goal

Create pay run data model with status state machine (draft → calculated → reviewed → approved → finalised → reopened).

---

## Decision Checklist

- [x] States: draft, calculated, reviewed, approved, finalised, reopened
- [x] Transitions: Valid state change rules
- [x] Storage: PayRun table with status, version
- [x] Audit: Who changed state when
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — PayRun entity
- 02-10-audit-compliance-spec.md — State change audit

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/pay-runs.ts` | create | PayRun schema |
| `src/lib/state-machine/pay-run.ts` | create | State machine logic |
| `src/server/routers/pay-runs.ts` | create | tRPC procedures |
| `src/tests/pay-run-state.test.ts` | create | State machine tests |

---

## Responsibilities

1. Define PayRun model with status field
2. Implement state transition validation
3. Record state change history
4. Support versioning for reopened pay runs
5. Enforce business rules per state

---

## Contracts

### PayRunState
| State | Description | Allowed Transitions |
|-------|-------------|---------------------|
| draft | Initial state | calculated |
| calculated | Computed | reviewed, draft |
| reviewed | Under review | approved, calculated |
| approved | Locked | finalised, reopened |
| finalised | Immutable | none |
| reopened | Was approved | calculated |

### payRuns.updateStatus
- **Method:** tRPC mutation `payRuns.updateStatus`
- **Input:** `{ id: uuid, status: PayRunStatus, reason?: string }`
- **Output:** Updated pay run
- **Errors:** BAD_REQUEST (invalid transition), FORBIDDEN (permissions)

---

## Business Rules & Invariants

1. Only draft can be deleted
2. Approved requires approval authorization
3. Reopened requires reopen authorization with reason
4. Finalised is immutable
5. Version increments on reopen

---

## Edge Cases

1. **Concurrent status change** — Optimistic locking
2. **Invalid transition** — Return error with valid options
3. **Reopen finalised** — Not allowed, must be approved

---

## Tests

### pay-run-state.test.ts
- Valid state transitions
- Invalid transition blocked
- Approval authorization check
- Reopen with reason required
- Version increment on reopen

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Data Models → PayRun
- 02-10-audit-compliance-spec.md § Audit Trail
