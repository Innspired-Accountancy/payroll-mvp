# Slice b: Assessment Trigger Integration

**Story:** story-02-assessment-engine
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Integrate the assessment engine with payroll processing to trigger assessments automatically at the right points in the employee lifecycle.

---

## Decision Checklist

- [x] Trigger points: New employee added, pay period end, postponement period end
- [x] Integration method: tRPC procedure calls from payroll engine
- [x] Batch processing: Assess all employees in pay period with single call
- [x] Async handling: Queue-based for large employers (BullMQ)
- [x] Error scenarios: Failed assessments logged, retry mechanism
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Journey 1, Integration Points
- story-02-assessment-engine/slice-a.md — Core assessment contracts

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/assessmentTriggers.ts` | create | Trigger orchestration logic |
| `src/server/routers/pensionAssessments.ts` | update | Add batch assessment procedure |
| `src/lib/jobs/pensionAssessmentQueue.ts` | create | BullMQ queue for async processing |
| `src/lib/pension/postponementManager.ts` | create | Postponement period tracking |
| `src/tests/pension/assessmentTriggers.test.ts` | create | Integration tests |

---

## Responsibilities

1. Trigger assessment on new employee creation
2. Trigger assessment for all employees at pay period end
3. Track and trigger assessments when postponement periods end
4. Batch process assessments efficiently
5. Queue large employer assessments for background processing
6. Log all triggers and outcomes

---

## Contracts

### pensionAssessments.assessPayPeriod
- **Method:** tRPC mutation `pensionAssessments.assessPayPeriod`
- **Input:** `{ payPeriodId: UUID, employerId: UUID }`
- **Output:** `{ assessmentCount: number, eligibleCount: number, errors: AssessmentError[] }`
- **Errors:** NOT_FOUND (pay period), FORBIDDEN (wrong employer)

### pensionAssessments.assessEmployee
- **Method:** tRPC mutation `pensionAssessments.assessEmployee`
- **Input:** `{ employeeId: UUID, assessmentReason: enum }`
- **Output:** AssessmentResult
- **Errors:** NOT_FOUND (employee), ASSESSMENT_FAILED (calculation error)

### PostponementManager
- **Method:** `schedulePostponementEndAssessment(employeeId, endDate)`
- **Queue:** BullMQ delayed job for postponement end date
- **Action:** Trigger assessment when delay expires

---

## Business Rules & Invariants

1. New employees assessed within first pay period
2. All existing employees assessed every pay period
3. Postponement can delay assessment by up to 3 months
4. Postponement end triggers immediate assessment
5. Failed assessments retried 3 times before manual intervention
6. Assessment results feed into enrolment workflow

---

## Edge Cases

1. **Employee added mid-period** — Assess in current period
2. **Pay period with no employees** — Return zero counts, no error
3. **Assessment fails for one employee** — Log error, continue with others
4. **Postponement end on weekend** — Assess on next business day
5. **Multiple postponements** — Track each separately, assess at each end date

---

## Tests

### assessmentTriggers.test.ts
- New employee triggers assessment
- Pay period end triggers batch assessment
- Postponement end triggers delayed assessment
- Failed assessment retry logic
- Batch processing handles errors gracefully

---

## Verification

```bash
npm run test:unit src/tests/pension/assessmentTriggers.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-02-assessment-engine/slice-a.md → Core assessment logic
