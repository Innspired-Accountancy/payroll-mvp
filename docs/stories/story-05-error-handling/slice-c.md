# Slice c: Correction Workflow UI and API

**Story:** story-05-error-handling
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Build correction workflow UI and API that guides payroll managers through fixing HMRC rejection errors, tracking corrections, and triggering resubmission.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, React 18.x, Radix UI
- [x] SDK methods identified: tRPC procedures for correction workflow
- [x] External service endpoints: N/A (internal workflow)
- [x] Data contracts defined: CorrectionWorkflow, ErrorCorrection interfaces
- [x] Configuration: N/A
- [x] Error scenarios: Correction invalid, employee data locked, resubmission fails
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Manager corrects data and resubmits
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Maintain submission chain

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/hmrc-corrections.ts` | create | tRPC router for corrections |
| `src/app/(bureau)/fps/[id]/errors/page.tsx` | create | Error display and correction page |
| `src/components/fps/ErrorList.tsx` | create | Error list with correction actions |
| `src/components/fps/CorrectionWizard.tsx` | create | Step-by-step correction wizard |

---

## Responsibilities

1. Display rejection errors grouped by employee
2. Link errors to editable payroll data fields
3. Track correction status for each error
4. Validate corrections before allowing resubmission
5. Trigger FPS regeneration and resubmission

---

## Contracts

### hmrc.getSubmissionErrors
- **Method:** tRPC query `hmrc.getSubmissionErrors`
- **Input:** `{ submissionId: string }`
- **Output:**
  ```typescript
  const submissionErrorsOutput = z.array(z.object({
    id: z.string().uuid(),
    errorCode: z.string(),
    message: z.string(),
    severity: z.enum(['fatal', 'error', 'warning']),
    employee: z.object({
      id: z.string().uuid(),
      nino: z.string(),
      name: z.string()
    }).optional(),
    field: z.string(),
    currentValue: z.string().optional(),
    guidance: z.string(),
    resolved: z.boolean(),
    resolvedAt: z.string().datetime().optional(),
    resolutionNote: z.string().optional()
  }));
  ```
- **Auth:** Protected with `hmrc:errors:read` permission

### hmrc.markErrorResolved
- **Method:** tRPC mutation `hmrc.markErrorResolved`
- **Input:**
  ```typescript
  const markErrorResolvedInput = z.object({
    errorId: z.string().uuid(),
    resolutionNote: z.string().optional()
  });
  ```
- **Output:** `{ success: boolean }`
- **Auth:** Protected with `hmrc:errors:resolve` permission

### hmrc.initiateCorrection
- **Method:** tRPC mutation `hmrc.initiateCorrection`
- **Input:**
  ```typescript
  const initiateCorrectionInput = z.object({
    submissionId: z.string().uuid(),
    correctionData: z.array(z.object({
      employeeId: z.string().uuid().optional(),
      field: z.string(),
      newValue: z.unknown()
    }))
  });
  ```
- **Output:**
  ```typescript
  z.object({
    correctionSubmissionId: z.string().uuid(),
    correlationId: z.string(),
    parentSubmissionId: z.string().uuid()
  });
  ```
- **Behavior:**
  1. Update employee/payroll data with corrections
  2. Create new submission record (correction version)
  3. Link to parent submission
  4. Regenerate FPS XML
  5. Return new submission for validation/submission
- **Auth:** Protected with `hmrc:fps:correct` permission

---

## Business Rules & Invariants

1. All fatal errors must be resolved before resubmission
2. Warnings can be acknowledged without correction
3. Corrections are applied to payroll data (not just submission)
4. Correction submissions are versioned (v2, v3, etc.)
5. Parent-child submission chain preserved for audit
6. Late reporting reason auto-set for corrections after 19th

---

## Edge Cases

1. **Employee left company** — Flag for manual handling, suggest EPS correction
2. **Data already changed by another user** — Detect conflict, show diff
3. **Correction creates new validation errors** — Show in validation step
4. **User abandons correction** — Keep original rejected, mark as abandoned
5. **Partial correction (some errors fixed)** — Allow resubmission of partial fixes

---

## Tests

### corrections.router.test.ts
- Get errors for rejected submission
- Mark error as resolved
- Initiate correction workflow
- Verify parent-child submission linkage
- Verify late reason code set for late correction

---

## Verification

```bash
npm run typecheck
npm run test src/server/routers/hmrc-corrections.router.test.ts
npm run lint src/app/(bureau)/fps/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Manager corrects data and resubmits
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Link HMRC errors to employee records
