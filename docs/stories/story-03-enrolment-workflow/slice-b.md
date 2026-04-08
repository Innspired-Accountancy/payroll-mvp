# Slice b: Enrolment Status Management

**Story:** story-03-enrolment-workflow
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Implement enrolment status lifecycle management including tracking opt-out windows, handling status transitions, and maintaining audit trail.

---

## Decision Checklist

- [x] Status values: enrolled, opted_out, opted_in, ceased
- [x] Status transitions: Valid state machine with defined transitions
- [x] Opt-out window: Computed field based on enrolment_date + 1 month
- [x] Ceased reasons: Left employment, scheme change, death
- [x] History tracking: All status changes logged with timestamp
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Entity: PensionEnrolment status field

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/enrolmentStatus.ts` | create | Status state machine and transitions |
| `src/lib/db/schema/pensionEnrolmentHistory.ts` | create | Status history table |
| `src/server/routers/pensionEnrolments.ts` | update | Add status query procedures |
| `src/tests/pension/enrolmentStatus.test.ts` | create | Status transition tests |

---

## Responsibilities

1. Define valid status values and transitions
2. Validate status changes against state machine
3. Track opt-out window (open/closed)
4. Record all status changes in history table
5. Provide status query methods
6. Handle enrolment cessation reasons

---

## Contracts

### EnrolmentStatus State Machine
- **enrolled** → opted_out (valid during opt-out window)
- **enrolled** → ceased (employment ended, etc.)
- **opted_out** → enrolled (re-enrolment after 12 months)
- **opted_in** → opted_out (within window)
- **opted_in** → ceased (employment ended)

### isOptOutWindowOpen()
- **Method:** `isOptOutWindowOpen(enrolment: PensionEnrolment): boolean`
- **Logic:** current_date <= opt_out_deadline
- **Returns:** True if opt-out is still allowed

### transitionStatus()
- **Method:** `transitionStatus(enrolmentId: UUID, newStatus: enum, reason?: string): Promise<void>`
- **Validation:** Check transition is valid in state machine
- **Side Effect:** Create history record
- **Errors:** INVALID_TRANSITION (not allowed), NOT_FOUND (enrolment)

### pensionEnrolments.getStatusHistory
- **Method:** tRPC query `pensionEnrolments.getStatusHistory`
- **Input:** `{ enrolmentId: UUID }`
- **Output:** Array of status history records
- **Auth:** Tenant-scoped read access

---

## Business Rules & Invariants

1. Opt-out only allowed during 1-month window from enrolment
2. Status transitions must follow defined state machine
3. All transitions recorded with user_id, timestamp, and reason
4. Ceased status requires a cessation reason
5. Re-enrolment allowed after 12-month exclusion period
6. Opted_out employees tracked for cyclical re-enrolment

---

## Edge Cases

1. **Opt-out on last day of window** — Valid, processed
2. **Opt-out after window closed** — Rejected with error
3. **Status change during payroll run** — Applied from next period
4. **Multiple status changes same day** — Each recorded separately
5. **Re-enrolment same day as opt-out** — Allowed after 12 months only

---

## Tests

### enrolmentStatus.test.ts
- Valid transition: enrolled → opted_out
- Invalid transition: opted_out → enrolled (within 12 months)
- Valid transition: opted_out → enrolled (after 12 months)
- Opt-out window open calculation
- Opt-out window closed rejection
- History tracking for all transitions

---

## Verification

```bash
npm run test:unit src/tests/pension/enrolmentStatus.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-03-enrolment-workflow/slice-a.md → Enrolment records to manage
