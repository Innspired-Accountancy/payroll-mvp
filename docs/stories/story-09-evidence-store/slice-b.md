# Slice b: Payload and Response Capture

**Story:** story-09-evidence-store
**Epic:** epic-02-hmrc-submissions
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Implement automatic capture of submission payloads and HMRC responses at key lifecycle points. Hook into submission service to capture without manual intervention.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, custom hooks/middleware
- [x] SDK methods identified: EvidenceService.store(), SubmissionService hooks
- [x] External service endpoints: N/A (internal capture)
- [x] Data contracts defined: CaptureHook, CaptureContext interfaces
- [x] Configuration: EVIDENCE_AUTO_CAPTURE=true
- [x] Error scenarios: Capture failure, storage full, encoding issues
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → All submissions logged
- 02-02-hmrc-submissions-spec.md § User Journeys → System stores submission payload

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/evidence/capture.ts` | create | Evidence capture service |
| `src/lib/hmrc/submission/service.ts` | update | Add capture hooks |
| `src/lib/hmrc/polling/service.ts` | update | Add response capture |

---

## Responsibilities

1. Capture submission payload at submit time
2. Capture HMRC response on acknowledgement
3. Capture error responses on rejection
4. Store with proper metadata and hashing
5. Handle capture failures gracefully (don't block submission)

---

## Contracts

### EvidenceCaptureService
```typescript
class EvidenceCaptureService {
  // Capture submission payload
  async captureSubmission(
    submissionId: string,
    payload: string,
    metadata: CaptureMetadata
  ): Promise<EvidenceRecord>;
  
  // Capture HMRC response
  async captureResponse(
    submissionId: string,
    response: string,
    responseType: 'acknowledgement' | 'error',
    metadata: ResponseMetadata
  ): Promise<EvidenceRecord>;
  
  // Capture validation results
  async captureValidation(
    submissionId: string,
    validationResult: ValidationResult
  ): Promise<EvidenceRecord>;
}

interface CaptureMetadata {
  timestamp: Date;
  correlationId: string;
  userId?: string;
  source: 'manual' | 'auto' | 'system';
}

interface ResponseMetadata extends CaptureMetadata {
  hmrcCorrelationId?: string;
  statusCode: number;
  responseTimeMs: number;
}
```

### Capture Hooks in Submission Service
```typescript
class SubmissionService {
  async submit(submissionId: string): Promise<SubmissionResult> {
    // 1. Get submission data
    const submission = await this.getSubmission(submissionId);
    
    // 2. Capture payload BEFORE submission
    await this.evidenceCapture.captureSubmission(
      submissionId,
      submission.payloadXml,
      { timestamp: new Date(), correlationId: submission.correlationId, source: 'manual' }
    );
    
    // 3. Submit to HMRC
    const response = await this.gatewayClient.submit(submission.payloadXml);
    
    // 4. Capture response
    await this.evidenceCapture.captureResponse(
      submissionId,
      response.body,
      response.statusCode === 200 ? 'acknowledgement' : 'error',
      {
        timestamp: new Date(),
        correlationId: submission.correlationId,
        hmrcCorrelationId: response.headers['x-hmrc-correlation-id'],
        statusCode: response.statusCode,
        responseTimeMs: response.durationMs
      }
    );
    
    // 5. Process response
    return this.processResponse(submissionId, response);
  }
}
```

### Capture Points
| Lifecycle Event | Evidence Type | Captured By |
|-----------------|---------------|-------------|
| FPS/EPS generated | submission_payload | Generate service |
| Validation completed | validation_result | Validation service |
| Submitted to HMRC | submission_payload (duplicate) | Submission service |
| Acknowledgement received | acknowledgement | Polling service |
| Rejection received | error_response | Polling service |
| Correction generated | submission_payload | Correction service |

---

## Business Rules & Invariants

1. Capture is automatic and synchronous with submission
2. Capture failures are logged but don't block submission
3. Payload captured before submission (ensures record even if send fails)
4. Response captured on receipt (even if parsing fails)
5. All captures include timestamp and correlation ID

---

## Edge Cases

1. **Capture fails during submission** — Log error, continue submission
2. **Response too large** — Truncate with indicator, store hash
3. **Encoding issues** — Force UTF-8, escape invalid characters
4. **Duplicate capture** — Hash-based deduplication prevents duplicates
5. **System crash during capture** — Transaction ensures consistency

---

## Tests

### evidence-capture.test.ts
- Capture submission payload
- Capture HMRC response
- Capture validation result
- Handle capture failure gracefully
- Verify capture doesn't block submission

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/evidence/capture.test.ts
npm run lint src/lib/hmrc/evidence/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 1: System stores submission payload
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → All submissions logged
