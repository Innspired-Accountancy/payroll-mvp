# Slice a: State Machine Implementation

**Story:** story-04-polling
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** None

---

## Goal

Implement a robust state machine for HMRC submission lifecycle using XState. Define valid state transitions, prevent invalid transitions, and provide state change hooks for notifications and side effects.

---

## Decision Checklist

- [x] All libraries/packages named: XState 5.x, @xstate/react 4.x
- [x] SDK methods identified: createMachine(), interpret(), actor.send()
- [x] External service endpoints: N/A (internal state management)
- [x] Data contracts defined: SubmissionState, StateTransition, SubmissionEvent types
- [x] Configuration: N/A
- [x] Error scenarios: Invalid state transition, concurrent state changes
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmission.status enum
- 02-02-hmrc-submissions-spec.md § User Journeys → Submission status flow

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/state-machine/submission.machine.ts` | create | XState machine definition |
| `src/lib/hmrc/state-machine/types.ts` | create | State machine types |
| `src/lib/hmrc/state-machine/guards.ts` | create | Transition guards |

---

## Responsibilities

1. Define all submission states and valid transitions
2. Prevent invalid state transitions
3. Trigger side effects on state entry/exit
4. Persist state changes to database
5. Support transition history for audit

---

## Contracts

### Submission State Machine
```typescript
const submissionMachine = createMachine({
  id: 'hmrcSubmission',
  initial: 'draft',
  states: {
    draft: {
      on: {
        VALIDATE: { target: 'validated', guard: 'isValidatable' },
        SUBMIT: { target: 'submitted', guard: 'isSubmittable' }
      }
    },
    validated: {
      on: {
        SUBMIT: { target: 'submitted', guard: 'isSubmittable' },
        INVALIDATE: { target: 'draft' }
      }
    },
    submitted: {
      on: {
        ACKNOWLEDGE: { target: 'acknowledged' },
        TIMEOUT: { target: 'submission_timeout' },
        REJECT: { target: 'rejected' }
      }
    },
    acknowledged: {
      on: {
        ACCEPT: { target: 'accepted' },
        REJECT: { target: 'rejected' }
      }
    },
    accepted: {
      type: 'final'
    },
    rejected: {
      on: {
        CORRECT: { target: 'draft', guard: 'isCorrectable' }
      }
    },
    submission_timeout: {
      on: {
        POLL_RETRY: { target: 'submitted' },
        ABANDON: { target: 'draft' }
      }
    },
    resubmitted: {
      on: {
        ACKNOWLEDGE: { target: 'acknowledged' }
      }
    }
  }
});
```

### State Transitions
| From | To | Event | Guard |
|------|-----|-------|-------|
| draft | validated | VALIDATE | hasNoBlockers |
| draft | submitted | SUBMIT | hasNoBlockers |
| validated | submitted | SUBMIT | always |
| validated | draft | INVALIDATE | always |
| submitted | acknowledged | ACKNOWLEDGE | hasAckResponse |
| submitted | rejected | REJECT | hasRejectionResponse |
| submitted | submission_timeout | TIMEOUT | >24h elapsed |
| acknowledged | accepted | ACCEPT | status == 'accepted' |
| acknowledged | rejected | REJECT | status == 'rejected' |
| rejected | draft | CORRECT | isCurrentTaxYear |

### SubmissionEvent Types
```typescript
type SubmissionEvent =
  | { type: 'VALIDATE'; validationResult: ValidationResult }
  | { type: 'SUBMIT'; correlationId: string; submittedAt: Date }
  | { type: 'ACKNOWLEDGE'; hmrcCorrelationId: string; acknowledgedAt: Date }
  | { type: 'ACCEPT'; acceptedAt: Date }
  | { type: 'REJECT'; errors: HMRCError[] }
  | { type: 'TIMEOUT' }
  | { type: 'CORRECT' }
  | { type: 'POLL_RETRY' }
  | { type: 'ABANDON' };
```

---

## Business Rules & Invariants

1. Only 'draft' or 'validated' submissions can be submitted
2. 'accepted' is a terminal state (no outgoing transitions)
3. Rejected submissions can only be corrected in current tax year
4. State transitions are atomic with database updates
5. Transition history recorded for audit

---

## Edge Cases

1. **Concurrent state change attempts** — Row-level locking, first wins
2. **State change during processing** — Queue event, process after current
3. **Invalid transition attempt** — Reject with current state info
4. **System restart mid-transition** — Recover from database state

---

## Tests

### submission.machine.test.ts
- Transition from draft to validated
- Block invalid transition (draft to accepted)
- Guard prevents submission with blockers
- Final state (accepted) has no outgoing transitions
- Rejected submission can be corrected
- Timeout transition after 24 hours

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/state-machine/submission.machine.test.ts
npm run lint src/lib/hmrc/state-machine/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmission.status enum values
- 02-02-hmrc-submissions-spec.md § User Journeys → Submission status tracking
