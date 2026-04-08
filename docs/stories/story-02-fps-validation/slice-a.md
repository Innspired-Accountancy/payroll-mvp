# Slice a: XSD Schema Validation Engine

**Story:** story-02-fps-validation
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** None

---

## Goal

Implement XSD schema validation for FPS XML using libxml2 bindings. Validate against HMRC RTI EmployerPaymentSubmission schema v25-26, reporting structural errors with line numbers and XPath.

---

## Decision Checklist

- [x] All libraries/packages named: libxmljs2 0.33.x (Node.js libxml2 bindings)
- [x] SDK methods identified: xml.Schema.validate(), xml.Document.fromXml()
- [x] External service endpoints: N/A (local validation)
- [x] Data contracts defined: XSDValidationResult, XSDError interfaces
- [x] Configuration: HMRC_XSD_PATH environment variable (default: ./schemas/rti/)
- [x] Error scenarios: Schema parse errors, validation failures, malformed XML
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Compliance
- HMRC RTI Technical Specification v25-26

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/validation/xsd-validator.ts` | create | XSD validation engine |
| `src/lib/hmrc/validation/schemas/` | create | Directory for HMRC XSD files |
| `src/lib/hmrc/validation/schemas/EmployerPaymentSubmission-v25-26.xsd` | create | HMRC schema file |
| `src/lib/hmrc/types/validation.ts` | create | Validation result types |

---

## Responsibilities

1. Load and parse HMRC XSD schema files
2. Validate FPS XML against schema
3. Extract validation errors with line numbers and XPath
4. Cache parsed schemas for performance
5. Support schema versioning per tax year

---

## Contracts

### XsdValidator.validate()
- **Method:** `validate(xml: string, schemaVersion: string): Promise<XSDValidationResult>`
- **Input:** 
  - `xml: string` — FPS XML document
  - `schemaVersion: string` — Tax year schema (e.g., "25-26")
- **Output:** XSDValidationResult
  ```typescript
  interface XSDValidationResult {
    valid: boolean;
    errors: XSDError[];
    warnings: XSDWarning[];
    schemaVersion: string;
    validatedAt: Date;
  }
  
  interface XSDError {
    type: 'structural' | 'data-type' | 'constraint';
    message: string;
    line: number;
    column: number;
    xpath: string;
    code: string;
  }
  ```
- **Errors:**
  - `SchemaNotFoundError` — XSD file not found for version
  - `SchemaParseError` — XSD file malformed
  - `XmlParseError` — Input XML not well-formed
- **Auth:** N/A (internal service)

### Schema File Locations
```
src/lib/hmrc/validation/schemas/
├── EmployerPaymentSubmission-v24-25.xsd
├── EmployerPaymentSubmission-v25-26.xsd  # Current
├── common/
│   ├── types.xsd
│   └── address.xsd
```

---

## Business Rules & Invariants

1. Schema files must match HMRC published schemas exactly
2. Validation errors must include XPath for field mapping
3. Schema parsing errors are fatal (stop processing)
4. Parsed schemas are cached per version for performance
5. XSD validation is blocking (prevents submission if errors exist)

---

## Edge Cases

1. **Missing schema file** — Throw SchemaNotFoundError with available versions
2. **Malformed XML** — Return XmlParseError before XSD validation
3. **Encoding issues** — Force UTF-8 encoding on all inputs
4. **Very large XML (10MB+)** — Stream validation for memory efficiency
5. **Schema version mismatch** — Validate against declared namespace

---

## Tests

### xsd-validator.test.ts
- Validate correct FPS XML (passes)
- Validate XML with missing required element (fails)
- Validate XML with invalid data type (fails)
- Validate malformed XML (throws XmlParseError)
- Validate with non-existent schema (throws SchemaNotFoundError)
- Cache schema between validations

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/validation/xsd-validator.test.ts
npm run lint src/lib/hmrc/validation/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: System validates against HMRC schema
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Schema version tracking
