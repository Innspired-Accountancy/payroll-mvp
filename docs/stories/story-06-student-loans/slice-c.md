# Slice c: Postgraduate Loan Calculator

**Story:** story-06-student-loans
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Implement Postgraduate Loan (PGL) deduction calculation at 6% rate with £21,000 threshold.

---

## Decision Checklist

- [x] Rate: 6% on earnings above threshold
- [x] Threshold: £21,000 annual (2026-27)
- [x] Combined: Can have PGL + undergraduate plan
- [x] Rounding: Round down to nearest pound
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — Postgraduate loan deductions
- 02-01-core-payroll-spec.md — Student loans

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/pgl.ts` | create | PGL calculator |
| `src/tests/calculations/pgl.test.ts` | create | PGL tests |

---

## Responsibilities

1. Calculate PGL deduction at 6% rate
2. Apply pro-rata threshold
3. Handle PGL + undergraduate plan combination
4. Return separate PGL amount
5. Report correctly on FPS

---

## Contracts

### calculatePglDeduction
- **Method:** `calculatePglDeduction(input: PglInput): PglResult`
- **Input:** PglInput with earnings
- **Output:** PglResult with PGL deduction

### PglInput
| Field | Type | Description |
|-------|------|-------------|
| gross_pay | Decimal | Gross pay for period |
| has_pgl | boolean | PGL active flag |
| frequency | enum | weekly, monthly |
| tax_year | string | Tax year for thresholds |

### PglResult
| Field | Type | Description |
|-------|------|-------------|
| pgl_deduction | Decimal | PGL amount |
| threshold | Decimal | Threshold applied |
| rate | decimal | 0.06 (6%) |

---

## Business Rules & Invariants

1. PGL rate = 6% (different from 9% for undergraduate)
2. Threshold = £21,000 annual
3. Can be combined with undergraduate plan
4. Deducted separately and reported separately on FPS
5. Round down to nearest pound

---

## Edge Cases

1. **PGL + Plan 2** — Both deducted, separate calculations
2. **Below threshold** — Zero PGL deduction
3. **Stop notice** — Cease deductions immediately

---

## Tests

### pgl.test.ts
- PGL deduction above threshold
- Below threshold - zero deduction
- PGL + Plan 2 combined
- Pro-rata monthly threshold
- Rounding down

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § Postgraduate loan deductions
- 02-01-core-payroll-spec.md § Student Loans
