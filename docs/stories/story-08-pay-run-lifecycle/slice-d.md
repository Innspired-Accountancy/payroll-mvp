# Slice d: Approval Workflow with Snapshots

**Story:** story-08-pay-run-lifecycle
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement pay run approval workflow with snapshot hash generation for immutability and audit.

---

## Decision Checklist

- [x] Authorization: Approval role required
- [x] Snapshot: Hash of all calculation data
- [x] Immutability: Data locked after approval
- [x] Audit: Who approved, when, notes
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Approval workflow
- 02-10-audit-compliance-spec.md — Audit requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/snapshots/pay-run.ts` | create | Snapshot generator |
| `src/server/routers/approve-pay-run.ts` | create | Approval tRPC |
| `src/components/pay-run/approval-panel.tsx` | create | Approval UI |
| `src/tests/approval.test.ts` | create | Approval tests |

---

## Responsibilities

1. Verify approval authorization
2. Generate snapshot hash of pay run data
3. Lock pay run data from modification
4. Record approval with timestamp and user
5. Trigger downstream events

---

## Contracts

### payRuns.approve
- **Method:** tRPC mutation `payRuns.approve`
- **Input:** `{ id: uuid, notes?: string }`
- **Output:** Approved pay run with snapshot
- **Errors:** FORBIDDEN (no permission), BAD_REQUEST (not in reviewed state)

### SnapshotData
| Field | Type | Description |
|-------|------|-------------|
| pay_run_id | uuid | Pay run FK |
| snapshot_hash | string | SHA-256 hash |
| snapshot_data | JSON | Full calculation data |
| created_at | DateTime | Snapshot timestamp |

### ApprovalRecord
| Field | Type | Description |
|-------|------|-------------|
| pay_run_id | uuid | Pay run FK |
| approved_by | uuid | User FK |
| approved_at | DateTime | Approval timestamp |
| notes | string | Optional notes |
| snapshot_hash | string | Hash at approval |

---

## Business Rules & Invariants

1. Only users with approve:payroll permission can approve
2. Pay run must be in reviewed state to approve
3. Snapshot hash covers all payslips and calculations
4. Approved pay runs cannot be modified (must reopen)
5. Approval triggers FPS, pension, payment events

---

## Edge Cases

1. **Concurrent approval attempt** — First wins, second fails
2. **Approval after changes** — Recalculate before approval
3. **Downstream failure** — Log but don't rollback approval

---

## Tests

### approval.test.ts
- Approve with valid permission
- Reject approval without permission
- Snapshot hash generation
- Data locked after approval
- Approval with notes

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Approval Workflow
- 02-10-audit-compliance-spec.md § Audit Trail
