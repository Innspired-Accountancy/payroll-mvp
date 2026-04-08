# Slice b: Relief Method Application

**Story:** story-04-contributions
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Implement tax relief method application - relief at source (RAS) and net pay arrangement - per scheme configuration.

---

## Decision Checklist

- [x] Relief at Source: Add 20% basic rate tax relief to employee contribution
- [x] Net Pay: Deduct before tax calculation (reduce taxable pay)
- [x] Tax relief calculation: Decimal.js precision, rounded to 2dp
- [x] Integration: RAS → contribution record, Net Pay → payroll calculation
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — relief_method field
- HMRC: "Tax relief on pension contributions"

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/reliefCalculator.ts` | create | Tax relief calculation |
| `src/lib/pension/reliefMethods.ts` | create | RAS and Net Pay implementations |
| `src/tests/pension/reliefCalculator.test.ts` | create | Relief calculation tests |

---

## Responsibilities

1. Calculate relief at source amount (20% of employee contribution)
2. Apply RAS relief to gross up contribution
3. Integrate net pay with payroll taxable pay calculation
4. Record relief method used on contribution
5. Handle higher rate taxpayers (RAS only - they claim via self-assessment)

---

## Contracts

### calculateReliefAtSource()
- **Method:** `calculateReliefAtSource(employeeContribution: Decimal): ReliefResult`
- **Returns:** { grossContribution, taxRelief, netDeduction }
- **Calculation:**
  - taxRelief = employeeContribution × 0.25 (20% of gross = 25% of net)
  - grossContribution = employeeContribution + taxRelief
  - netDeduction = employeeContribution

### applyNetPayArrangement()
- **Method:** `applyNetPayArrangement(taxablePay: Decimal, contribution: Decimal): AdjustedPay`
- **Returns:** { taxablePayAfterPension, pensionContribution }
- **Integration:** Called by payroll calculation before tax

### ReliefResult
| Field | Type | Description |
|-------|------|-------------|
| gross_contribution | Decimal | Total going to pension (with relief) |
| tax_relief | Decimal | Amount claimed by scheme |
| net_deduction | Decimal | Amount deducted from payslip |
| relief_method | enum | "relief_at_source" |

---

## Business Rules & Invariants

1. RAS: Employee contributes net amount, scheme claims 20% from HMRC
2. RAS gross-up: Multiply net by 1.25 (or divide by 0.8)
3. Net Pay: Pension deducted before tax, employee gets full tax relief at marginal rate
4. Higher rate taxpayers with RAS claim additional relief via self-assessment
5. Relief method determined by scheme configuration, not employee choice

---

## Edge Cases

1. **Zero employee contribution** — No relief calculated
2. **Employee non-taxpayer** — RAS still applies (scheme claims basic rate)
3. **Very small contribution** — Minimum 1p relief
4. **Scottish taxpayer** — Same 20% basic rate applies for RAS

---

## Tests

### reliefCalculator.test.ts
- RAS calculation for £100 contribution (£125 gross, £25 relief)
- Net Pay adjustment to taxable pay
- Zero contribution handling
- Small amount rounding

---

## Verification

```bash
npm run test:unit src/tests/pension/reliefCalculator.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-04-contributions/slice-a.md → Contribution calculation input
