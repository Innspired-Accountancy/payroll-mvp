# Slice c: YTD Summary and Access Logging

**Story:** story-19-payslip-view
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** slice-a (payslip list API)

---

## Goal

Implement year-to-date summary calculation and comprehensive access logging for payslip viewing. This ensures compliance with audit requirements and provides employees with cumulative payroll information.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: audit.logDocumentAccess(data)
- [x] External service endpoints: None
- [x] Data contracts defined: YTDSummary, EmployeeDocumentAccessLog, TaxYearConfig
- [x] Configuration variables: TAX_YEAR_START_MONTH=4, TAX_YEAR_START_DAY=6
- [x] Error scenarios identified: Tax year boundary edge cases, logging failures
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Data Models, Security (access logging)
- 02-10-audit-compliance-spec.md — Audit logging requirements
- 02-01-core-payroll-spec.md — Tax year handling

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/audit.ts` | update | Add EmployeeDocumentAccess table |
| `src/lib/audit/document-access.ts` | create | Document access logging utilities |
| `src/server/routers/payslips.ts` | update | Add YTD calculation and logging |
| `src/components/payslips/ytd-summary.tsx` | create | YTD summary display component |
| `src/lib/calculations/tax-year.ts` | create | Tax year utility functions |

---

## Responsibilities

1. Calculate YTD totals from tax year start (6 April) to current payslip
2. Include taxable pay, tax paid, NIC paid, pension contributions
3. Log every payslip list view and detail view access
4. Store access logs with IP address, timestamp, document type
5. Handle tax year boundaries correctly (new tax year = reset YTD)
6. Display YTD summary on both list and detail views

---

## Contracts

### calculateYTD
- **Method:** Utility function `calculateYTD(employeeId, upToDate)`
- **Input:**
  ```typescript
  {
    employeeId: string;     // UUID
    upToDate: string;       // ISO date (calculate YTD up to this date)
  }
  ```
- **Output:**
  ```typescript
  {
    taxablePay: number;
    taxPaid: number;
    nicPaid: number;
    employeePension: number;
    employerPension: number;
    taxYear: string;        // "2025-2026"
  }
  ```

### logDocumentAccess
- **Method:** Database insert to `employee_document_access` table
- **Input:**
  ```typescript
  {
    employeeId: string;
    documentType: "payslip" | "p60" | "p45";
    documentId: string;
    accessedAt: Date;
    ipAddress: string;
    userAgent: string;
  }
  ```
- **Output:** void (fire-and-forget with error handling)

### YTDSummary Schema
```typescript
export const ytdSummarySchema = z.object({
  taxablePay: z.number().nonnegative(),
  taxPaid: z.number().nonnegative(),
  nicPaid: z.number().nonnegative(),
  employeePension: z.number().nonnegative(),
  employerPension: z.number().nonnegative(),
  taxYear: z.string().regex(/^\d{4}-\d{4}$/),  // "2025-2026"
});
```

---

## Business Rules & Invariants

1. Tax year runs from 6 April to 5 April following year
2. YTD calculated from 6 April of current tax year up to and including specified date
3. All document access logged regardless of success/failure
4. Access logs retained for 6 years per HMRC requirements
5. IP address stored for security auditing
6. YTD values updated in real-time (not cached) for accuracy

---

## Edge Cases

1. **Before tax year start** — If date < 6 April, use previous tax year
2. **New tax year (no payslips)** — YTD shows all zeros
3. **Cross-year payslip view** — Viewing April payslip in May shows YTD including April
4. **Multiple employments** — YTD per employment, not aggregated
5. **Logging failure** — Log to secondary log, don't block user access

---

## Tests

### tax-year.test.ts
- Tax year calculation for dates before/after 6 April
- Leap year handling
- Tax year string formatting

### document-access.test.ts
- Access log created on list view
- Access log created on detail view
- IP address captured correctly
- Handles logging failures gracefully

### ytd-summary.test.tsx
- Renders YTD values correctly
- Formats currency with £ symbol
- Shows tax year label
- Updates when payslip changes

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/lib/calculations/tax-year.test.ts
npx vitest run src/lib/audit/document-access.test.ts
npx vitest run src/components/payslips/ytd-summary.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Payslip View → YTD summary
- 02-05-employee-portal-spec.md § Data Models → EmployeeDocumentAccess
- 02-05-employee-portal-spec.md § Security → All access logged with IP and timestamp
- 02-10-audit-compliance-spec.md § Audit logging requirements
