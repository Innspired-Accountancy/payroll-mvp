# Slice b: Undergraduate Plan Deduction Calculator

**Story:** story-06-student-loans
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement student loan deduction calculation for Plans 1, 2, 4, and 5 with threshold checks and pro-rata calculations.

---

## Decision Checklist

- [x] Algorithm: (Earnings - threshold) × 9%
- [x] Thresholds: Pro-rata per pay frequency
- [x] Multiple plans: Calculate each independently
- [x] Rounding: Round down to nearest pound
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — Student loan deduction method
- 02-01-core-payroll-spec.md — Student loans

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/student-loans.ts` | create | Student loan calculator |
| `src/tests/calculations/student-loans.test.ts` | create | Calculation tests |

---

## Responsibilities

1. Calculate deduction for each active plan
2. Apply pro-rata threshold for pay frequency
3. Deduct 9% on earnings above threshold
4. Handle multiple plans concurrently
5. Return breakdown per plan

---

## Contracts

### calculateStudentLoanDeductions
- **Method:** `calculateStudentLoanDeductions(input: StudentLoanInput): StudentLoanResult`
- **Input:** StudentLoanInput with earnings and plans
- **Output:** StudentLoanResult with deductions per plan

### StudentLoanInput
| Field | Type | Description |
|-------|------|-------------|
| gross_pay | Decimal | Gross pay for period |
| active_plans | StudentLoanPlan[] | Array of active plans |
| frequency | enum | weekly, monthly |
| tax_year | string | Tax year for thresholds |

### StudentLoanResult
| Field | Type | Description |
|-------|------|-------------|
| total_deduction | Decimal | Sum of all deductions |
| plan_deductions | PlanDeduction[] | Breakdown per plan |

### PlanDeduction
| Field | Type | Description |
|-------|------|-------------|
| plan_type | enum | plan_1, plan_2, plan_4, plan_5, pgl |
| deduction | Decimal | Amount deducted |
| threshold | Decimal | Threshold applied |

---

## Business Rules & Invariants

1. Deduction = (gross_pay - threshold) × 0.09
2. If gross_pay <= threshold, deduction = 0
3. Each plan calculated independently with own threshold
4. Round down to nearest pound for final amount
5. Report on FPS with correct plan type indicator

---

## Edge Cases

1. **Earnings exactly at threshold** — Zero deduction
2. **Multiple plans** — Sum all applicable deductions
3. **Plan 1 + Plan 2** — Both deducted if above respective thresholds
4. **Week 53** — Use same threshold as Week 52

---

## Tests

### student-loans.test.ts
- Plan 1 deduction above threshold
- Plan 2 deduction above threshold
- Below threshold - zero deduction
- Exactly at threshold - zero deduction
- Multiple plans active
- Plan 1 + Plan 2 both deducted
- Pro-rata weekly threshold
- Rounding down to nearest pound

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
