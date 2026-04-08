# Slice c: Validation Results API and UI

**Story:** story-02-fps-validation
**Epic:** epic-02-hmrc-submissions
**Effort:** S
**Dependencies:** slice-b

---

## Goal

Expose validation functionality through tRPC procedures and display validation results in a user-friendly format with clear error messages, field-level links, and correction guidance.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, React 18.x, Tailwind CSS
- [x] SDK methods identified: tRPC query/mutation procedures
- [x] External service endpoints: N/A (internal)
- [x] Data contracts defined: ValidateFPSInput, ValidationResultOutput
- [x] Configuration: N/A
- [x] Error scenarios: Submission not found, validation service errors
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: Manager reviews validation results
- 02-02-hmrc-submissions-spec.md § API Contracts → Validation output shapes

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/hmrc-submissions.ts` | update | Add validation procedures |
| `src/app/(bureau)/fps/[id]/validate/page.tsx` | create | Validation results page |
| `src/components/fps/ValidationResults.tsx` | create | Validation results component |
| `src/components/fps/ValidationIssueCard.tsx` | create | Individual issue display |

---

## Responsibilities

1. Add tRPC procedures for validating FPS submissions
2. Store validation results in database
3. Display validation results grouped by severity
4. Link issues to specific employees and fields
5. Show correction guidance for each issue

---

## Contracts

### hmrc.validateFPS
- **Method:** tRPC mutation `hmrc.validateFPS`
- **Input:**
  ```typescript
  const validateFPSInput = z.object({
    submissionId: z.string().uuid()
  });
  ```
- **Output:** ValidationResultOutput
  ```typescript
  const validationResultOutput = z.object({
    submissionId: z.string().uuid(),
    status: z.enum(['draft', 'validated', 'validation_failed']),
    xsdValid: z.boolean(),
    xsdErrors: z.array(z.object({
      line: z.number(),
      message: z.string(),
      xpath: z.string()
    })),
    businessRulesValid: z.boolean(),
    blockers: z.array(z.object({
      code: z.string(),
      message: z.string(),
      employeeId: z.string().uuid().optional(),
      employeeName: z.string().optional(),
      field: z.string(),
      guidance: z.string()
    })),
    warnings: z.array(z.object({
      code: z.string(),
      message: z.string(),
      employeeId: z.string().uuid().optional(),
      employeeName: z.string().optional(),
      field: z.string(),
      guidance: z.string()
    })),
    validatedAt: z.string().datetime(),
    canSubmit: z.boolean()
  });
  ```
- **Errors:**
  - `NOT_FOUND` — Submission not found
  - `BAD_REQUEST` — Submission not in draft status
  - `FORBIDDEN` — User lacks permission
- **Auth:** Protected procedure with `hmrc:fps:validate` permission

### hmrc.getValidationResults
- **Method:** tRPC query `hmrc.getValidationResults`
- **Input:** `{ submissionId: string }`
- **Output:** Latest ValidationResultOutput for submission
- **Auth:** Protected procedure

---

## Business Rules & Invariants

1. Validation can only be run on submissions in 'draft' status
2. XSD validation runs before business rules
3. Business rules only run if XSD validation passes
4. Submission moves to 'validated' status only if no blockers
5. Validation results are persisted for audit trail
6. CanSubmit is true only if XSD valid AND no blockers

---

## Edge Cases

1. **Validation takes >30 seconds** — Show progress, background processing
2. **Validation service unavailable** — Queue for retry, notify user
3. **Large number of issues (100+)** — Paginate, prioritize blockers
4. **Issue field no longer exists** — Graceful handling, show XML line

---

## Tests

### validation.router.test.ts
- Validate FPS with no issues
- Validate FPS with XSD errors
- Validate FPS with business rule blockers
- Validate FPS with warnings only
- Reject validation for non-existent submission
- Reject validation for already-submitted FPS

---

## Verification

```bash
npm run typecheck
npm run test src/server/routers/hmrc-submissions.validation.test.ts
npm run lint src/app/(bureau)/fps/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: Manager reviews validation results
- 02-02-hmrc-submissions-spec.md § API Contracts → Validation output JSON
