# Slice c: Contribution Schedule Submission

**Story:** story-05-nest-integration
**Epic:** epic-03-pension-ae
**Effort:** L
**Dependencies:** slice-a

---

## Goal

Implement NEST contribution schedule submission including employee contributions, employer contributions, and payment instructions.

---

## Decision Checklist

- [x] API endpoint: POST /contribution-schedules
- [x] Schedule format: NEST Contribution Schedule schema
- [x] Payment timing: Direct Debit or bank transfer options
- [x] Validation: Pre-submission checks for data integrity
- [x] Reconciliation: Match submitted vs accepted records
- [x] Status polling: Check schedule processing status
- [x] Error handling: Individual record errors vs schedule errors
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- NEST Web Services Developer Guide — Contribution Schedule API
- 02-03-pension-auto-enrolment-spec.md — Journey 4

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/nest/contributionSubmission.ts` | create | Contribution schedule logic |
| `src/lib/nest/mappers/contributionMapper.ts` | create | Map contributions to NEST format |
| `src/lib/nest/reconciliation.ts` | create | Submission reconciliation |
| `src/lib/jobs/nestSubmissionPoller.ts` | create | Status polling job |
| `src/server/routers/nestContributions.ts` | create | tRPC contribution procedures |
| `src/tests/nest/contributionSubmission.test.ts` | create | Submission tests |

---

## Responsibilities

1. Aggregate contributions for pay period
2. Build NEST contribution schedule
3. Validate all contribution records
4. Submit schedule to NEST API
5. Poll for processing status
6. Reconcile submitted vs accepted
7. Handle individual record errors
8. Update payment status

---

## Contracts

### submitContributionSchedule()
- **Method:** `submitContributionSchedule(payPeriodId: UUID): Promise<ScheduleResult>`
- **Input:** Pay period with contributions
- **Output:** ScheduleResult with submission status
- **Process:**
  1. Fetch all contributions for period
  2. Group by scheme/employer
  3. Build NEST ContributionSchedule
  4. Validate records
  5. POST to /contribution-schedules
  6. Store submission reference
  7. Schedule status polling

### NEST ContributionSchedule
| Field | Type | Description |
|-------|------|-------------|
| organisationId | string | NEST organisation ID |
| employerReference | string | Employer reference |
| payPeriodStart | string | YYYY-MM-DD |
| payPeriodEnd | string | YYYY-MM-DD |
| paymentMethod | string | "direct_debit" / "bank_transfer" |
| paymentDueDate | string | YYYY-MM-DD |
| totalAmount | decimal | Total contributions |
| records | ContributionRecord[] | Individual records |

### ContributionRecord
| Field | Type | Description |
|-------|------|-------------|
| employeeReference | string | Employer employee ref |
| niNumber | string | NI number |
| pensionablePay | decimal | Earnings used |
| employeeContribution | decimal | Employee amount |
| employerContribution | decimal | Employer amount |
| totalContribution | decimal | Sum of both |

### ScheduleResult
| Field | Type | Description |
|-------|------|-------------|
| submissionId | UUID | Internal submission ID |
| providerReference | string | NEST schedule reference |
| status | enum | "submitted" / "accepted" / "partial" / "rejected" |
| totalRecords | number | Count of records |
| acceptedRecords | number | Count accepted |
| rejectedRecords | number | Count rejected |
| errors | RecordError[] | Individual errors |

---

## Business Rules & Invariants

1. Contribution schedules submitted after pay period ends
2. Payment due date typically 22nd of following month
3. All enrolled employees included even if zero contribution
4. Employee reference must match existing NEST enrolment
5. Total amount must equal sum of all record amounts
6. Rejected records must be corrected and resubmitted
7. Partial acceptance requires follow-up for rejected records

---

## Edge Cases

1. **Zero contribution employee** — Included with £0.00 amounts
2. **Leaver mid-period** — Pro-rated or final contribution
3. **Payment date on weekend** — Adjust to next business day
4. **Individual record rejected** — Other records still processed
5. **Schedule total mismatch** — Validation error before submit

---

## Tests

### contributionSubmission.test.ts
- Full schedule submission
- Partial acceptance handling
- Individual record error handling
- Reconciliation after status polling
- Payment due date calculation
- Zero contribution handling

---

## Verification

```bash
npm run test:unit src/tests/nest/contributionSubmission.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-05-nest-integration/slice-a.md → API client
- story-04-contributions → Contribution data source
