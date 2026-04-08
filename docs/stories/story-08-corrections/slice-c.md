# Slice c: Correction Workflow UI and History

**Story:** story-08-corrections
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Build UI for correction workflow showing submission history, correction eligibility, and step-by-step correction process. Display full submission chain with version history.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, React 18.x, Radix UI
- [x] SDK methods identified: tRPC queries/mutations for corrections
- [x] External service endpoints: N/A (internal)
- [x] Data contracts defined: SubmissionHistory, CorrectionWorkflow interfaces
- [x] Configuration: N/A
- [x] Error scenarios: Chain not found, ineligible correction, version conflict
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Manager corrects data
- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmission.submission_version

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/hmrc-corrections.ts` | update | Add correction workflow endpoints |
| `src/app/(bureau)/fps/[id]/history/page.tsx` | create | Submission history page |
| `src/components/fps/SubmissionChain.tsx` | create | Chain visualization component |
| `src/components/fps/CorrectionWizard.tsx` | update | Correction workflow UI |

---

## Responsibilities

1. Display submission chain (original → corrections)
2. Show correction eligibility status
3. Guide user through correction process
4. Show differences between versions
5. Track correction progress

---

## Contracts

### hmrc.getSubmissionChain
- **Method:** tRPC query `hmrc.getSubmissionChain`
- **Input:** `{ submissionId: string }`
- **Output:**
  ```typescript
  const submissionChainOutput = z.object({
    rootSubmission: z.object({
      id: z.string().uuid(),
      correlationId: z.string(),
      taxYear: z.string(),
      taxPeriod: z.number(),
      status: z.string(),
      submittedAt: z.string().datetime(),
      version: z.number()
    }),
    chain: z.array(z.object({
      id: z.string().uuid(),
      correlationId: z.string(),
      status: z.string(),
      submittedAt: z.string().datetime().optional(),
      version: z.number(),
      isCorrection: z.boolean(),
      parentSubmissionId: z.string().uuid().optional(),
      lateReason: z.string().optional(),
      correctionsSummary: z.array(z.object({
        employeeName: z.string(),
        field: z.string(),
        oldValue: z.string(),
        newValue: z.string()
      })).optional()
    })),
    currentVersion: z.number(),
    canCorrect: z.boolean(),
    correctionEligibility: z.object({
      eligible: z.boolean(),
      reason: z.string().optional(),
      correctionType: z.enum(['current_year', 'prior_year']).optional(),
      earliestDate: z.string().datetime().optional()
    })
  });
  ```

### hmrc.initiateCorrection
- **Method:** tRPC mutation `hmrc.initiateCorrection`
- **Input:**
  ```typescript
  const initiateCorrectionInput = z.object({
    parentSubmissionId: z.string().uuid(),
    corrections: z.array(z.object({
      employeeId: z.string().uuid(),
      fieldPath: z.string(),
      newValue: z.string()
    })),
    lateReason: z.enum(['H', 'I', 'J', 'K', 'L', 'M']).optional(),
    correctionNote: z.string().optional()
  });
  ```
- **Output:**
  ```typescript
  z.object({
    correctionSubmissionId: z.string().uuid(),
    correlationId: z.string(),
    version: z.number(),
    status: z.enum(['draft', 'validated']),
    lateReason: z.string(),
    correctionsApplied: z.number(),
    previewUrl: z.string().url() // Link to preview correction
  });
  ```

### Version Comparison API
```typescript
hmrc.compareSubmissions({
  submissionIdA: string;
  submissionIdB: string;
}): Promise<{
  differences: Array<{
    type: 'employee' | 'total' | 'employer';
    field: string;
    employeeId?: string;
    employeeName?: string;
    oldValue: string;
    newValue: string;
  }>;
}>;
```

---

## Business Rules & Invariants

1. Chain always starts with original submission (version 1)
2. Versions are sequential (1, 2, 3... no gaps)
3. Only latest version can be corrected (no branching)
4. Correction eligibility shown for latest version only
5. Full chain visible for audit purposes

---

## Edge Cases

1. **Very long chain (10+ corrections)** — Paginate or collapse older versions
2. **Original submission archived** — Show metadata, link to evidence store
3. **Correction in progress** — Show draft status in chain
4. **Multiple PAYE schemes** — Separate chains per scheme
5. **Prior year chain** — Show tax year indicator

---

## Tests

### corrections.router.test.ts
- Get submission chain with corrections
- Compare two submission versions
- Initiate correction workflow
- Block correction if ineligible
- Show correction eligibility correctly

---

## Verification

```bash
npm run typecheck
npm run test src/server/routers/hmrc-corrections.router.test.ts
npm run lint src/app/(bureau)/fps/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: System maintains submission chain
- 02-02-hmrc-submissions-spec.md § Data Models → submission_version, parent_submission_id
