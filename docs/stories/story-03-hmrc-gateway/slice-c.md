# Slice c: Submission Service with Idempotency

**Story:** story-03-hmrc-gateway
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement FPS/EPS submission service with idempotency protection, queuing, retry logic, and circuit breaker for HMRC gateway unavailability. Update submission status throughout the lifecycle.

---

## Decision Checklist

- [x] All libraries/packages named: BullMQ 5.x (Redis queue), opossum 8.x (circuit breaker)
- [x] SDK methods identified: Queue.add(), CircuitBreaker.fire()
- [x] External service endpoints: HMRC submission endpoint (POST /submit)
- [x] Data contracts defined: SubmissionJob, SubmissionResult interfaces
- [x] Configuration: REDIS_URL, CIRCUIT_BREAKER_THRESHOLD
- [x] Error scenarios: Duplicate submission, gateway timeout, retry exhaustion
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/hmrc-submissions/{id}/submit
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Reliability

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/submission/service.ts` | create | Submission orchestration service |
| `src/lib/hmrc/submission/queue.ts` | create | BullMQ queue configuration |
| `src/lib/hmrc/submission/circuit-breaker.ts` | create | Circuit breaker configuration |
| `src/server/routers/hmrc-submissions.ts` | update | Add submit procedure |

---

## Responsibilities

1. Accept submission requests via tRPC
2. Check idempotency (prevent duplicate submissions)
3. Queue submission for background processing
4. Handle HMRC gateway communication
5. Implement circuit breaker for gateway failures
6. Update submission status and store HMRC correlation ID
7. Retry on transient failures (max 3 attempts)

---

## Contracts

### SubmissionService.submit()
- **Method:** `async submit(submissionId: string): Promise<SubmissionQueuedResult>`
- **Idempotency Key:** `submissionId` + `attemptNumber`
- **Output:**
  ```typescript
  interface SubmissionQueuedResult {
    submissionId: string;
    status: 'queued' | 'already_submitted';
    correlationId: string;
    queuePosition?: number;
    estimatedProcessingTime?: number; // seconds
  }
  ```
- **Errors:**
  - `SubmissionNotFoundError` — Submission doesn't exist
  - `InvalidStateError` — Submission not in 'validated' status
  - `DuplicateSubmissionError` — Already submitted, returned HMMC correlation ID
  - `CircuitOpenError` — Gateway unavailable, queued for retry

### Queue Job Processing
- **Queue:** `hmrc-submissions` (BullMQ)
- **Job Data:**
  ```typescript
  interface SubmissionJob {
    submissionId: string;
    correlationId: string;
    attemptNumber: number;
    maxAttempts: number;
  }
  ```
- **Job Options:**
  ```typescript
  {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 30000 // 30s, 60s, 120s
    },
    removeOnComplete: 100,
    removeOnFail: 50
  }
  ```

### Circuit Breaker Configuration
```typescript
{
  timeout: 30000, // 30s
  errorThresholdPercentage: 50,
  resetTimeout: 60000, // 1 minute
  volumeThreshold: 5 // Min requests before checking threshold
}
```

### HMRC Submission Endpoint
```
POST /submit
Content-Type: text/xml
Gov-Client-Connection-Method: WEB_APP_VIA_SERVER
Gov-Client-Device-ID: {deviceId}
Gov-Client-User-IDs: {hashedUserId}
Gov-Vendor-Product-Name: UKBureauPayroll
Gov-Vendor-Version: {version}

{FPS XML payload}
```

### HMRC Response
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Acknowledgement xmlns="http://www.govtalk.gov.uk/taxation/PAYE/RTI/Acknowledgement/25-26">
  <CorrelationId>{platform-correlation-id}</CorrelationId>
  <HMRCcorrelationId>{hmrc-correlation-id}</HMRCcorrelationId>
  <Status>accepted</Status>
  <Timestamp>2026-04-28T14:31:15Z</Timestamp>
</Acknowledgement>
```

---

## Business Rules & Invariants

1. Submissions can only be made from 'validated' status
2. Correlation ID is generated once and never changes
3. Idempotency checked via correlation ID in database
4. Circuit breaker opens after 50% failure rate
5. Max 3 retry attempts with exponential backoff
6. All submission attempts logged with timestamp

---

## Edge Cases

1. **Duplicate submission detection** — Query by correlation_id before submit
2. **Gateway timeout during submission** — Mark as uncertain, poll for status
3. **Circuit breaker open** — Queue job, retry when closed
4. **Partial success (accepted by HMRC, network error)** — Poll for confirmation
5. **Redis queue unavailable** — Store in database, process on recovery

---

## Tests

### submission.service.test.ts
- Queue valid submission successfully
- Reject submission from invalid state
- Detect and handle duplicate submission
- Circuit breaker opens after failures
- Retry on transient failure
- Store HMRC correlation ID on success

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/submission/service.test.ts
npm run lint src/lib/hmrc/submission/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/hmrc-submissions/{id}/submit
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Idempotent submissions
