# Slice d: SLC Stop Notice Handling

**Story:** story-06-student-loans
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-b, slice-c

---

## Goal

Implement SLC stop notice handling to cease student loan deductions when notified by Student Loans Company.

---

## Decision Checklist

- [x] Stop codes: SD1, SD2, SD4, SD5, PGL defined
- [x] Effective date: Immediate or specified date
- [x] Storage: Stop notice records linked to employee
- [x] UI: Manual entry of stop notices
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — Student loan stop notices
- SLC employer guidance — Stop notice process
- 02-01-core-payroll-spec.md — Student loans

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/student-loan-stops.ts` | create | Stop notice tRPC |
| `src/lib/db/schema/student-loan-stops.ts` | create | Stop notice schema |
| `src/components/student-loans/stop-notice-form.tsx` | create | UI for entry |
| `src/tests/student-loan-stops.test.ts` | create | Stop notice tests |

---

## Responsibilities

1. Record stop notices with code and effective date
2. Cease deductions from effective date
3. Support manual entry of stop notices
4. Display stop notice history
5. Report stop on FPS

---

## Contracts

### studentLoanStops.create
- **Method:** tRPC mutation `studentLoanStops.create`
- **Input:** StopNoticeCreateInput
- **Output:** Stop notice record

### StopNoticeCreateInput
| Field | Type | Description |
|-------|------|-------------|
| employee_id | uuid | Employee FK |
| stop_code | enum | SD1, SD2, SD4, SD5, PGL |
| effective_date | Date | Date to stop deductions |
| received_date | Date | Date notice received |
| notes | string | Optional notes |

### StopNotice
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| employee_id | uuid | Employee FK |
| stop_code | enum | Stop code applied |
| effective_date | Date | Deductions stopped from |
| is_active | boolean | Still active or superseded |

---

## Business Rules & Invariants

1. Stop notice takes effect from effective date
2. New stop notice can supersede previous
3. Employee may receive new loan notice after stop
4. Stop must be reported on next FPS

---

## Edge Cases

1. **Stop effective mid-period** — Stop from next period
2. **New loan after stop** — Allow new plan registration
3. **Wrong stop code** — Validate against employee's active plans

---

## Tests

### student-loan-stops.test.ts
- Create stop notice
- Deductions cease from effective date
- Stop notice validation
- Supersede previous stop
- New loan after stop

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § Stop notices
- 02-01-core-payroll-spec.md § Student Loans
