# Slice a: Assessment Engine Core Logic

**Story:** story-02-assessment-engine
**Epic:** epic-03-pension-ae
**Effort:** M
**Dependencies:** story-01-pension-schemes

---

## Goal

Implement the core worker categorization algorithm that determines auto-enrolment eligibility based on age and qualifying earnings thresholds.

---

## Decision Checklist

- [x] Algorithm: The Pensions Regulator categorization rules (TPR guidance)
- [x] Libraries: date-fns 3.6.x for age calculations, Decimal.js 10.4.x for money
- [x] Data contracts: WorkerCategory enum, PensionAssessmentInput/Output Zod schemas
- [x] Thresholds: Config-driven per tax year (2026-27: £6,240 lower, £50,270 upper)
- [x] State Pension Age: HMRC lookup table (born before 6 Apr 1970 = 67, after = increasing to 68)
- [x] Error scenarios: Invalid input, missing employee data, calculation overflow
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Entity: PensionAssessment, Journey 1
- The Pensions Regulator: "Automatic enrolment detailed guidance for employers"

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/assessmentEngine.ts` | create | Core assessment algorithm |
| `src/lib/pension/workerCategories.ts` | create | Worker category definitions and rules |
| `src/lib/db/schema/pensionAssessments.ts` | create | Assessment table schema |
| `src/lib/config/pensionThresholds.ts` | create | Tax-year specific thresholds |
| `src/lib/types/pensionAssessment.ts` | create | TypeScript types |
| `src/server/routers/pensionAssessments.ts` | create | tRPC procedures |
| `src/tests/pension/assessmentEngine.test.ts` | create | Unit tests for categorization |

---

## Responsibilities

1. Calculate employee age on assessment date
2. Determine if age is between 22 and State Pension Age
3. Calculate qualifying earnings for the period
4. Apply earnings thresholds (£6,240 and £50,270 annually, pro-rated for period)
5. Categorize worker into Eligible/Non-Eligible/Entitled
6. Determine enrolment obligations (must_enrol, can_opt_in, can_join)
7. Record immutable assessment record

---

## Contracts

### assessWorker()
- **Method:** `assessWorker(input: AssessmentInput): AssessmentResult`
- **Input:** AssessmentInput Zod schema
- **Output:** AssessmentResult with category and obligations
- **Errors:** ZodError (validation), AssessmentError (calculation failure)

### AssessmentInput Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| employee_id | UUID | Yes | Employee to assess |
| pay_period_id | UUID | Yes | Period being assessed |
| assessment_date | Date | Yes | Date of assessment |
| date_of_birth | Date | Yes | Employee DOB |
| gross_earnings | Decimal | Yes | Period gross earnings |
| tax_year | string | Yes | "2026-27" format |
| assessment_reason | enum | Yes | "new_employee" / "periodic" / "postponement_end" |

### AssessmentResult
| Field | Type | Description |
|-------|------|-------------|
| worker_category | enum | "eligible_jobholder" / "non_eligible_jobholder" / "entitled_worker" |
| must_enrol | boolean | Auto-enrolment required |
| can_opt_in | boolean | Employee can request opt-in |
| can_join | boolean | Employee can request to join |
| age_at_assessment | number | Calculated age in years |
| qualifying_earnings | Decimal | Earnings within qualifying band |
| assessment_basis | string | Pro-rated or annual calculation |

---

## Business Rules & Invariants

1. Eligible Jobholder: Age 22 to SPA AND qualifying earnings > £6,240 (2026-27)
2. Non-Eligible Jobholder: Age 16-74 AND earnings > £6,240 but either age outside 22-SPA OR earnings < £6,240
3. Entitled Worker: Age 16-74 AND earnings £6,240-£50,270
4. State Pension Age determined by date of birth per HMRC tables
5. Qualifying earnings = gross earnings between £6,240 and £50,270 (annual)
6. Period thresholds pro-rated: monthly = annual/12, weekly = annual/52
7. Assessment records are immutable (no updates, only new records)
8. Every worker assessed every pay reference period

---

## Edge Cases

1. **Age exactly 22 on assessment date** — Eligible if earnings meet threshold
2. **Age exactly SPA on assessment date** — Non-eligible (SPA is upper bound exclusive)
3. **Earnings exactly at threshold** — Include in category (>= lower, <= upper)
4. **Birthday during pay period** — Age calculated as of assessment date
5. **Zero earnings** — Entitled worker if age 16-74 (can request to join)
6. **Multiple employments** — Aggregate all earnings under same employer before assessment
7. **Leaver mid-period** — Assess based on actual earnings in period

---

## Tests

### assessmentEngine.test.ts
- Eligible jobholder: age 25, earnings £2,500/month
- Non-eligible jobholder (young): age 20, earnings £2,500/month
- Non-eligible jobholder (old): age 68 (SPA), earnings £2,500/month
- Entitled worker: age 30, earnings £400/month (below threshold)
- Boundary: Exactly 22 years old
- Boundary: Exactly at lower earnings threshold
- Multiple employments aggregated correctly
- Different tax years use correct thresholds

---

## Verification

```bash
cd "/Users/josephstephenson-mouzo/Projects/03 - development/16 - payroll mvp"
npm run test:unit src/tests/pension/assessmentEngine.test.ts
npm run typecheck
npm run lint
npm run build
```

---

## Source Sections

- 02-03-pension-auto-enrolment-spec.md § Entity: PensionAssessment → Data model
- 02-03-pension-auto-enrolment-spec.md § Journey 1 → Worker Assessment flow
