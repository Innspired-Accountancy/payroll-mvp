# Slice b: Correction FPS Generation

**Story:** story-08-corrections
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Generate correction FPS with incremented version numbers and appropriate late reporting reason codes. Copy all employees from original with corrections applied, preserving unchanged data.

---

## Decision Checklist

- [x] All libraries/packages named: Handlebars 4.7.x, fast-xml-parser 4.3.x
- [x] SDK methods identified: FpsGenerator.generate(), CorrectionEligibilityChecker
- [x] External service endpoints: N/A (internal generation)
- [x] Data contracts defined: CorrectionFPSOptions, LateReasonCode interfaces
- [x] Configuration: N/A
- [x] Error scenarios: Ineligible for correction, invalid late reason, data mismatch
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Manager corrects data and resubmits
- HMRC RTI Late Reporting Reason Codes

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/corrections/generator.ts` | create | Correction FPS generator |
| `src/lib/hmrc/corrections/late-reasons.ts` | create | Late reporting reason logic |
| `src/lib/hmrc/templates/fps-correction-template.hbs` | create | Correction FPS template |

---

## Responsibilities

1. Generate correction FPS with incremented version
2. Include late reporting reason code (H-M)
3. Copy all employees from original submission
4. Apply corrections to specific employees/fields
5. Link correction to parent submission

---

## Contracts

### CorrectionFpsGenerator.generate()
- **Method:** `async generate(options: CorrectionFPSOptions): Promise<CorrectionFPSResult>`
- **Input:**
  ```typescript
  interface CorrectionFPSOptions {
    parentSubmissionId: string;
    correctionType: 'current_year' | 'prior_year';
    // Corrections to apply
    corrections: EmployeeCorrection[];
    // Late reason (auto-determined if not provided)
    lateReason?: LateReasonCode;
    // Override flags
    forceLateReason?: boolean;
  }
  
  interface EmployeeCorrection {
    employeeId: string;
    nino?: string; // If NINO changed
    fields: {
      field: string; // XML field path
      oldValue: unknown;
      newValue: unknown;
    }[];
  }
  
  type LateReasonCode = 'H' | 'I' | 'J' | 'K' | 'L' | 'M';
  ```
- **Output:** CorrectionFPSResult
  ```typescript
  interface CorrectionFPSResult {
    submissionId: string;
    parentSubmissionId: string;
    correlationId: string;
    version: number;
    type: 'fps';
    correctionType: 'current_year' | 'prior_year';
    lateReason: LateReasonCode;
    lateReasonDescription: string;
    xml: string;
    employeeCount: number;
    correctionsApplied: number;
    generatedAt: Date;
  }
  ```

### Late Reporting Reason Codes (HMRC)
| Code | Description | Use When |
|------|-------------|----------|
| H | Correction to earlier submission | Correcting errors in earlier FPS |
| I | Casual employee | Employee not paid regularly |
| J | Correction to EPS | Not applicable for FPS |
| K | No payments in period | Not applicable for corrections |
| L | Reasonable excuse | Late due to exceptional circumstances |
| M | Other | Other reason not covered above |

### Late Reason Determination
```typescript
function determineLateReason(
  correctionType: 'current_year' | 'prior_year',
  originalSubmissionDate: Date,
  correctionDate: Date,
  hasReasonableExcuse: boolean
): LateReasonCode {
  if (correctionType === 'prior_year') {
    // Prior year corrections always use 'H' (correction to earlier)
    return 'H';
  }
  
  if (hasReasonableExcuse) {
    return 'L';
  }
  
  // Current year correction after 19th
  return 'H';
}
```

### Correction FPS XML Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<EmployerPaymentSubmission xmlns="http://www.govtalk.gov.uk/taxation/PAYE/RTI/EmployerPaymentSubmission/25-26">
  <MessageHeader>
    <MessageId>{uuid}</MessageId>
    <CreationTimestamp>{ISO8601}</CreationTimestamp>
  </MessageHeader>
  <Employer>
    <EmployerPayeReference>{ref}</EmployerPayeReference>
    <!-- ... -->
  </Employer>
  <FullPaymentSubmission>
    <SubmissionId>{new-correlation-id}</SubmissionId>
    <HMRCsiteIdentifier>0</HMRCsiteIdentifier>
    <DateFormReceived>{YYYY-MM-DD}</DateFormReceived>
    <TaxYear>{26-27}</TaxYear>
    <TaxMonth>{1-12}</TaxMonth>
    <LateReportingReason>{H|M}</LateReportingReason>
    <!-- All employees from original, with corrections applied -->
    <Employee>
      <!-- ... corrected data ... -->
    </Employee>
    <TotalPayments>
      <!-- ... corrected totals ... -->
    </TotalPayments>
  </FullPaymentSubmission>
</EmployerPaymentSubmission>
```

---

## Business Rules & Invariants

1. Correction version number = parent version + 1
2. All employees from original must be included (even if unchanged)
3. Late reporting reason required for all corrections
4. Prior year corrections use code 'H' (correction to earlier)
5. Correction FPS includes all data, not just changes

---

## Edge Cases

1. **NINO correction** — Update NINO, include both old and new in chain
2. **Employee no longer with company** — Still include in correction
3. **New employee discovered** — Add to correction with late reason
4. **Original data no longer available** — Reconstruct from stored XML
5. **Multiple corrections** — Each is new version in chain

---

## Tests

### correction-generator.test.ts
- Generate current year correction with late reason H
- Generate prior year correction with late reason H
- Apply corrections to specific employee fields
- Include all employees from original
- Increment version number correctly

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/corrections/generator.test.ts
npm run lint src/lib/hmrc/corrections/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Manager corrects data and resubmits
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Maintain submission chain
