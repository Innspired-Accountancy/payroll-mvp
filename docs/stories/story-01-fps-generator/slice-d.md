# Slice d: FPS Generation API

**Story:** story-01-fps-generator
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-c

---

## Goal

Expose FPS generation functionality through tRPC procedures with proper authentication, validation, and error handling. Store generated FPS in database and return generation results with validation summary.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, Zod 3.22.x, Drizzle ORM 0.30.x
- [x] SDK methods identified: tRPC protectedProcedure, zod input validation
- [x] External service endpoints: N/A (internal API)
- [x] Data contracts defined: GenerateFPSInput, GenerateFPSOutput Zod schemas
- [x] Configuration: N/A
- [x] Error scenarios: TRPCError codes BAD_REQUEST, NOT_FOUND, FORBIDDEN, INTERNAL_SERVER_ERROR
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/pay-runs/{id}/generate-fps
- 08-architecture-and-patterns.md — tRPC router patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/hmrc-submissions.ts` | create | tRPC router for HMRC submissions |
| `src/server/routers/index.ts` | update | Register new router |
| `src/lib/validation/hmrc-submissions.ts` | update | Add FPS generation schemas |

---

## Responsibilities

1. Define tRPC procedure for FPS generation
2. Validate pay run exists and is approved
3. Call aggregation and generation services
4. Store generated FPS in database
5. Return generation result with summary statistics

---

## Contracts

### hmrc.generateFPS
- **Method:** tRPC mutation `hmrc.generateFPS`
- **Input:** GenerateFPSInput Zod schema
  ```typescript
  const generateFPSInput = z.object({
    payRunId: z.string().uuid(),
    lateReason: z.enum(['H', 'I', 'J', 'K', 'L', 'M']).optional(),
    isFinal: z.boolean().optional().default(false),
    dryRun: z.boolean().optional().default(false) // Validate only, don't store
  });
  ```
- **Output:** GenerateFPSOutput
  ```typescript
  const generateFPSOutput = z.object({
    submissionId: z.string().uuid(),
    status: z.enum(['draft', 'validated']),
    correlationId: z.string(),
    employeeCount: z.number().int(),
    totals: z.object({
      taxablePay: z.number(),
      taxDeducted: z.number(),
      employeeNICs: z.number(),
      employerNICs: z.number()
    }),
    generatedAt: z.string().datetime(),
    xmlSize: z.number().int() // Bytes
  });
  ```
- **Errors:**
  - `NOT_FOUND` — Pay run not found
  - `BAD_REQUEST` — Pay run not approved or invalid input
  - `FORBIDDEN` — User lacks hmrc:fps:generate permission
  - `INTERNAL_SERVER_ERROR` — Generation failure
- **Auth:** Protected procedure with permission check `hmrc:fps:generate`

### Implementation Flow
1. Validate input with Zod
2. Check user has access to pay run's employer (tenant isolation)
3. Verify pay run exists and status is 'approved'
4. Call PayRunAggregator.aggregate(payRunId)
5. Call FpsGenerator.generate(aggregatedData)
6. If not dryRun: Insert into hmrc_submissions table
7. Return generation result

---

## Business Rules & Invariants

1. Only users with hmrc:fps:generate permission can generate FPS
2. Pay run must be in 'approved' status
3. User must have access to the pay run's employer (tenant isolation)
4. Generated FPS is stored with status 'draft' awaiting validation
5. Correlation ID is generated as SUB-{timestamp}-{random}

---

## Edge Cases

1. **Concurrent generation requests** — Row-level lock on pay_run_id
2. **Large pay runs (1000+ employees)** — Async generation with progress tracking
3. **Generation failure mid-way** — Transaction rollback, error logged
4. **Dry run mode** — XML validated but not stored, no database insert

---

## Tests

### hmrc-submissions.router.test.ts
- Generate FPS with valid pay run
- Reject FPS generation for non-existent pay run (404)
- Reject FPS generation for unapproved pay run (400)
- Reject FPS generation without permission (403)
- Dry run returns XML without database insert
- Validate tenant isolation (can't generate for other employer)

---

## Verification

```bash
npm run typecheck
npm run test src/server/routers/hmrc-submissions.router.test.ts
npm run lint src/server/routers/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/pay-runs/{id}/generate-fps
- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: Submit FPS for Pay Run
