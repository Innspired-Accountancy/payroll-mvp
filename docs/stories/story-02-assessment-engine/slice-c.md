# Slice c: Age and Earnings Calculation Utilities

**Story:** story-02-assessment-engine
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Create utility functions for accurate age calculation and earnings aggregation to support the assessment engine.

---

## Decision Checklist

- [x] Age calculation: date-fns differenceInYears with boundary handling
- [x] State Pension Age: Lookup table per HMRC (born 6 Apr 1960-5 Apr 1977 = 67)
- [x] Earnings aggregation: Sum all employments under same employer
- [x] Period pro-rating: Annual threshold / pay frequency
- [x] Decimal precision: Decimal.js for all money calculations
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Data Models, Qualifying earnings
- HMRC: State Pension age timetable

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/ageCalculator.ts` | create | Age and SPA calculation utilities |
| `src/lib/pension/earningsCalculator.ts` | create | Earnings aggregation and pro-rating |
| `src/lib/config/statePensionAge.ts` | create | SPA lookup tables by DOB |
| `src/lib/types/pensionCalculations.ts` | create | Calculation type definitions |
| `src/tests/pension/ageCalculator.test.ts` | create | Age calculation tests |
| `src/tests/pension/earningsCalculator.test.ts` | create | Earnings calculation tests |

---

## Responsibilities

1. Calculate precise age in years given DOB and assessment date
2. Determine State Pension Age based on date of birth
3. Aggregate earnings across multiple employments for same employee/employer
4. Pro-rate annual thresholds for weekly/monthly pay periods
5. Calculate qualifying earnings within band

---

## Contracts

### calculateAge()
- **Method:** `calculateAge(dob: Date, asOfDate: Date): number`
- **Returns:** Age in complete years
- **Logic:** differenceInYears with birthday boundary handling

### getStatePensionAge()
- **Method:** `getStatePensionAge(dob: Date): number`
- **Returns:** State Pension Age (66, 67, or 68)
- **Lookup:** HMRC SPA timetable by date of birth ranges

### calculateQualifyingEarnings()
- **Method:** `calculateQualifyingEarnings(grossEarnings: Decimal, thresholds: PeriodThresholds): Decimal`
- **Returns:** Earnings between lower and upper thresholds
- **Logic:** max(0, min(gross, upper) - lower)

### aggregateEmployments()
- **Method:** `aggregateEmployments(employeeId: UUID, employerId: UUID, periodId: UUID): Decimal`
- **Returns:** Sum of gross earnings across all employments
- **Query:** Sum earnings where employee_id = X AND employer_id = Y

### proRateThresholds()
- **Method:** `proRateThresholds(annualThresholds: AnnualThresholds, payFrequency: enum): PeriodThresholds`
- **Returns:** Pro-rated lower and upper thresholds
- **Calculation:** Monthly = annual/12, Weekly = annual/52, 4-weekly = annual/13

---

## Business Rules & Invariants

1. Age calculated as complete years (birthday inclusive)
2. SPA lookup uses date of birth ranges per HMRC
3. Qualifying earnings never negative
4. Multiple employments aggregated before threshold application
5. Pro-rated thresholds rounded to 2 decimal places (pence)
6. Annual thresholds configurable per tax year

---

## Edge Cases

1. **Birthday on assessment date** — Age increments on that date
2. **SPA boundary DOB (5 Apr 1960)** — SPA = 66, birthday after 6 Apr = 67
3. **Zero earnings** — Qualifying earnings = 0
4. **Earnings below lower threshold** — Qualifying earnings = 0
5. **Earnings above upper threshold** — Qualifying earnings capped at (upper - lower)
6. **Irregular pay frequency** — Use nearest standard frequency pro-rating

---

## Tests

### ageCalculator.test.ts
- Age calculation at birthday boundary
- Age calculation day before birthday
- SPA lookup for various DOBs
- SPA boundary case (6 Apr 1960)

### earningsCalculator.test.ts
- Qualifying earnings within band
- Earnings below lower threshold
- Earnings above upper threshold
- Multiple employments sum correctly
- Monthly pro-rating accuracy
- Weekly pro-rating accuracy

---

## Verification

```bash
npm run test:unit src/tests/pension/ageCalculator.test.ts
npm run test:unit src/tests/pension/earningsCalculator.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- 02-03-pension-auto-enrolment-spec.md § Entity: PensionAssessment → Fields
