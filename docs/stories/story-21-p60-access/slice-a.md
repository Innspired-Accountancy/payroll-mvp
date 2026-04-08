# Slice a: P60 List and View API/UI

**Story:** story-21-p60-access
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** epic-01-core-payroll (year-end processing), story-18-portal-auth (authentication)

---

## Goal

Create the P60 document list and view functionality that allows employees to access their year-end tax documents. P60s are only available for completed tax years where the employee was employed on 5 April.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: documents.listP60s(), documents.getP60(id)
- [x] External service endpoints: None (internal database queries)
- [x] Data contracts defined: P60ListItem, P60Detail, P60Totals
- [x] Configuration variables: P60_AVAILABLE_AFTER="05-31" (31 May)
- [x] Error scenarios identified: P60 not yet generated, unauthorized access, invalid tax year
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 2: Access P60
- 02-11-reporting-documents-spec.md — Year-end documents, P60 generation
- 02-02-hmrc-submissions-spec.md — FPS year-end data

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/documents.ts` | create | Document procedures including P60 |
| `src/app/(employee)/documents/page.tsx` | create | Documents list page |
| `src/app/(employee)/documents/p60/[id]/page.tsx` | create | P60 detail view |
| `src/components/documents/p60-list.tsx` | create | P60 list component |
| `src/components/documents/p60-detail.tsx` | create | P60 detail display |
| `src/lib/validation/documents.ts` | create | Document schemas |

---

## Responsibilities

1. List P60s grouped by tax year (e.g., "2025-2026")
2. Only show P60s for tax years where employee was employed on 5 April
3. Display P60 after year-end processing (available from 31 May)
4. Show employer info, employee info, and tax year totals
5. Include total pay, tax deducted, National Insurance contributions
6. Handle multiple employments (separate P60 per employment)
7. Log all access with IP and timestamp

---

## Contracts

### documents.listP60s
- **Method:** tRPC query `documents.listP60s`
- **Input:** None
- **Output:**
  ```typescript
  {
    p60s: [
      {
        id: string;                // UUID
        taxYear: string;           // "2025-2026"
        employerName: string;
        payeReference: string;
        employmentStartDate: string;  // ISO date
        employmentEndDate: string;    // null if still employed
        isAvailable: boolean;      // true if after 31 May
        availableFrom: string;     // ISO date (31 May following tax year)
      }
    ];
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
- **Auth:** Protected procedure with employee session

### documents.getP60
- **Method:** tRPC query `documents.getP60`
- **Input:**
  ```typescript
  {
    id: string;                // UUID of P60 record
  }
  ```
- **Output:**
  ```typescript
  {
    id: string;
    taxYear: string;           // "2025-2026"
    employer: {
      name: string;
      payeReference: string;
      address: string;
    };
    employee: {
      name: string;
      niNumber: string;
      worksNumber: string;     // Employee number
    };
    employment: {
      startDate: string;
      endDate: string | null;
      leavingDate: string | null;
    };
    totals: {
      taxablePay: number;      // Box 1: Pay
      taxDeducted: number;     // Box 2: Tax deducted
      finalTaxCode: string;    // Box 3: Final tax code
      nicTableLetter: string;  // Box 4: NIC table letter
      employeeNic: number;     // Box 5: Employee NIC
      employerNic: number;     // Not on P60 but tracked
      statutoryPayments: number;  // Box 6: SMP, SPP, etc.
    };
    generatedAt: string;       // ISO timestamp
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — P60 doesn't exist
  - `FORBIDDEN` — P60 belongs to different employee
  - `NOT_AVAILABLE` — P60 not yet available (before 31 May)
- **Auth:** Protected procedure with employee session validation

### P60ListItem Schema
```typescript
export const p60ListItemSchema = z.object({
  id: z.string().uuid(),
  taxYear: z.string().regex(/^\d{4}-\d{4}$/),
  employerName: z.string(),
  payeReference: z.string(),
  employmentStartDate: z.string().datetime(),
  employmentEndDate: z.string().datetime().nullable(),
  isAvailable: z.boolean(),
  availableFrom: z.string().datetime(),
});
```

---

## Business Rules & Invariants

1. P60 only generated if employee was employed on 5 April of tax year end
2. P60 available to employees from 31 May following tax year end
3. P60 data sourced from final FPS submission for the tax year
4. Multiple employments result in multiple P60s
5. P60 shows "to be continued" if employment ongoing (no end date)
6. All P60 access logged for HMRC compliance

---

## Edge Cases

1. **Not employed on 5 April** — No P60 generated, don't show in list
2. **Before 31 May** — Show "Available 31 May" message, prevent access
3. **Leaver during year** — P45 instead of P60, show in separate section
4. **Multiple employments** — List all P60s, group by tax year
5. **Year-end corrections** — Update P60 data if FPS amended

---

## Tests

### documents.router.test.ts
- listP60s returns only employee's P60s
- getP60 returns full P60 details
- Rejects access before availability date
- Validates employee ownership
- Returns empty list if no P60s

### p60-list.test.tsx
- Renders list of tax years
- Shows availability status
- Click navigates to detail
- Empty state when no P60s

### p60-detail.test.tsx
- Displays employer and employee info
- Shows all tax year totals
- Formats currency correctly
- Download button present

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/documents.test.ts
npx vitest run src/components/documents/p60-list.test.tsx
npx vitest run src/components/documents/p60-detail.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § P60 Access → Year-end document access
- 02-05-employee-portal-spec.md § Journey 2: Access P60
- 02-11-reporting-documents-spec.md § Journey 4: Generate Year-End P60s
- 02-02-hmrc-submissions-spec.md § FPS year-end data
