# Slice c: Pay Run Data Aggregation

**Story:** story-01-fps-generator
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Aggregate payroll data from approved pay runs into FPS-ready format, calculating totals, handling multiple employments, and preparing employee-level records with all RTI-required fields.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x for queries
- [x] SDK methods identified: db.query.payRuns.findFirst(), db.query.payslips.findMany()
- [x] External service endpoints: N/A (internal aggregation)
- [x] Data contracts defined: PayRunFPSData, EmployeeFPSRecord, FPSAggregateResult
- [x] Configuration: N/A
- [x] Error scenarios: Missing employee data, incomplete calculations, schema mismatch
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Pay run data structure
- 02-02-hmrc-submissions-spec.md § Data Models

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/aggregation/payrun-aggregator.ts` | create | Pay run data aggregation logic |
| `src/lib/hmrc/aggregation/employee-adapter.ts` | create | Employee data transformation |
| `src/lib/hmrc/aggregation/totals-calculator.ts` | create | FPS totals calculation |

---

## Responsibilities

1. Query approved pay run with all related payslips
2. Aggregate employee-level RTI data (pay, tax, NICs, statutory payments)
3. Calculate submission-level totals
4. Handle employees with multiple employments
5. Validate data completeness before FPS generation

---

## Contracts

### PayRunAggregator.aggregate()
- **Method:** `aggregate(payRunId: string): Promise<PayRunFPSData>`
- **Input:** `payRunId: string` — UUID of approved pay run
- **Output:**
  ```typescript
  interface PayRunFPSData {
    employer: {
      id: string;
      payeReference: string;
      accountsOfficeReference: string;
      name: string;
      address: Address;
    };
    payeScheme: {
      id: string;
      taxYear: string;
      taxPeriod: number;
      periodType: 'monthly' | 'weekly';
    };
    payRun: {
      id: string;
      paymentDate: Date;
      periodStart: Date;
      periodEnd: Date;
    };
    employees: EmployeeFPSRecord[];
    totals: FPSTotals;
  }
  ```
- **Errors:**
  - `PayRunNotFoundError` — Code: PAYRUN_NOT_FOUND
  - `PayRunNotApprovedError` — Code: PAYRUN_NOT_APPROVED
  - `IncompleteDataError` — Code: INCOMPLETE_DATA, details: missing fields
- **Auth:** Internal service, caller must have payrun:read permission

### EmployeeFPSRecord
```typescript
interface EmployeeFPSRecord {
  employeeId: string;
  nino: string;
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
  gender: 'M' | 'F';
  employmentId: string;
  taxCode: string;
  paymentDate: Date;
  taxablePay: Decimal;
  taxDeducted: Decimal;
  employeeNICs: Decimal;
  employerNICs: Decimal;
  studentLoanDeduction?: Decimal;
  sspPaid?: Decimal;
  smpPaid?: Decimal;
  sppPaid?: Decimal;
  sapPaid?: Decimal;
  hoursWorked: 'A' | 'B' | 'C' | 'D'; // A=<=16, B=16-23.99, C=24-29.99, D=>=30
  leavingDate?: Date;
  isDirector: boolean;
  directorNicCalculation?: 'AL' | 'AN' | 'AM';
}
```

### FPSTotals
```typescript
interface FPSTotals {
  employeeCount: number;
  taxablePayTotal: Decimal;
  taxDeductedTotal: Decimal;
  employeeNICsTotal: Decimal;
  employerNICsTotal: Decimal;
  studentLoansTotal?: Decimal;
  sspTotal?: Decimal;
  smpTotal?: Decimal;
  sppTotal?: Decimal;
  sapTotal?: Decimal;
}
```

---

## Business Rules & Invariants

1. Only approved pay runs can be aggregated for FPS
2. All employees must have valid NINO for FPS inclusion
3. Employees with zero taxable pay are still included if they had payments
4. Hours worked categories: A (<=16), B (16-23.99), C (24-29.99), D (>=30)
5. Director NIC calculations: AL (annual), AN (pro-rata), AM (alternative)
6. Statutory payments are reported gross, not net

---

## Edge Cases

1. **Employee with multiple payslips** — Aggregate across all payslips for the period
2. **Employee left during period** — Include leaving date
3. **New starter** — Ensure start date is before or within period
4. **Director with annual NIC** — Flag and use correct calculation method
5. **Statutory payments only** — Report even if taxable pay is zero

---

## Tests

### payrun-aggregator.test.ts
- Aggregate single employee pay run
- Aggregate 50 employee pay run with multiple payslips
- Handle employee with statutory payments only
- Calculate totals correctly
- Reject unapproved pay run
- Handle missing NINO (skip employee with warning)
- Handle multiple employments for same employee

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/aggregation/payrun-aggregator.test.ts
npm run lint src/lib/hmrc/aggregation/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/pay-runs/{id}/generate-fps
- 02-01-core-payroll-spec.md — Pay run calculation details
