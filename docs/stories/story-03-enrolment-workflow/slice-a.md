# Slice a: Enrolment Workflow Engine

**Story:** story-03-enrolment-workflow
**Epic:** epic-03-pension-ae
**Effort:** M
**Dependencies:** story-02-assessment-engine

---

## Goal

Implement the core enrolment workflow that processes eligible jobholders and creates pension enrolment records with proper opt-out windows.

---

## Decision Checklist

- [x] Enrolment trigger: Assessment result with must_enrol = true
- [x] Opt-out window: 1 calendar month from enrolment date (date-fns addMonths)
- [x] Deduplication: Check existing active enrolment before creating new
- [x] Re-enrolment check: 12-month exclusion after opt-out
- [x] Postponement: 3-month delay with tracked end date
- [x] Enrolment reason: "automatic" for auto-enrol, "opt_in" for requests
- [x] Error scenarios: Duplicate enrolment, scheme not found, communication failure
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Entity: PensionEnrolment, Journey 2
- The Pensions Regulator: "What to do when you've automatically enrolled"

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/enrolmentEngine.ts` | create | Core enrolment workflow |
| `src/lib/db/schema/pensionEnrolments.ts` | create | Enrolment table schema |
| `src/lib/pension/enrolmentRules.ts` | create | Eligibility and exclusion rules |
| `src/server/routers/pensionEnrolments.ts` | create | tRPC procedures |
| `src/lib/types/pensionEnrolment.ts` | create | TypeScript types |
| `src/tests/pension/enrolmentEngine.test.ts` | create | Unit tests |

---

## Responsibilities

1. Process assessment results with must_enrol = true
2. Check for existing active enrolment (deduplication)
3. Check 12-month opt-out exclusion period
4. Determine enrolment reason (automatic vs opt-in)
5. Select appropriate scheme (default or specified)
6. Calculate opt-out deadline (1 month from enrolment date)
7. Create immutable PensionEnrolment record
8. Trigger communication generation

---

## Contracts

### enrolEligibleWorker()
- **Method:** `enrolEligibleWorker(input: EnrolmentInput): EnrolmentResult`
- **Input:** EnrolmentInput Zod schema
- **Output:** EnrolmentResult with enrolment record
- **Errors:** 
  - ALREADY_ENROLLED (existing active enrolment)
  - RECENT_OPT_OUT (within 12-month exclusion)
  - SCHEME_NOT_FOUND (no default scheme configured)
  - ENROLMENT_FAILED (database error)

### EnrolmentInput Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| employee_id | UUID | Yes | Employee to enrol |
| assessment_id | UUID | Yes | Triggering assessment |
| scheme_id | UUID | No | Specific scheme (uses default if null) |
| enrolment_date | Date | Yes | Date of enrolment |
| enrolment_reason | enum | Yes | "automatic" / "opt_in" / "join_request" |

### EnrolmentResult
| Field | Type | Description |
|-------|------|-------------|
| enrolment_id | UUID | Created enrolment record ID |
| scheme_name | string | Enrolled scheme name |
| opt_out_deadline | Date | One month from enrolment |
| status | enum | "enrolled" |
| communication_queued | boolean | Whether communication was triggered |

### pensionEnrolments.enrolEmployee
- **Method:** tRPC mutation `pensionEnrolments.enrolEmployee`
- **Input:** EnrolmentInput
- **Output:** EnrolmentResult
- **Auth:** Requires pension:enrolment:create permission

---

## Business Rules & Invariants

1. Only one active enrolment per employee per scheme
2. Opt-out deadline = enrolment_date + 1 month
3. 12-month exclusion after opt-out prevents re-auto-enrolment
4. Automatic enrolment uses employer's default scheme
5. Enrolment record immutable after creation
6. Communication must be triggered within 6 weeks (42 days)
7. Employee included in contributions from enrolment date

---

## Edge Cases

1. **Already enrolled** — Return existing enrolment, no error
2. **Opted out 11 months ago** — Exclude from auto-enrol (within 12 months)
3. **Opted out 13 months ago** — Allow auto-enrol
4. **No default scheme** — Error, requires manual scheme selection
5. **Enrolment on last day of month** — Deadline is last day of next month
6. **Leap year February** — Handle 29 Feb correctly

---

## Tests

### enrolmentEngine.test.ts
- Enrol eligible jobholder with default scheme
- Skip enrolment for already enrolled employee
- Skip enrolment for recent opt-out (11 months)
- Allow enrolment for old opt-out (13 months)
- Error when no default scheme configured
- Correct opt-out deadline calculation
- Enrolment with specified scheme
- Handle leap year enrolment dates

---

## Verification

```bash
cd "/Users/josephstephenson-mouzo/Projects/03 - development/16 - payroll mvp"
npm run test:unit src/tests/pension/enrolmentEngine.test.ts
npm run typecheck
npm run lint
npm run build
```

---

## Source Sections

- 02-03-pension-auto-enrolment-spec.md § Entity: PensionEnrolment → Data model
- 02-03-pension-auto-enrolment-spec.md § Journey 2 → Enrolment workflow
