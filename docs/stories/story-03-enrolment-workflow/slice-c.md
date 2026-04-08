# Slice c: Communication Trigger Integration

**Story:** story-03-enrolment-workflow
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Integrate enrolment workflow with the communication service to trigger statutory enrolment letters within the 6-week compliance window.

---

## Decision Checklist

- [x] Trigger timing: Immediately after successful enrolment
- [x] Queue system: BullMQ for async processing
- [x] SLA monitoring: 6-week (42-day) deadline tracking
- [x] Retry logic: 3 attempts with exponential backoff
- [x] Failure handling: Alert to manual queue for intervention
- [x] Content template: Enrolment letter template reference
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Journey 2, Communications

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/communicationTrigger.ts` | create | Enrolment communication trigger |
| `src/lib/jobs/enrolmentCommunicationQueue.ts` | create | BullMQ queue definition |
| `src/lib/pension/slaMonitor.ts` | create | 6-week SLA tracking |
| `src/server/routers/pensionCommunications.ts` | create | Communication status API |
| `src/tests/pension/communicationTrigger.test.ts` | create | Integration tests |

---

## Responsibilities

1. Queue communication job on successful enrolment
2. Track SLA deadline (42 days from enrolment)
3. Monitor communication delivery status
4. Alert on approaching SLA breach (35 days)
5. Retry failed communications automatically
6. Escalate persistent failures to manual queue

---

## Contracts

### queueEnrolmentCommunication()
- **Method:** `queueEnrolmentCommunication(enrolmentId: UUID): Promise<JobId>`
- **Action:** Add job to enrolmentCommunicationQueue
- **Data:** { enrolmentId, employeeId, schemeId, enrolmentDate, deadline }
- **Delay:** Immediate (0ms)

### EnrolmentCommunicationJob
- **Processor:** `processEnrolmentCommunication(job)`
- **Steps:**
  1. Fetch enrolment and employee details
  2. Generate enrolment letter content
  3. Send via email/post based on preferences
  4. Record dispatch in communication log
  5. Update enrolment.communication_sent = true
- **Retry:** 3 attempts with 1hr, 4hr, 12hr delays
- **Failure:** Move to manual-intervention queue after retries

### SLAMonitor
- **Method:** `checkEnrolmentCommunications()`
- **Schedule:** Daily at 09:00
- **Alert thresholds:**
  - 35 days: Warning (7 days to SLA)
  - 42 days: Critical (SLA breach imminent)
- **Actions:** Log alerts, send notifications to compliance team

---

## Business Rules & Invariants

1. Communication must be queued within 24 hours of enrolment
2. 6-week (42-day) SLA from enrolment date to dispatch
3. Failed communications retried 3 times before manual intervention
4. Approaching SLA breach (35 days) triggers warning alerts
5. SLA breach (42 days) triggers critical escalation
6. All communication attempts logged with timestamps

---

## Edge Cases

1. **Communication service down** — Queue holds jobs, retry when available
2. **Invalid email address** — Mark for postal dispatch
3. **Employee contact preferences missing** — Default to postal
4. **Duplicate communication queued** — Idempotent processing
5. **Enrolment cancelled before communication** — Cancel queued job

---

## Tests

### communicationTrigger.test.ts
- Communication queued on enrolment
- SLA monitoring detects approaching deadline
- Retry logic for failed communications
- Escalation after max retries
- Idempotent duplicate handling

---

## Verification

```bash
npm run test:unit src/tests/pension/communicationTrigger.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-03-enrolment-workflow/slice-a.md → Enrolment success trigger
- story-07-communications → Full communication service
