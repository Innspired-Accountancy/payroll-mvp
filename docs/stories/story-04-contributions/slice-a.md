# Slice a: Contribution Calculation Engine

**Story:** story-04-contributions
**Epic:** epic-03-pension-ae
**Effort:** M
**Dependencies:** story-02-assessment-engine

---

## Goal

Implement the core pension contribution calculation engine that computes employee and employer contributions based on qualifying earnings and scheme rates.

---

## Decision Checklist

- [x] Calculation basis: Qualifying earnings (banded) or total earnings per scheme
- [x] Math library: Decimal.js 10.4.x for precision
- [x] Precision: 4 decimal places for rates, 2 for money amounts
- [x] Rounding: Half-even (banker's rounding) for final amounts
- [x] Contribution types: Percentage of earnings or fixed amount
- [x] Certified schemes: Separate calculation path for schemes with certification
- [x] Error scenarios: Invalid rates, zero earnings, scheme not found
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Entity: PensionContribution
- HMRC: "Workplace pension contribution rates and thresholds"

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/contributionCalculator.ts` | create | Core calculation engine |
| `src/lib/db/schema/pensionContributions.ts` | create | Contribution table schema |
| `src/lib/pension/earningsBasis.ts` | create | Earnings basis calculation utilities |
| `src/lib/types/pensionContributions.ts` | create | TypeScript types |
| `src/tests/pension/contributionCalculator.test.ts` | create | Unit tests |

---

## Responsibilities

1. Determine earnings basis per scheme configuration
2. Calculate qualifying earnings (if applicable)
3. Apply employee contribution rate to earnings
4. Apply employer contribution rate to earnings
5. Round contributions to 2 decimal places
6. Calculate total contribution
7. Record contribution with full calculation trace

---

## Contracts

### calculateContributions()
- **Method:** `calculateContributions(input: ContributionInput): ContributionResult`
- **Input:** ContributionInput Zod schema
- **Output:** ContributionResult with all contribution amounts
- **Errors:** 
  - INVALID_RATE (negative or >100%)
  - SCHEME_NOT_FOUND
  - ZERO_EARNINGS (warning, not error)

### ContributionInput Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| enrolment_id | UUID | Yes | Pension enrolment |
| gross_earnings | Decimal | Yes | Period gross pay |
| earnings_basis | enum | Yes | "qualifying" / "banded" / "total" |
| employee_rate | decimal(5,2) | Yes | Employee contribution rate |
| employer_rate | decimal(5,2) | Yes | Employer contribution rate |
| rate_type | enum | Yes | "percentage" / "fixed" |
| thresholds | PeriodThresholds | Yes | Lower/upper earnings thresholds |

### ContributionResult
| Field | Type | Description |
|-------|------|-------------|
| employee_contribution | Decimal | Employee deduction amount |
| employer_contribution | Decimal | Employer contribution amount |
| total_contribution | Decimal | Sum of both |
| qualifying_earnings | Decimal | Earnings used for calculation |
| earnings_basis | enum | How earnings were calculated |
| calculation_trace | TraceStep[] | Step-by-step calculation |

---

## Business Rules & Invariants

1. Qualifying earnings = max(0, min(gross, upper_threshold) - lower_threshold)
2. Percentage contributions: (qualifying_earnings × rate) / 100
3. Fixed contributions: Use specified amount (if earnings > 0)
4. Minimum contributions met (3% employer, 5% total for 2026-27)
5. Rounding: Half-even to nearest penny at final step
6. Zero earnings = zero contributions (valid result)
7. Contribution record created for every payslip with pension

---

## Edge Cases

1. **Earnings below lower threshold** — Qualifying earnings = 0, contributions = 0
2. **Earnings above upper threshold** — Capped at upper threshold
3. **Rate exactly 0%** — Zero contribution recorded
4. **Very small earnings** — Minimum 1p contribution if rate applies
5. **Mid-period rate change** — Use rate effective on pay date
6. **Certified scheme** — Use certified calculation rules

---

## Tests

### contributionCalculator.test.ts
- Standard qualifying earnings calculation
- Earnings below lower threshold
- Earnings above upper threshold
- Percentage rate calculation (5%, 3%)
- Fixed amount contribution
- Rounding to nearest penny
- Zero earnings handling
- HMRC reference calculation verification

---

## Verification

```bash
cd "/Users/josephstephenson-mouzo/Projects/03 - development/16 - payroll mvp"
npm run test:unit src/tests/pension/contributionCalculator.test.ts
npm run typecheck
npm run lint
npm run build
```

---

## Source Sections

- 02-03-pension-auto-enrolment-spec.md § Entity: PensionContribution → Data model
- story-02-assessment-engine/slice-c.md → Earnings calculation utilities
