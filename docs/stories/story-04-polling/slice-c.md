# Slice c: Acknowledgement Response Parser

**Story:** story-04-polling
**Epic:** epic-02-hmrc-submissions
**Effort:** S
**Dependencies:** slice-b

---

## Goal

Parse HMRC acknowledgement XML responses, extract status and error details, and trigger appropriate state machine transitions. Handle both success and rejection responses with full error extraction.

---

## Decision Checklist

- [x] All libraries/packages named: fast-xml-parser 4.3.x
- [x] SDK methods identified: XMLParser.parse(), validate()
- [x] External service endpoints: N/A (response parsing)
- [x] Data contracts defined: AcknowledgementResponse, ParsedError interfaces
- [x] Configuration: N/A
- [x] Error scenarios: Malformed XML, unexpected response format, missing fields
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Handle FPS Rejection
- HMRC RTI Acknowledgement Schema v25-26

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/polling/parser.ts` | create | Acknowledgement parser |
| `src/lib/hmrc/polling/types.ts` | update | Response type definitions |

---

## Responsibilities

1. Parse HMRC acknowledgement XML
2. Extract HMRC correlation ID and timestamp
3. Parse status (accepted/rejected/processing)
4. Extract error details from rejection responses
5. Map errors to submission record

---

## Contracts

### AcknowledgementParser.parse()
- **Method:** `parse(xml: string): AcknowledgementResponse`
- **Input:** `xml: string` — Raw HMRC acknowledgement XML
- **Output:** AcknowledgementResponse
  ```typescript
  interface AcknowledgementResponse {
    correlationId: string; // Platform correlation ID
    hmrcCorrelationId: string; // HMRC assigned ID
    status: 'processing' | 'accepted' | 'rejected';
    processedAt?: Date;
    errors?: ParsedError[];
    warnings?: ParsedWarning[];
    rawXml: string;
  }
  
  interface ParsedError {
    code: string; // HMRC error code
    message: string;
    severity: 'fatal' | 'error' | 'warning';
    xpath?: string; // Field location in XML
    employeeNino?: string; // If employee-specific
  }
  
  interface ParsedWarning {
    code: string;
    message: string;
  }
  ```
- **Errors:**
  - `XmlParseError` — Malformed XML
  - `ValidationError` — Required fields missing

### Accepted Response Example
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Acknowledgement xmlns="http://www.govtalk.gov.uk/taxation/PAYE/RTI/Acknowledgement/25-26">
  <CorrelationId>SUB-2026-000123</CorrelationId>
  <HMRCcorrelationId>HMRC-2026-A7F3B2</HMRCcorrelationId>
  <Status>accepted</Status>
  <ProcessedAt>2026-04-28T14:31:15Z</ProcessedAt>
</Acknowledgement>
```

### Rejected Response Example
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Acknowledgement xmlns="http://www.govtalk.gov.uk/taxation/PAYE/RTI/Acknowledgement/25-26">
  <CorrelationId>SUB-2026-000123</CorrelationId>
  <HMRCcorrelationId>HMRC-2026-A7F3B2</HMRCcorrelationId>
  <Status>rejected</Status>
  <ProcessedAt>2026-04-28T14:31:15Z</ProcessedAt>
  <Errors>
    <Error>
      <Code>3001</Code>
      <Message>National Insurance number not recognized</Message>
      <Severity>fatal</Severity>
      <Location>/EmployerPaymentSubmission/FullPaymentSubmission/Employee[3]/EmployeeDetails/Nino</Location>
      <EmployeeNino>AB999999C</EmployeeNINO>
    </Error>
  </Errors>
</Acknowledgement>
```

---

## Business Rules & Invariants

1. Correlation ID in response must match submission's correlation_id
2. HMRC correlation ID is stored for future reference
3. Processing status means continue polling
4. Accepted/rejected are final statuses (stop polling)
5. Errors include severity: fatal = blocking, error = correctable
6. Employee NINO in error response maps to employee_id

---

## Edge Cases

1. **Empty response body** — Treat as error, retry
2. **Missing correlation ID** — Reject, log error
3. **Unknown status value** — Log warning, retry
4. **Malformed error block** — Parse what we can, log remainder
5. **Encoding issues** — Force UTF-8, handle BOM

---

## Tests

### parser.test.ts
- Parse accepted acknowledgement
- Parse rejected acknowledgement with errors
- Parse processing acknowledgement
- Handle malformed XML (throw XmlParseError)
- Handle missing correlation ID (throw ValidationError)
- Handle unknown status (log warning, default to processing)

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/polling/parser.test.ts
npm run lint src/lib/hmrc/polling/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: Handle FPS Rejection
- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmissionError structure
