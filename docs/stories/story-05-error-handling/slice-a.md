# Slice a: Error Parser and Classification Engine

**Story:** story-05-error-handling
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** None

---

## Goal

Parse HMRC rejection responses and classify errors as correctable or permanent. Build an error code catalog with HMRC error mappings and severity classifications.

---

## Decision Checklist

- [x] All libraries/packages named: fast-xml-parser 4.3.x
- [x] SDK methods identified: XMLParser.parse(), error code lookup
- [x] External service endpoints: N/A (error processing)
- [x] Data contracts defined: HMRCErrorCode, ErrorClassification interfaces
- [x] Configuration: N/A
- [x] Error scenarios: Unknown error codes, ambiguous classifications
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Handle FPS Rejection
- HMRC RTI Error Codes Reference

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/errors/parser.ts` | create | HMRC error response parser |
| `src/lib/hmrc/errors/catalog.ts` | create | Error code catalog |
| `src/lib/hmrc/errors/classifier.ts` | create | Error classification engine |
| `src/lib/hmrc/errors/types.ts` | create | Error type definitions |

---

## Responsibilities

1. Parse HMRC rejection XML error blocks
2. Lookup error codes in catalog
3. Classify errors as correctable or permanent
4. Map errors to severity levels
5. Handle unknown error codes gracefully

---

## Contracts

### ErrorParser.parseRejection()
- **Method:** `parseRejection(xml: string): ParsedRejection`
- **Input:** `xml: string` — HMRC rejection response XML
- **Output:** ParsedRejection
  ```typescript
  interface ParsedRejection {
    hmrcCorrelationId: string;
    processedAt: Date;
    errors: ClassifiedError[];
    isCorrectable: boolean; // True if any error is correctable
  }
  
  interface ClassifiedError {
    code: string;
    message: string;
    severity: 'fatal' | 'error' | 'warning';
    classification: 'correctable' | 'permanent' | 'unknown';
    category: 'employee_data' | 'employer_data' | 'technical' | 'schema';
    xpath?: string;
    employeeNino?: string;
    guidance: string; // User-facing guidance
    canAutoCorrect: boolean;
  }
  ```

### Error Code Catalog
```typescript
const HMRC_ERROR_CATALOG: Record<string, HMRCErrorDefinition> = {
  '3001': {
    code: '3001',
    message: 'National Insurance number not recognized',
    severity: 'fatal',
    classification: 'correctable',
    category: 'employee_data',
    guidance: 'Check the employee\'s National Insurance number. If they don\'t have one, use the temporary number scheme.',
    canAutoCorrect: false
  },
  '3002': {
    code: '3002',
    message: 'Invalid tax code',
    severity: 'error',
    classification: 'correctable',
    category: 'employee_data',
    guidance: 'Verify the employee\'s tax code with HMRC or use the emergency tax code.',
    canAutoCorrect: false
  },
  '5001': {
    code: '5001',
    message: 'Duplicate submission',
    severity: 'fatal',
    classification: 'permanent',
    category: 'technical',
    guidance: 'This submission has already been received. Check the correlation ID.',
    canAutoCorrect: false
  },
  '6001': {
    code: '6001',
    message: 'Invalid Employer PAYE Reference',
    severity: 'fatal',
    classification: 'correctable',
    category: 'employer_data',
    guidance: 'Check your Employer PAYE Reference in Settings > PAYE Scheme.',
    canAutoCorrect: false
  }
};
```

### ErrorClassifier.classify()
- **Method:** `classify(code: string): ErrorClassification`
- **Logic:**
  1. Lookup code in catalog
  2. If found, return catalog classification
  3. If not found, default to 'unknown' classification
  4. Unknown errors are treated as correctable (conservative)

---

## Business Rules & Invariants

1. All fatal errors are considered blocking
2. Unknown error codes default to 'correctable' classification
3. Correctable errors allow resubmission after fix
4. Permanent errors indicate submission should not be retried
5. Error catalog must be updateable without code deployment

---

## Edge Cases

1. **Unknown error code** — Log, classify as 'unknown', allow manual handling
2. **Multiple errors with conflicting classifications** — Most severe wins
3. **Empty error list** — Treat as processing error, retry
4. **Error code without catalog entry** — Use raw message, generic guidance

---

## Tests

### error-parser.test.ts
- Parse rejection with known error codes
- Parse rejection with unknown error code
- Classify mix of correctable and permanent errors
- Handle empty error list
- Determine isCorrectable based on error classifications

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/errors/parser.test.ts
npm run lint src/lib/hmrc/errors/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Handle FPS Rejection
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Error categorization
