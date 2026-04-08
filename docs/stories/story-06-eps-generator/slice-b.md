# Slice b: EPS XML Generation Engine

**Story:** story-06-eps-generator
**Epic:** epic-02-hmrc-submissions
**Effort:** L
**Dependencies:** slice-a

---

## Goal

Build EPS XML generation engine using shared XML infrastructure from FPS. Generate Employer Payment Summary XML with recovery amounts, Employment Allowance declarations, and CIS deductions.

---

## Decision Checklist

- [x] All libraries/packages named: fast-xml-parser 4.3.x, handlebars 4.7.x
- [x] SDK methods identified: XMLBuilder (reused from FPS)
- [x] External service endpoints: N/A (internal generation)
- [x] Data contracts defined: EPSGenerationOptions, EPSTemplateData interfaces
- [x] Configuration: HMRC_SCHEMA_VERSION
- [x] Error scenarios: Template errors, missing recovery data, invalid combinations
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 3: System generates EPS XML
- HMRC RTI Employer Payment Summary Schema v25-26

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/eps/generator.ts` | create | EPS generation orchestrator |
| `src/lib/hmrc/templates/eps-template.hbs` | create | Handlebars EPS template |
| `src/lib/hmrc/eps/employment-allowance.ts` | create | Employment Allowance logic |

---

## Responsibilities

1. Generate EPS XML with recovery amounts
2. Include Employment Allowance claim if applicable
3. Include CIS deductions suffered if applicable
4. Handle "no payment to declare" scenario
5. Validate EPS-specific business rules

---

## Contracts

### EpsGenerator.generate()
- **Method:** `generate(options: EPSGenerationOptions): Promise<EPSGenerationResult>`
- **Input:**
  ```typescript
  interface EPSGenerationOptions {
    payeSchemeId: string;
    taxYear: string;      // e.g., "2026-27"
    taxMonth: number;     // 1-12
    includeRecoveries: boolean;
    includeEmploymentAllowance: boolean;
    includeCIS: boolean;
    noPaymentToDeclare: boolean;
  }
  ```
- **Output:**
  ```typescript
  interface EPSGenerationResult {
    xml: string;
    correlationId: string;
    type: 'eps';
    taxYear: string;
    taxMonth: number;
    recoveries?: {
      ssp: Decimal;
      smp: Decimal;
      spp: Decimal;
      sap: Decimal;
      nicCompensation: Decimal;
    };
    employmentAllowance?: {
      claimed: boolean;
      amount: Decimal;
    };
    cisDeductions?: Decimal;
    totalOffset: Decimal;
    generatedAt: Date;
  }
  ```

### EPS XML Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<EmployerPaymentSummary xmlns="http://www.govtalk.gov.uk/taxation/PAYE/RTI/EmployerPaymentSummary/25-26">
  <MessageHeader>
    <MessageId>{uuid}</MessageId>
    <CreationTimestamp>{ISO8601}</CreationTimestamp>
  </MessageHeader>
  <Employer>
    <EmployerPayeReference>{format: 123/AB45678}</EmployerPayeReference>
    <AccountsOfficeReference>{format: 123PA12345678}</AccountsOfficeReference>
    <RelatedTaxYear>{format: 26-27}</RelatedTaxYear>
  </Employer>
  <PaymentPeriod>
    <Month>{1-12}</Month>
    <Year>{26-27}</Year>
    <NoPaymentToDeclare>{yes|no}</NoPaymentToDeclare>
  </PaymentPeriod>
  <RecoverableAmountsSMP>
    <AmountSMP>{decimal}</AmountSMP>
    <AmountSPP>{decimal}</AmountSPP>
    <AmountSAP>{decimal}</AmountSAP>
    <AmountSSP>{decimal}</AmountSSP>
    <NICCompensationOnSMP>{decimal}</NICCompensationOnSMP>
    <NICCompensationOnSPP>{decimal}</NICCompensationOnSPP>
    <NICCompensationOnSAP>{decimal}</NICCompensationOnSAP>
    <NICCompensationOnSSP>{decimal}</NICCompensationOnSSP>
    <Total>{decimal}</Total>
  </RecoverableAmountsSMP>
  <EmploymentAllowance>
    <Claimed>{yes|no}</Claimed>
    <EmploymentAllowanceInd>{yes|no}</EmploymentAllowanceInd>
  </EmploymentAllowance>
  <CISdeductionsSuffered>
    <Total>{decimal}</Total>
  </CISdeductionsSuffered>
</EmployerPaymentSummary>
```

### Employment Allowance Rules
```typescript
interface EmploymentAllowanceConfig {
  // Eligibility
  maxEmployerNICsPreviousYear: 100000; // Disqualified if exceeded
  
  // Claim amount (2026-27 rate)
  annualAllowance: 10470; // £10,470 per year
  
  // Claim can be spread across tax year
  maxMonthlyClaim: Decimal; // annualAllowance / 12
}
```

---

## Business Rules & Invariants

1. EPS can only claim recoveries for current or prior tax year
2. Employment Allowance only claimable if eligible (NICs <= £100k previous year)
3. CIS deductions only included if employer is also contractor
4. No Payment To Declare requires no recoveries, no EA claim, no CIS
5. Monthly EPS deadline: 19th of following month

---

## Edge Cases

1. **Zero recovery but EA claim** — Valid, include EA only
2. **Recovery exceeds liability** — Claim full recovery, carry forward
3. **EA claim exceeds NIC liability** — Claim up to liability, carry forward
4. **Mid-year EA eligibility change** — Stop claiming if disqualified
5. **Final EPS of year** — Finalize EA claim amount

---

## Tests

### eps-generator.test.ts
- Generate EPS with recoveries only
- Generate EPS with EA claim
- Generate EPS with CIS deductions
- Generate EPS with no payment to declare
- Reject invalid combination (no payment + recoveries)

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/eps/generator.test.ts
npm run lint src/lib/hmrc/eps/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/paye-schemes/{id}/generate-eps
- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 3: Submit EPS for Adjustments
