# Slice e: Leave Tracking and History

**Story:** story-05-statutory-payments
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a, slice-b, slice-c

---

## Goal

Implement leave tracking and payment history for statutory payments with carry-forward and state management.

---

## Decision Checklist

- [x] Storage: Database records for each leave type
- [x] State: Active, completed, cancelled statuses
- [x] History: Payment records per period
- [x] Carry-forward: YTD statutory pay tracking
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Statutory payments tracking
- 02-10-audit-compliance-spec.md — Audit trail requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/leave.ts` | create | Leave management tRPC |
| `src/components/leave/leave-manager.tsx` | create | Leave UI |
| `src/components/leave/leave-history.tsx` | create | History view |
| `src/tests/leave.test.ts` | create | Leave tracking tests |

---

## Responsibilities

1. Track leave start/end dates
2. Record statutory payments per period
3. Carry forward YTD statutory pay amounts
4. Support leave cancellation/adjustment
5. Display leave history and remaining entitlement

---

## Contracts

### leave.create
- **Method:** tRPC mutation `leave.create`
- **Input:** LeaveCreateInput
- **Output:** Leave record

### LeaveCreateInput
| Field | Type | Description |
|-------|------|-------------|
| employee_id | uuid | Employee FK |
| type | enum | sick, maternity, paternity |
| start_date | Date | Leave start |
| expected_end_date | Date | Expected end |
| qualifying_days | string[] | For SSP: days worked |

### LeavePaymentRecord
| Field | Type | Description |
|-------|------|-------------|
| leave_id | uuid | FK to leave |
| pay_period_id | uuid | FK to period |
| amount | Decimal | Payment amount |
| payment_type | enum | ssp, smp, spp |
| weeks_paid | int | Week count |

---

## Business Rules & Invariants

1. Only one active statutory leave per type at a time
2. YTD statutory pay tracked separately per type
3. Leave history immutable once paid
4. Adjustments create new records, don't modify history

---

## Edge Cases

1. **Leave extension** — Update expected end date
2. **Early return** — Mark actual end date, stop payments
3. **Overlapping leaves** — Block or warn
4. **Backdated leave** — Historical adjustment process

---

## Tests

### leave.test.ts
- Create leave record
- Record payment against leave
- YTD accumulation
- Leave cancellation
- Extension handling

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Statutory Payments
- 02-10-audit-compliance-spec.md § Audit Trail
