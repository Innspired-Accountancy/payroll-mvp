# Slice b: Business Rules Validation Layer

**Story:** story-02-fps-validation
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement business rule validation beyond XSD schema checks: NI number validation, tax code format, date ranges, cross-field validations, and HMRC-specific business rules. Classify issues as blockers or warnings.

---

## Decision Checklist

- [x] All libraries/packages named: Zod 3.22.x for rule schemas
- [x] SDK methods identified: z.string().regex(), custom refinements
- [x] External service endpoints: N/A (internal validation)
- [x] Data contracts defined: BusinessRule, ValidationIssue, Severity types
- [x] Configuration: N/A
- [x] Error scenarios: Invalid NI, invalid tax code, date mismatches, calculation errors
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: Manager reviews validation results
- HMRC RTI Business Rules Specification

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/validation/business-rules.ts` | create | Business rules engine |
| `src/lib/hmrc/validation/rules/` | create | Individual rule implementations |
| `src/lib/hmrc/validation/rules/ni-number.ts` | create | NI number validation rule |
| `src/lib/hmrc/validation/rules/tax-code.ts` | create | Tax code validation rule |
| `src/lib/hmrc/validation/rules/dates.ts` | create | Date validation rules |

---

## Responsibilities

1. Validate NI numbers against HMRC format rules
2. Validate tax codes are current HMRC codes
3. Check date ranges (payment dates within tax period)
4. Validate monetary calculations (totals match sum of lines)
5. Cross-field validations (tax code vs age for K codes)
6. Classify issues as blockers (prevent submission) or warnings (allow)

---

## Contracts

### BusinessRulesValidator.validate()
- **Method:** `validate(fpsData: FPSData): Promise<BusinessValidationResult>`
- **Input:** `fpsData: FPSData` — Parsed FPS data structure
- **Output:** BusinessValidationResult
  ```typescript
  interface BusinessValidationResult {
    valid: boolean; // true if no blockers
    blockers: ValidationIssue[];
    warnings: ValidationIssue[];
    info: ValidationIssue[];
    validatedAt: Date;
  }
  
  interface ValidationIssue {
    code: string; // e.g., "BR001"
    severity: 'blocker' | 'warning' | 'info';
    message: string;
    employeeId?: string; // Null for submission-level issues
    field?: string; // Field name or XPath
    value?: unknown; // Actual value
    expected?: unknown; // Expected value/pattern
    guidance: string; // User-facing correction guidance
  }
  ```
- **Errors:** N/A (returns validation result, never throws for validation failures)
- **Auth:** N/A (internal service)

### Business Rules Catalog

| Rule ID | Description | Severity | Fields |
|---------|-------------|----------|--------|
| BR001 | NI number format invalid | Blocker | Employee.NINO |
| BR002 | NI number check digit invalid | Warning | Employee.NINO |
| BR003 | Tax code not recognized | Blocker | Employment.TaxCode |
| BR004 | Tax code K-prefix for under-25 | Warning | Employment.TaxCode, Employee.BirthDate |
| BR005 | Payment date outside tax period | Blocker | Employment.PaymentDate |
| BR006 | Taxable pay negative | Blocker | Employment.TaxablePay |
| BR007 | Taxable pay exceeds 999999.99 | Blocker | Employment.TaxablePay |
| BR008 | Employee under 16 with taxable pay | Warning | Employee.BirthDate, Employment.TaxablePay |
| BR009 | Total tax doesn't match sum of lines | Blocker | TotalPayments.TaxDeducted |
| BR010 | Duplicate NINO in submission | Blocker | Employee.NINO |
| BR011 | Director with monthly NIC calculation | Warning | Employment.Director, NICs.CalculationMethod |

### NI Number Validation
```typescript
// Format: AB123456C
// First two chars: Not D, F, I, Q, U, V
// First char: Not second char (no doubles)
// Suffix: A, B, C, D only
const NI_NUMBER_REGEX = /^[A-CEGHJ-PR-TW-Z][A-CEGHJ-NPR-TW-Z]\d{6}[A-D]$/;
```

---

## Business Rules & Invariants

1. Blockers prevent submission; warnings allow with acknowledgment
2. NI numbers must pass both format and (where possible) check digit validation
3. Tax codes must match current HMRC valid codes list
4. Payment dates must fall within the declared tax period
5. Monetary totals must match sum of individual line items
6. Validation issues must include actionable guidance for correction

---

## Edge Cases

1. **Temporary NI numbers** — Allow (warn) for genuinely new employees
2. **Overseas addresses** — Validate differently (no UK postcode)
3. **Very high earners** — Special handling for 7-figure sums
4. **Leaver in period** — Payment date may differ from period dates
5. **Multiple warnings for same employee** — Group by employee in UI

---

## Tests

### business-rules.test.ts
- Valid FPS data passes all rules
- Invalid NI format returns BR001 blocker
- Unrecognized tax code returns BR003 blocker
- Payment date outside period returns BR005 blocker
- Calculation mismatch returns BR009 blocker
- Tax code K for young employee returns BR004 warning
- Duplicate NINO returns BR010 blocker

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/validation/business-rules.test.ts
npm run lint src/lib/hmrc/validation/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: Manager reviews validation results
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Error categorization
