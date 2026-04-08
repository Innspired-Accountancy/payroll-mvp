# Slice b: Enrolment Submission to NEST

**Story:** story-05-nest-integration
**Epic:** epic-03-pension-ae
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement NEST enrolment submission API to register newly enrolled employees with NEST.

---

## Decision Checklist

- [x] API endpoint: POST /enrolments
- [x] Payload format: NEST Enrolment Request schema
- [x] Required fields: personal details, NI number, employment info
- [x] Opt-out preference: Include opt-out channel preference
- [x] Response handling: Parse enrolment reference
- [x] Status tracking: Record submission status and provider reference
- [x] Error handling: Map NEST errors to user-friendly messages
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- NEST Web Services Developer Guide — Enrolment API
- 02-03-pension-auto-enrolment-spec.md — Entity: PensionProviderSubmission

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/nest/enrolmentSubmission.ts` | create | Enrolment submit logic |
| `src/lib/nest/mappers/enrolmentMapper.ts` | create | Map internal to NEST format |
| `src/lib/nest/validators/enrolmentValidator.ts` | create | Pre-submission validation |
| `src/server/routers/nestSubmissions.ts` | create | tRPC submission procedures |
| `src/tests/nest/enrolmentSubmission.test.ts` | create | Submission tests |

---

## Responsibilities

1. Map internal enrolment data to NEST format
2. Validate data before submission
3. Submit enrolment to NEST API
4. Handle response (success/failure)
5. Store provider reference on success
6. Log submission for audit trail
7. Queue for retry on transient failure

---

## Contracts

### submitEnrolment()
- **Method:** `submitEnrolment(enrolmentId: UUID): Promise<SubmissionResult>`
- **Input:** Internal enrolment record
- **Output:** SubmissionResult with status and reference
- **Process:**
  1. Fetch enrolment and employee data
  2. Validate required fields
  3. Map to NEST EnrolmentRequest format
  4. POST to /enrolments
  5. Handle response
  6. Update submission record

### NEST EnrolmentRequest
| Field | Type | Description |
|-------|------|-------------|
| organisationId | string | NEST organisation ID |
| employeeReference | string | Employer employee ref |
| niNumber | string | National Insurance number |
| title | string | Mr/Mrs/Ms/etc |
| firstName | string | First name |
| lastName | string | Last name |
| dateOfBirth | string | YYYY-MM-DD |
| gender | string | M/F/U |
| address | object | { line1, line2, line3, town, county, postcode } |
| email | string | Email address |
| phone | string | Phone number |
| employmentStartDate | string | YYYY-MM-DD |
| optOutPreference | string | "online" / "paper" |

### SubmissionResult
| Field | Type | Description |
|-------|------|-------------|
| status | enum | "submitted" / "accepted" / "rejected" |
| providerReference | string | NEST enrolment reference |
| errors | ErrorDetail[] | Validation errors if rejected |
| submittedAt | DateTime | Submission timestamp |

---

## Business Rules & Invariants

1. Enrolment must be submitted within 6 weeks of enrolment date
2. NI number required for NEST enrolment
3. Employee reference must be unique within organisation
4. Opt-out preference defaults to "online"
5. Successful submission stores NEST reference for future updates
6. Rejected submissions logged with error details for correction

---

## Edge Cases

1. **Employee already exists in NEST** — Update instead of create
2. **Invalid NI number** — Validation error before submission
3. **NEST API unavailable** — Queue for retry
4. **Duplicate submission** — Check existing reference first
5. **Missing required field** — Pre-submission validation failure

---

## Tests

### enrolmentSubmission.test.ts
- Successful enrolment submission
- NI number validation failure
- Duplicate employee handling
- NEST API error handling
- Retry on transient failure
- Response parsing and storage

---

## Verification

```bash
npm run test:unit src/tests/nest/enrolmentSubmission.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-05-nest-integration/slice-a.md → API client to use
- story-03-enrolment-workflow → Enrolment data source
