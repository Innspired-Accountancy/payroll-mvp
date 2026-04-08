# Slice b: XML Generation Engine

**Story:** story-01-fps-generator
**Epic:** epic-02-hmrc-submissions
**Effort:** L
**Dependencies:** slice-a

---

## Goal

Build a type-safe XML generation engine for RTI FPS documents using template-based construction. Support HMRC RTI schema v2025-26 with proper namespace handling, date formatting, and monetary value precision.

---

## Decision Checklist

- [x] All libraries/packages named: fast-xml-parser 4.3.x, handlebars 4.7.x
- [x] SDK methods identified: XMLBuilder.build(), XMLValidator.validate()
- [x] External service endpoints: N/A (internal generation)
- [x] Data contracts defined: FPSData, EmployeeFPSRecord, FPSTotals interfaces
- [x] Configuration: HMRC_SCHEMA_VERSION environment variable (default: "2025-26")
- [x] Error scenarios: Template errors, data type mismatches, missing required fields
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/pay-runs/{id}/generate-fps
- 08-architecture-and-patterns.md — XML generation patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/xml-builder.ts` | create | Core XML builder utilities |
| `src/lib/hmrc/templates/fps-template.hbs` | create | Handlebars FPS template |
| `src/lib/hmrc/fps-generator.ts` | create | FPS generation orchestrator |
| `src/lib/hmrc/types/fps.ts` | create | TypeScript interfaces for FPS data |

---

## Responsibilities

1. Build well-formed XML with proper RTI namespace declarations
2. Format dates as YYYY-MM-DD per HMRC specification
3. Format monetary values with 2 decimal places
4. Handle employee-level and submission-level data
5. Generate unique Message ID and timestamp

---

## Contracts

### FpsGenerator.generate()
- **Method:** `generate(payRunId: string, options: FPSOptions): Promise<GenerationResult>`
- **Input:** 
  ```typescript
  interface FPSOptions {
    payRunId: string;
    lateReason?: 'H' | 'I' | 'J' | 'K' | 'L' | 'M';
    isFinal?: boolean;
    testScenario?: number; // HMRC test scenario ID
  }
  ```
- **Output:**
  ```typescript
  interface GenerationResult {
    xml: string;
    correlationId: string;
    employeeCount: number;
    totals: FPSTotals;
    generatedAt: Date;
  }
  ```
- **Errors:** 
  - `PayRunNotFoundError` — Pay run doesn't exist
  - `PayRunNotApprovedError` — Pay run not in approved status
  - `MissingDataError` — Required employee or employer data missing
- **Auth:** Requires hmrc:fps:generate permission

### FPS XML Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<EmployerPaymentSubmission xmlns="http://www.govtalk.gov.uk/taxation/PAYE/RTI/EmployerPaymentSubmission/25-26">
  <MessageHeader>
    <MessageId>{uuid}</MessageId>
    <CreationTimestamp>{ISO8601}</CreationTimestamp>
  </MessageHeader>
  <Employer>
    <EmployerPayeReference>{format: 123/AB45678}</EmployerPayeReference>
    <AccountsOfficeReference>{format: 123PA12345678}</AccountsOfficeReference>
    <RelatedTaxYear>{format: 26-27}</RelatedTaxYear>
  </Employer>
  <FullPaymentSubmission>
    <SubmissionId>{correlationId}</SubmissionId>
    <HMRCsiteIdentifier>0</HMRCsiteIdentifier>
    <DateFormReceived>{YYYY-MM-DD}</DateFormReceived>
    <TaxYear>{26-27}</TaxYear>
    <TaxMonth>{1-12}</TaxMonth>
    <LateReportingReason>{H-M}</LateReportingReason>
    <FinalSubmission>{yes|no}</FinalSubmission>
    <!-- Employee records -->
    <Employee>...</Employee>
    <!-- Payment totals -->
    <TotalPayments>...</TotalPayments>
  </FullPaymentSubmission>
</EmployerPaymentSubmission>
```

### Employee Record Structure
```xml
<Employee>
  <EmployeeDetails>
    <Nino>{format: AB123456C}</Nino>
    <Name>
      <Forename>{first name}</Forename>
      <Surname>{last name}</Surname>
    </Name>
    <BirthDate>{YYYY-MM-DD}</BirthDate>
    <Gender>{M|F}</Gender>
  </EmployeeDetails>
  <Employment>
    <PaymentDate>{YYYY-MM-DD}</PaymentDate>
    <TaxablePay>{decimal}</TaxablePay>
    <TaxDeductedOrRefunded>{decimal}</TaxDeductedOrRefunded>
    <TaxCode>{format: 1257L}</TaxCode>
    <NumberOfHoursWorked>{A|B|C|D}</NumberOfHoursWorked>
    <!-- Statutory payments -->
    <SSP>{decimal}</SSP>
    <SMP>{decimal}</SMP>
    <!-- Student loan deductions -->
    <StudentLoanDeduction>{decimal}</StudentLoanDeduction>
    <!-- NICs -->
    <EmployeeNICs>{decimal}</EmployeeNICs>
    <EmployerNICs>{decimal}</EmployerNICs>
  </Employment>
</Employee>
```

---

## Business Rules & Invariants

1. MessageId must be UUID v4 format
2. CreationTimestamp must be UTC in ISO8601 format
3. EmployerPayeReference format: 3 digits, slash, 1-2 letters, 4-6 digits
4. AccountsOfficeReference format: 3 digits, "PA", 8 alphanumeric
5. TaxablePay and TaxDeducted must have exactly 2 decimal places
6. PaymentDate must be within the reported tax month
7. NINO format validation: 2 letters, 6 digits, 1 letter (A-D, exclude D/F/I/Q/U/V)

---

## Edge Cases

1. **Employee with multiple employments** — Generate separate Employee element for each
2. **Zero values** — Include element with 0.00, don't omit
3. **Special characters in names** — XML escape: & < > " '
4. **Very long names** — Truncate to HMRC maximum lengths
5. **Leavers in period** — Include leaving date if applicable

---

## Tests

### fps-generator.test.ts
- Generate FPS for single employee
- Generate FPS for 100+ employees
- Validate XML is well-formed
- Verify date formatting (YYYY-MM-DD)
- Verify monetary formatting (2 decimal places)
- Handle missing pay run (throws PayRunNotFoundError)
- Handle unapproved pay run (throws PayRunNotApprovedError)
- Handle special characters in names (proper XML escaping)

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/fps-generator.test.ts
npm run lint src/lib/hmrc/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: Submit FPS for Pay Run
- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmission entity
