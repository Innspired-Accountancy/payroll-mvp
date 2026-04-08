# Slice a: Payslip List API and UI

**Story:** story-19-payslip-view
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** story-18-portal-auth (authentication)

---

## Goal

Create the payslip list API endpoint and UI component that displays an employee's payslip history with pagination. This is the entry point for payslip viewing and must enforce strict employee-only data access.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: payslips.list(query), payslips.getById(id)
- [x] External service endpoints: None (internal database queries)
- [x] Data contracts defined: PayslipListItem, PayslipListResponse, PayslipListQuery
- [x] Configuration variables: PAGE_SIZE=10
- [x] Error scenarios identified: Unauthorized access, payslip not found, database timeout
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 1: View Payslip, API Contracts
- 02-01-core-payroll-spec.md — Payslip data model
- 02-11-reporting-documents-spec.md — Document access patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/payslips.ts` | create | tRPC payslip procedures |
| `src/lib/db/schema/payslips.ts` | update | Add portal-specific views |
| `src/app/(employee)/payslips/page.tsx` | create | Payslip list page |
| `src/components/payslips/payslip-list.tsx` | create | Payslip list component |
| `src/components/payslips/payslip-card.tsx` | create | Individual payslip card |
| `src/lib/validation/payslips.ts` | create | Zod schemas for payslip queries |

---

## Responsibilities

1. Query payslips filtered by authenticated employee_id only
2. Return paginated list sorted by pay_date descending
3. Display pay period, gross pay, net pay in list view
4. Enforce strict data isolation at database query level
5. Log all access with IP address and timestamp
6. Support pagination controls (previous/next, page numbers)

---

## Contracts

### payslips.list
- **Method:** tRPC query `payslips.list`
- **Input:**
  ```typescript
  {
    page?: number;        // Default: 1
    pageSize?: number;    // Default: 10, Max: 50
  }
  ```
- **Output:**
  ```typescript
  {
    payslips: [
      {
        id: string;           // UUID
        payDate: string;      // ISO date (2026-04-30)
        period: string;       // "April 2026"
        grossPay: number;     // 3500.00
        netPay: number;       // 2750.00
        downloadUrl: string;  // "/api/payslips/uuid/download"
      }
    ];
    pagination: {
      page: number;
      pageSize: number;
      totalCount: number;
      totalPages: number;
    };
    ytdSummary: {
      taxablePay: number;     // Year-to-date taxable pay
      taxPaid: number;        // Year-to-date tax paid
      nicPaid: number;        // Year-to-date NIC paid
    };
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid or expired session
  - `BAD_REQUEST` — Invalid pagination parameters
  - `INTERNAL_SERVER_ERROR` — Database error
- **Auth:** Protected procedure with employee session (auto-filters by employee_id)

### PayslipListItem Schema
```typescript
export const payslipListItemSchema = z.object({
  id: z.string().uuid(),
  payDate: z.string().datetime(),
  period: z.string(),
  grossPay: z.number().nonnegative(),
  netPay: z.number().nonnegative(),
  downloadUrl: z.string().url(),
});
```

---

## Business Rules & Invariants

1. Employee can only view payslips where employee_id matches their session
2. Payslips sorted by pay_date descending (newest first)
3. YTD summary calculated from current tax year payslips only
4. Period displayed in format "Month YYYY" or "Week NN YYYY"
5. Amounts displayed with 2 decimal places in GBP
6. All payslip access logged to EmployeeDocumentAccess table

---

## Edge Cases

1. **No payslips found** — Display empty state with helpful message
2. **Single payslip** — Show pagination as disabled, totalPages=1
3. **Large payslip history** — Pagination with page size selector (10/25/50)
4. **Tax year boundary** — YTD summary resets in new tax year
5. **Multiple employments** — Group by employment, show employer name

---

## Tests

### payslips.router.test.ts
- List returns only current employee's payslips
- Pagination works correctly (page, pageSize)
- YTD summary calculated correctly
- Empty list returns appropriate response
- Unauthorized access rejected

### payslip-list.test.tsx
- Renders list of payslip cards
- Pagination controls functional
- Clicking card navigates to detail
- Empty state displayed when no payslips
- Loading state during fetch

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/payslips.test.ts
npx vitest run src/components/payslips/payslip-list.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Payslip View → List and detail views
- 02-05-employee-portal-spec.md § API Contracts → GET /api/v1/portal/payslips
- 02-05-employee-portal-spec.md § Data Models → EmployeeDocumentAccess
