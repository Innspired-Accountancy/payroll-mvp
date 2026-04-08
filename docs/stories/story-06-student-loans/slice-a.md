# Slice a: Student Loan Plan Configuration

**Story:** story-06-student-loans
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** None

---

## Goal

Create student loan plan configuration with thresholds and rates for all plans (1, 2, 4, 5) and Postgraduate Loan (PGL).

---

## Decision Checklist

- [x] Plans: 1, 2, 4, 5, PGL defined with thresholds
- [x] Rates: 9% for plans, 6% for PGL
- [x] Thresholds: Annual, pro-rata for pay frequency
- [x] Stop notices: SD1, SD2, PGL stop codes
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — Student loan deductions
- SLC guidance — Plan thresholds
- 02-01-core-payroll-spec.md — Student loans

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/config/student-loans/2026-27.ts` | create | Student loan config |
| `src/lib/types/student-loans.ts` | create | Type definitions |
| `src/lib/db/schema/student-loans.ts` | create | Schema for employee plans |
| `src/tests/student-loan-config.test.ts` | create | Config tests |

---

## Responsibilities

1. Define all student loan plans with thresholds
2. Configure deduction rates per plan
3. Support pro-rata thresholds
4. Handle stop notice codes
5. Version per tax year

---

## Contracts

### StudentLoanPlanConfig
| Plan | Annual Threshold | Rate |
|------|------------------|------|
| Plan 1 | £24,990 | 9% |
| Plan 2 | £27,295 | 9% |
| Plan 4 | £31,395 | 9% |
| Plan 5 | £25,000 | 9% |
| PGL | £21,000 | 6% |

### StopNoticeCodes
| Code | Description |
|------|-------------|
| SD1 | Stop Plan 1 deductions |
| SD2 | Stop Plan 2 deductions |
| SD4 | Stop Plan 4 deductions |
| SD5 | Stop Plan 5 deductions |
| PGL | Stop PGL deductions |

---

## Business Rules & Invariants

1. Multiple plans can be active simultaneously
2. Deduction applies only on earnings above threshold
3. Stop notice takes effect immediately
4. Plan type determined by SLC or employee declaration

---

## Edge Cases

1. **Multiple plans** — Deduct for each above respective threshold
2. **Plan change mid-year** — Update plan type, continue deductions
3. **Stop notice received** — Stop immediately, log reason

---

## Tests

### student-loan-config.test.ts
- All plan thresholds loaded
- Pro-rata calculations correct
- Stop codes recognized
- Multiple plan handling

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § Student loan deductions
- 02-01-core-payroll-spec.md § Student Loans
