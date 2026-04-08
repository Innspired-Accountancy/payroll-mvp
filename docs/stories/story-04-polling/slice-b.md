# Slice b: Polling Service with Scheduler

**Story:** story-04-polling
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement scheduled polling service for pending submissions. Poll HMRC gateway every 30 seconds for acknowledgements, respect rate limits, and stop polling after 24 hours or final status reached.

---

## Decision Checklist

- [x] All libraries/packages named: node-cron 3.0.x, BullMQ 5.x for job scheduling
- [x] SDK methods identified: cron.schedule(), Queue.add(), QueueScheduler
- [x] External service endpoints: HMRC status endpoint (GET /status)
- [x] Data contracts defined: PollJob, PollResult, PollingConfig interfaces
- [x] Configuration: POLL_INTERVAL_MS=30000, MAX_POLL_DURATION_MS=86400000
- [x] Error scenarios: Rate limiting, gateway timeout, max polling duration exceeded
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Status polling interval
- 02-02-hmrc-submissions-spec.md § API Contracts → GET /api/v1/hmrc-submissions/{id}/status

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/polling/service.ts` | create | Polling orchestration service |
| `src/lib/hmrc/polling/scheduler.ts` | create | Cron-based job scheduler |
| `src/lib/hmrc/polling/queue.ts` | create | Polling job queue |

---

## Responsibilities

1. Schedule polling jobs for submitted submissions
2. Poll HMRC gateway for acknowledgement status
3. Handle rate limiting with exponential backoff
4. Stop polling after 24 hours (timeout condition)
5. Trigger state machine transitions on status change
6. Clean up completed polling jobs

---

## Contracts

### PollingService.startPolling()
- **Method:** `async startPolling(submissionId: string): Promise<void>`
- **Input:** `submissionId: string` — UUID of submitted submission
- **Behavior:** 
  - Schedule first poll immediately
  - Schedule subsequent polls every 30 seconds
  - Stop after 24 hours or final status reached
- **Errors:**
  - `SubmissionNotFoundError` — Submission doesn't exist
  - `InvalidStateError` — Submission not in 'submitted' or 'acknowledged' status

### PollJob Structure
```typescript
interface PollJob {
  submissionId: string;
  correlationId: string;
  attemptNumber: number;
  startedAt: Date;
  nextPollAt: Date;
}
```

### Polling Config
```typescript
const POLLING_CONFIG = {
  intervalMs: 30000, // 30 seconds
  maxDurationMs: 86400000, // 24 hours
  maxRetries: 3,
  backoffMultiplier: 2,
  rateLimitDelayMs: 60000 // 1 minute if rate limited
};
```

### HMRC Status Endpoint
```
GET /status?correlationId={platform-correlation-id}
Headers: Auth session headers

Response:
<?xml version="1.0" encoding="UTF-8"?>
<StatusResponse xmlns="http://www.govtalk.gov.uk/taxation/PAYE/RTI/Status/25-26">
  <CorrelationId>{platform-correlation-id}</CorrelationId>
  <HMRCcorrelationId>{hmrc-correlation-id}</HMRCcorrelationId>
  <Status>processing|accepted|rejected</Status>
  <ProcessedAt>2026-04-28T14:31:15Z</ProcessedAt>
  <Errors>
    <Error>...</Error>
  </Errors>
</StatusResponse>
```

### Polling Lifecycle
1. Submission queued → Start polling schedule
2. Poll HMRC /status endpoint
3. If 'processing' → Schedule next poll in 30s
4. If 'accepted' → Transition to accepted, stop polling
5. If 'rejected' → Transition to rejected, stop polling
6. If 24h elapsed → Transition to timeout, stop polling
7. If rate limited → Backoff 60s, retry

---

## Business Rules & Invariants

1. Only poll submissions in 'submitted' or 'acknowledged' status
2. Maximum polling duration: 24 hours from submission
3. Polling interval: 30 seconds (respect HMRC rate limits)
4. Stop polling immediately on final status (accepted/rejected)
5. Poll jobs are idempotent (safe to retry)
6. Cleanup completed poll jobs after 7 days

---

## Edge Cases

1. **HMRC rate limiting** — Backoff to 60s, then 120s
2. **Gateway timeout** — Retry with exponential backoff
3. **Submission deleted while polling** — Stop polling, log warning
4. **System restart during polling** — Recover from database, resume
5. **Status endpoint returns unknown status** — Log error, retry

---

## Tests

### polling.service.test.ts
- Start polling for submitted submission
- Poll transitions to accepted status
- Poll transitions to rejected status
- Timeout after 24 hours
- Rate limit handling with backoff
- Stop polling on final status

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/polling/service.test.ts
npm run lint src/lib/hmrc/polling/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Status polling
- 02-02-hmrc-submissions-spec.md § API Contracts → Status endpoint
