# Slice b: Error-to-Employee/Field Mapping

**Story:** story-05-error-handling
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Map HMRC errors to specific employees and payroll data fields using XPath analysis and NINO matching. Enable precise error location and targeted corrections.

---

## Decision Checklist

- [x] All libraries/packages named: xpath 0.0.34 (XPath evaluator)
- [x] SDK methods identified: xpath.select(), DOMParser
- [x] External service endpoints: N/A (internal mapping)
- [x] Data contracts defined: ErrorMapping, FieldMapping interfaces
- [x] Configuration: N/A
- [x] Error scenarios: XPath not found, NINO mismatch, ambiguous mapping
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Manager views error details
- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmissionError.field_path

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/errors/mapper.ts` | create | Error-to-field mapper |
| `src/lib/hmrc/errors/xpath-parser.ts` | create | XPath extraction utilities |
| `src/lib/hmrc/errors/field-lookup.ts` | create | Field-to-payroll mapping |

---

## Responsibilities

1. Parse XPath locations from HMRC errors
2. Map XPath to employee array index
3. Extract NINO from XPath context
4. Map XML fields to payroll database fields
5. Store mappings for correction workflow

---

## Contracts

### ErrorMapper.mapErrors()
- **Method:** `async mapErrors(errors: ParsedError[], submissionId: string): Promise<MappedError[]>`
- **Input:**
  - `errors: ParsedError[]` — Errors from HMRC rejection
  - `submissionId: string` — Submission UUID
- **Output:** MappedError[]
  ```typescript
  interface MappedError {
    id: string; // Generated UUID
    submissionId: string;
    errorCode: string;
    message: string;
    severity: string;
    // Mapping info
    employeeId?: string; // Database employee ID
    employeeNino?: string;
    employeeName?: string;
    fieldPath: string; // XPath from HMRC
    payrollField?: string; // Mapped to payroll schema
    currentValue?: string;
    // Display info
    guidance: string;
    uiLocation: string; // Human-readable location
  }
  ```

### XPath to Field Mapping
| XPath Pattern | Payroll Field | UI Location |
|---------------|---------------|-------------|
| `/EmployeePaymentSubmission/Employer/EmployerPayeReference` | employer.paye_reference | Settings > PAYE Scheme |
| `/EmployeePaymentSubmission/FullPaymentSubmission/Employee[N]/EmployeeDetails/Nino` | employee.ni_number | Employee > Tax Details |
| `/EmployeePaymentSubmission/FullPaymentSubmission/Employee[N]/EmployeeDetails/Name/Forename` | employee.first_name | Employee > Personal |
| `/EmployeePaymentSubmission/FullPaymentSubmission/Employee[N]/Employment/TaxablePay` | payslip.taxable_pay | Pay Run > Payslip |
| `/EmployeePaymentSubmission/FullPaymentSubmission/Employee[N]/Employment/TaxCode` | employment.tax_code | Employee > Tax Details |
| `/EmployeePaymentSubmission/FullPaymentSubmission/Employee[N]/Employment/PaymentDate` | pay_run.payment_date | Pay Run > Details |

### Employee Resolution from XPath
```typescript
function resolveEmployeeFromXPath(
  xpath: string,
  submissionId: string
): Promise<EmployeeResolutionResult> {
  // 1. Parse employee index from XPath (e.g., Employee[3])
  // 2. Query submission for NINO at that position
  // 3. Lookup employee by NINO in employer's records
  // 4. Return employee ID, name, current field value
}
```

---

## Business Rules & Invariants

1. Every mappable error must have employeeId or submission-level indicator
2. XPath parsing is deterministic (same XPath = same field)
3. Employee lookup by NINO falls back to historical records
4. Unmapped errors are flagged for manual review
5. Field mappings are versioned per tax year

---

## Edge Cases

1. **Employee not found by NINO** — Flag for manual resolution
2. **Employee index out of bounds** — Log error, map to submission level
3. **Multiple employees with same NINO** — Use most recent, flag for review
4. **Field no longer exists (correction scenario)** — Show XML excerpt
5. **XPath uses namespace prefixes** — Normalize before parsing

---

## Tests

### error-mapper.test.ts
- Map employee-specific error to correct employee
- Map employer-level error (no employee)
- Handle XPath with array index
- Handle employee not found scenario
- Map multiple errors for same employee

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/errors/mapper.test.ts
npm run lint src/lib/hmrc/errors/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Manager views error details linked to payroll data
- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmissionError with employee_id and field_path
