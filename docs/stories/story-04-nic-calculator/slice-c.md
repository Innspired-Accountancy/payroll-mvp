# Slice c: Employer NIC Calculator

**Story:** story-04-nic-calculator
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement employer NIC calculation with secondary thresholds and category-specific employer rates, including Employment Allowance consideration.

---

## Decision Checklist

- [x] Algorithm: HMRC employer NIC calculation
- [x] Data types: Decimal.js for money amounts
- [x] Thresholds: ST (Secondary Threshold), UEL from config
- [x] Output: employer_nic amount with calculation trace
- [x] Error handling: Validation, invalid category
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — Employer NIC calculation
- 02-01-core-payroll-spec.md — NIC requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/nic-employer.ts` | create | Employer NIC calculator |
| `src/tests/calculations/nic-employer.test.ts` | create | Employer NIC tests |

---

## Responsibilities

1. Calculate employer NIC based on secondary threshold
2. Apply employer rates per category
3. Handle Employment Allowance offset (basic support)
4. Support all categories including director calculations
5. Return detailed calculation trace

---

## Contracts

### calculateEmployerNic
- **Method:** `calculateEmployerNic(input: NicCalculationInput): EmployerNicResult`
- **Input:** NicCalculationInput (same as employee)
- **Output:** EmployerNicResult with employer NIC amount

### EmployerNicResult
| Field | Type | Description |
|-------|------|-------------|
| nic_employer | Decimal | Employer NIC due |
| secondary_earnings | Decimal | Earnings above ST |
| trace | NicTraceStep[] | Calculation steps |

---

## Business Rules & Invariants

1. Employer NIC starts at ST (Secondary Threshold)
2. No employer NIC for categories C, X (over SPA, no liability)
3. Apprentice under 25: zero employer NIC (category H)
4. Employee under 21: zero employer NIC (category M, Z)
5. Standard employer rate: 13.8% above ST

---

## Edge Cases

1. **Employment Allowance** — Reduce employer NIC to zero up to £5,000 (basic implementation)
2. **Director annual calculation** — Use annual thresholds
3. **Multiple employments** — Each calculated separately

---

## Tests

### nic-employer.test.ts
- Standard category A employer NIC
- Category C (over SPA) - zero employer NIC
- Category H (apprentice) - zero employer NIC
- Category M (under 21) - zero employer NIC
- Earnings below ST - zero employer NIC
- Earnings above UEL - continues at 13.8%
- Employment Allowance offset

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § Employer NICs
- 02-01-core-payroll-spec.md § NIC Calculation
