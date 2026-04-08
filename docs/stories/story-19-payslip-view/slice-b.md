# Slice b: Payslip Detail View with Itemized Breakdown

**Story:** story-19-payslip-view
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** slice-a (payslip list API and UI)

---

## Goal

Create the payslip detail view that displays full itemized breakdown including earnings, deductions, employer contributions, and tax details. This view provides employees with comprehensive payslip information matching the format of generated PDFs.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: payslips.getById(id)
- [x] External service endpoints: None (internal database queries)
- [x] Data contracts defined: PayslipDetail, EarningItem, DeductionItem, PayslipTotals
- [x] Configuration variables: None
- [x] Error scenarios identified: Payslip not found, unauthorized access, invalid ID format
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 1: View Payslip, API Contracts
- 02-01-core-payroll-spec.md — Payslip elements, calculations
- 02-11-reporting-documents-spec.md — Document display patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(employee)/payslips/[id]/page.tsx` | create | Payslip detail page |
| `src/components/payslips/payslip-detail.tsx` | create | Full payslip detail component |
| `src/components/payslips/earnings-section.tsx` | create | Earnings breakdown |
| `src/components/payslips/deductions-section.tsx` | create | Deductions breakdown |
| `src/components/payslips/payslip-header.tsx` | create | Employer and tax info |
| `src/server/routers/payslips.ts` | update | Add getById procedure |

---

## Responsibilities

1. Display employer name, tax code, NI number in header
2. Show itemized earnings (basic, bonus, overtime, etc.)
3. Show itemized deductions (tax, NI, pension, student loan, etc.)
4. Display employer contributions (pension, NI)
5. Show gross, deductions, net totals
6. Display pay date and period clearly
7. Provide download button for PDF
8. Log access with IP and timestamp

---

## Contracts

### payslips.getById
- **Method:** tRPC query `payslips.getById`
- **Input:**
  ```typescript
  {
    id: string;           // UUID of payslip
  }
  ```
- **Output:**
  ```typescript
  {
    id: string;
    employerName: string;
    payeReference: string;
    payDate: string;           // ISO date
    period: string;            // "April 2026"
    taxCode: string;           // "1257L"
    niNumber: string;          // "AB123456C"
    earnings: [
      {
        description: string;   // "Basic Salary"
        amount: number;        // 3000.00
        type: "fixed" | "variable";
      }
    ];
    deductions: [
      {
        description: string;   // "Income Tax"
        amount: number;        // 390.50
        type: "tax" | "nic" | "pension" | "other";
      }
    ];
    employerContributions: [
      {
        description: string;   // "Employer Pension"
        amount: number;        // 150.00
      }
    ];
    totals: {
      gross: number;           // 3500.00
      deductions: number;      // 785.82
      net: number;             // 2714.18
    };
    ytd: {
      taxablePay: number;
      taxPaid: number;
      nicPaid: number;
      pensionContributions: number;
    };
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — Payslip doesn't exist or doesn't belong to employee
  - `BAD_REQUEST` — Invalid ID format
- **Auth:** Protected procedure, validates payslip.employee_id matches session

### PayslipDetail Schema
```typescript
export const earningItemSchema = z.object({
  description: z.string(),
  amount: z.number().nonnegative(),
  type: z.enum(["fixed", "variable", "bonus", "overtime"]),
});

export const deductionItemSchema = z.object({
  description: z.string(),
  amount: z.number().nonnegative(),
  type: z.enum(["tax", "nic", "pension", "student_loan", "other"]),
});

export const payslipDetailSchema = z.object({
  id: z.string().uuid(),
  employerName: z.string(),
  payeReference: z.string(),
  payDate: z.string().datetime(),
  period: z.string(),
  taxCode: z.string(),
  niNumber: z.string(),
  earnings: z.array(earningItemSchema),
  deductions: z.array(deductionItemSchema),
  employerContributions: z.array(z.object({
    description: z.string(),
    amount: z.number().nonnegative(),
  })),
  totals: z.object({
    gross: z.number().nonnegative(),
    deductions: z.number().nonnegative(),
    net: z.number(),
  }),
  ytd: z.object({
    taxablePay: z.number().nonnegative(),
    taxPaid: z.number().nonnegative(),
    nicPaid: z.number().nonnegative(),
    pensionContributions: z.number().nonnegative(),
  }),
});
```

---

## Business Rules & Invariants

1. Employee can only view payslip details for their own payslips
2. NI number displayed in format AB123456C (masked as AB***56C if configured)
3. Tax code validated against HMRC format (e.g., 1257L, K475, BR, 0T)
4. Earnings and deductions sum to totals.gross and totals.deductions
5. Net pay = gross - deductions (validated server-side)
6. YTD values cumulative from tax year start (6 April)

---

## Edge Cases

1. **Zero net pay** — Display as £0.00, explain if all deductions
2. **Negative adjustment** — Show as negative earning or separate adjustment line
3. **Multiple tax codes** — Display primary tax code, note secondary in details
4. **Week 53** — Special handling for tax week 53 in period label
5. **Leaver payslip** — Show "Final payslip" indicator with P45 triggered

---

## Tests

### payslips.router.test.ts
- getById returns full payslip details
- Validates employee ownership
- Returns 404 for non-existent payslip
- Calculates totals correctly

### payslip-detail.test.tsx
- Renders all payslip sections
- Earnings and deductions displayed correctly
- Totals calculated and formatted
- Download button present and functional
- Back navigation to list

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/payslips.test.ts
npx vitest run src/components/payslips/payslip-detail.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Payslip View → Detail view with itemization
- 02-05-employee-portal-spec.md § API Contracts → GET /api/v1/portal/payslips/{id}
- 02-01-core-payroll-spec.md § Payslip elements → Earnings and deductions
