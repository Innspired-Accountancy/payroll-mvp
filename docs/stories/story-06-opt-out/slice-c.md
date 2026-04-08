# Slice c: NEST Opt-Out Notification

**Story:** story-06-opt-out
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Notify NEST of employee opt-outs via API or file submission.

---

## Decision Checklist

- [x] API method: POST /opt-outs (NEST API)
- [x] File method: Opt-out CSV file upload
- [x] Required data: Employee reference, NI number, opt-out date
- [x] Timing: Within 1 month of opt-out
- [x] Confirmation: NEST reference for opt-out record
- [x] Retry logic: Queue for retry on failure
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- NEST Web Services Developer Guide — Opt-out API
- 02-03-pension-auto-enrolment-spec.md — Provider submissions

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/nest/optOutNotification.ts` | create | NEST opt-out submission |
| `src/lib/nest/mappers/optOutMapper.ts` | create | Map to NEST format |
| `src/tests/nest/optOutNotification.test.ts` | create | Notification tests |

---

## Responsibilities

1. Map internal opt-out to NEST format
2. Submit opt-out to NEST API
3. Handle file-based fallback
4. Record NEST reference
5. Retry on failure

---

## Contracts

### notifyNestOptOut()
- **Method:** `notifyNestOptOut(optOutId: UUID): Promise<NestResult>`
- **Input:** Internal opt-out record
- **Output:** NestResult with reference or error
- **API:** POST /opt-outs with employee details

### NEST OptOutRequest
| Field | Type | Description |
|-------|------|-------------|
| organisationId | string | NEST org ID |
| employeeReference | string | Employer employee ref |
| niNumber | string | NI number |
| optOutDate | string | YYYY-MM-DD |
| optOutMethod | string | How employee opted out |

### NestResult
| Field | Type | Description |
|-------|------|-------------|
| success | boolean | Whether notification succeeded |
| reference | string | NEST opt-out reference |
| error | string | Error message if failed |

---

## Business Rules & Invariants

1. NEST notified within 1 month of opt-out
2. Employee reference must match existing NEST enrolment
3. Opt-out date cannot be in future
4. Failed notifications retried up to 3 times
5. Persistent failures escalate to manual processing

---

## Edge Cases

1. **Employee never enrolled in NEST** — No notification needed
2. **NEST API unavailable** — Queue for file-based submission
3. **Opt-out date before NEST enrolment** — Validation error

---

## Tests

### optOutNotification.test.ts
- Successful NEST notification
- API failure with retry
- File fallback on persistent failure

---

## Verification

```bash
npm run test:unit src/tests/nest/optOutNotification.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-05-nest-integration/slice-a.md → NEST API client
- story-06-opt-out/slice-a.md → Opt-out records
