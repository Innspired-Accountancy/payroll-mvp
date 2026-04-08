# Slice a: Opt-Out Recording and Validation

**Story:** story-06-opt-out
**Epic:** epic-03-pension-ae
**Effort:** M
**Dependencies:** story-03-enrolment-workflow

---

## Goal

Implement opt-out recording with window validation, status updates, and compliance record keeping.

---

## Decision Checklist

- [x] Window check: current_date <= opt_out_deadline
- [x] Evidence capture: Opt-out date, method, reference, notes
- [x] Status transition: enrolled → opted_out
- [x] Exclusion tracking: 12-month re-enrolment exclusion
- [x] Retention: 4-year record retention per TPR
- [x] Methods supported: Online portal, paper form, phone
- [x] Error scenarios: Window closed, already opted out, not enrolled
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Journey 3, Entity: PensionEnrolment
- The Pensions Regulator: "Opting out of workplace pensions"

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/optOutService.ts` | create | Opt-out recording logic |
| `src/lib/db/schema/optOutRecords.ts` | create | Opt-out record table |
| `src/server/routers/optOuts.ts` | create | tRPC opt-out procedures |
| `src/lib/validation/optOut.ts` | create | Opt-out validation schemas |
| `src/tests/pension/optOutService.test.ts` | create | Opt-out tests |

---

## Responsibilities

1. Validate opt-out request is within window
2. Record opt-out details (date, method, evidence)
3. Update enrolment status to opted_out
4. Set 12-month re-enrolment exclusion
5. Trigger refund calculation
6. Trigger confirmation communication
7. Notify NEST of opt-out

---

## Contracts

### recordOptOut()
- **Method:** `recordOptOut(input: OptOutInput): Promise<OptOutResult>`
- **Input:** OptOutInput Zod schema
- **Output:** OptOutResult with refund details
- **Validation:**
  1. Enrolment exists and is active
  2. Opt-out window is open
  3. Not already opted out

### OptOutInput Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| enrolment_id | UUID | Yes | Enrolment to opt out of |
| opt_out_date | Date | Yes | Date of opt-out |
| opt_out_method | enum | Yes | "online" / "paper" / "phone" |
| evidence_reference | string | No | Form reference number |
| notes | string | No | Additional notes |
| processed_by | UUID | No | User who processed |

### OptOutResult
| Field | Type | Description |
|-------|------|-------------|
| opt_out_id | UUID | Created opt-out record |
| enrolment_status | enum | "opted_out" |
| refund_due | Decimal | Calculated refund amount |
| refund_period | string | Pay period for refund |
| exclusion_until | Date | 12 months from opt-out |
| confirmation_queued | boolean | Whether letter queued |

### pensionEnrolments.recordOptOut
- **Method:** tRPC mutation `pensionEnrolments.recordOptOut`
- **Input:** OptOutInput
- **Output:** OptOutResult
- **Auth:** Requires pension:optout:create permission

---

## Business Rules & Invariants

1. Opt-out only allowed during 1-month window from enrolment
2. Opt-out date cannot be before enrolment date
3. Opt-out date cannot be in the future
4. Employee contributions refunded, employer contributions not refunded
5. 12-month exclusion from automatic re-enrolment
6. Opt-out records retained for 4 years
7. Confirmation letter sent within 1 week of opt-out

---

## Edge Cases

1. **Opt-out on last day of window** — Valid
2. **Opt-out 1 day after window** — Rejected with clear message
3. **Already opted out** — Return existing opt-out details
4. **Never enrolled** — Error, no enrolment found
5. **Future opt-out date** — Rejected, must be today or past

---

## Tests

### optOutService.test.ts
- Valid opt-out within window
- Opt-out rejected after window closes
- Opt-out on exact deadline
- Duplicate opt-out handling
- Future date rejection
- Status transition to opted_out

---

## Verification

```bash
cd "/Users/josephstephenson-mouzo/Projects/03 - development/16 - payroll mvp"
npm run test:unit src/tests/pension/optOutService.test.ts
npm run typecheck
npm run lint
npm run build
```

---

## Source Sections

- 02-03-pension-auto-enrolment-spec.md § Journey 3 → Opt-out flow
- story-03-enrolment-workflow → Enrolment status management
