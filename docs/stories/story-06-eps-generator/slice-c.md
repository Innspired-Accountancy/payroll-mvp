# Slice c: EPS Validation and API

**Story:** story-06-eps-generator
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement EPS validation using shared XSD validator and expose EPS generation through tRPC procedures. Store EPS submissions and integrate with existing submission infrastructure.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, XState 5.x (shared state machine)
- [x] SDK methods identified: XsdValidator.validate(), SubmissionService.submit()
- [x] External service endpoints: HMRC Gateway (shared with FPS)
- [x] Data contracts defined: GenerateEPSInput, EPSSubmissionOutput
- [x] Configuration: N/A
- [x] Error scenarios: Validation failure, submission failure, deadline exceeded
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/paye-schemes/{id}/generate-eps
- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/hmrc-submissions/{id}/submit

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/hmrc-submissions.ts` | update | Add EPS generation procedure |
| `src/lib/hmrc/eps/validator.ts` | create | EPS-specific validation rules |
| `src/app/(bureau)/eps/page.tsx` | create | EPS management page |

---

## Responsibilities

1. Add tRPC procedure for EPS generation
2. Validate EPS against HMRC schema
3. Apply EPS-specific business rules
4. Store EPS with type='eps' in submissions table
5. Reuse submission service for HMRC gateway submission

---

## Contracts

### hmrc.generateEPS
- **Method:** tRPC mutation `hmrc.generateEPS`
- **Input:**
  ```typescript
  const generateEPSInput = z.object({
    payeSchemeId: z.string().uuid(),
    taxYear: z.string().regex(/^\d{4}-\d{2}$/),
    taxMonth: z.number().int().min(1).max(12),
    includeRecoveries: z.boolean().default(true),
    includeEmploymentAllowance: z.boolean().default(true),
    includeCIS: z.boolean().default(false),
    noPaymentToDeclare: z.boolean().default(false),
    dryRun: z.boolean().default(false)
  }).refine(data => {
    // No payment to declare excludes all amounts
    if (data.noPaymentToDeclare) {
      return !data.includeRecoveries && !data.includeEmploymentAllowance && !data.includeCIS;
    }
    return true;
  }, { message: "No payment to declare cannot include recoveries, EA, or CIS" });
  ```
- **Output:**
  ```typescript
  const generateEPSOutput = z.object({
    submissionId: z.string().uuid(),
    status: z.enum(['draft', 'validated']),
    correlationId: z.string(),
    type: z.literal('eps'),
    taxYear: z.string(),
    taxMonth: z.number(),
    recoveries: z.object({
      ssp: z.number(),
      smp: z.number(),
      spp: z.number(),
      sap: z.number(),
      nicCompensation: z.number()
    }).optional(),
    employmentAllowance: z.object({
      claimed: z.boolean(),
      amount: z.number()
    }).optional(),
    cisDeductions: z.number().optional(),
    deadline: z.string().datetime(), // 19th of following month
    generatedAt: z.string().datetime()
  });
  ```
- **Errors:**
  - `NOT_FOUND` — PAYE scheme not found
  - `BAD_REQUEST` — Invalid combination or past deadline
  - `FORBIDDEN` — User lacks hmrc:eps:generate permission
- **Auth:** Protected with `hmrc:eps:generate` permission

### EPS-Specific Business Rules
| Rule | Severity | Description |
|------|----------|-------------|
| EPS001 | Blocker | Tax month outside current/prior year |
| EPS002 | Blocker | Deadline exceeded (19th passed, no reasonable excuse) |
| EPS003 | Warning | Recovery amount differs from calculated by >£1 |
| EPS004 | Blocker | Employment Allowance claimed when ineligible |
| EPS005 | Blocker | CIS deductions without contractor flag |

### Submission Deadline Calculation
```typescript
function getEPSDeadline(taxYear: string, taxMonth: number): Date {
  // EPS must be submitted by 19th of following month
  const year = parseInt(taxYear.split('-')[0]);
  const month = taxMonth; // 1-12
  
  // For tax month 1 (April), deadline is 19 May
  const deadlineMonth = month === 12 ? 1 : month + 1;
  const deadlineYear = month === 12 ? year + 1 : year;
  
  return new Date(deadlineYear, deadlineMonth - 1, 19);
}
```

---

## Business Rules & Invariants

1. EPS uses same state machine and submission infrastructure as FPS
2. EPS-specific validation rules run in addition to XSD validation
3. Deadline for EPS is 19th of month following tax month
4. EPS can be submitted for current or prior tax year only
5. Submission type='eps' distinguishes from FPS in database

---

## Edge Cases

1. **19th falls on weekend** — Deadline is next working day
2. **EPS generated after FPS for same period** — Allowed, separate submissions
3. **Multiple EPS for same period** — Only one should claim EA
4. **Final EPS of tax year** — Include year-end flags

---

## Tests

### eps.router.test.ts
- Generate EPS with recoveries
- Reject EPS with invalid combination
- Check deadline calculation
- Validate EA eligibility
- Submit EPS through shared submission service

---

## Verification

```bash
npm run typecheck
npm run test src/server/routers/hmrc-submissions.eps.test.ts
npm run lint src/app/(bureau)/eps/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/paye-schemes/{id}/generate-eps
- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 3: Submit EPS for Adjustments
