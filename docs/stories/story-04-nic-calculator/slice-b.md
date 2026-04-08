# Slice b: Employee NIC Calculator

**Story:** story-04-nic-calculator
**Epic:** epic-01-core-payroll
**Effort:** L
**Dependencies:** slice-a

---

## Goal

Implement the core employee NIC calculation using HMRC exact percentage method with correct threshold application and category-specific rates.

---

## Decision Checklist

- [x] Algorithm: HMRC exact percentage method
- [x] Data types: Decimal.js for money, config-driven rates
- [x] Input: NicCalculationInput with earnings, category, frequency
- [x] Output: NicCalculationResult with employee NIC amount
- [x] Error handling: Zod validation, invalid category error
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — NIC calculation methodology
- 02-01-core-payroll-spec.md — NIC calculation requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/nic.ts` | create | NIC calculation engine |
| `src/lib/calculations/nic-employee.ts` | create | Employee NIC calculator |
| `src/lib/types/calculations.ts` | update | Add NIC types |
| `src/tests/calculations/nic-employee.test.ts` | create | Employee NIC tests |

---

## Responsibilities

1. Calculate earnings between thresholds
2. Apply employee rates per category
3. Handle pro-rata thresholds for pay frequency
4. Support all employee categories (A, B, C, H, J, M, Z, X)
5. Return detailed calculation trace

---

## Contracts

### calculateEmployeeNic
- **Method:** `calculateEmployeeNic(input: NicCalculationInput): NicCalculationResult`
- **Input:** NicCalculationInput
- **Output:** NicCalculationResult with nicEmployee amount

### NicCalculationInput
| Field | Type | Description |
|-------|------|-------------|
| gross_pay | Decimal | Gross pay for period |
| category | enum | NIC category letter |
| frequency | enum | weekly, monthly, annual |
| tax_year | string | Tax year config to use |
| is_director | boolean | Director calculation flag |

### NicCalculationResult
| Field | Type | Description |
|-------|------|-------------|
| nic_employee | Decimal | Employee NIC due |
| nicable_earnings | Decimal | Earnings subject to NIC |
| trace | NicTraceStep[] | Calculation steps |

---

## Business Rules & Invariants

1. No NIC on earnings below LEL
2. Employee NIC = (earnings in band) × rate for each band
3. Category X = zero NIC
4. Pro-rata thresholds: weekly = annual ÷ 52, monthly = annual ÷ 12
5. Round NIC to nearest penny (0.5 rounds up)

---

## Edge Cases

1. **Earnings exactly on threshold** — Include in upper band
2. **Zero gross pay** — Return zero NIC
3. **Invalid category** — Throw validation error
4. **Negative earnings** — Return zero NIC (no negative contributions)

---

## Tests

### nic-employee.test.ts
- Category A monthly calculation
- Category C (over pension age) - zero employee NIC
- Category M (under 21) - zero employee NIC
- Category H (apprentice) - zero employee NIC
- Earnings below LEL - zero NIC
- Earnings between LEL and PT - zero NIC
- Earnings above UEL - multiple bands
- Pro-rata weekly calculation
- Round half up correctly

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § Calculating NICs
- 02-01-core-payroll-spec.md § NIC Calculation
