# Slice b: Conflict Detection and Submission Workflow

**Story:** story-23-leave-request
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** slice-a (leave request form)

---

## Goal

Implement conflict detection for overlapping leave requests and the submission workflow. This ensures employees cannot double-book leave and handles the request creation with proper state management.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: leave.checkConflicts(), leave.submitRequest()
- [x] External service endpoints: None (internal database)
- [x] Data contracts defined: ConflictCheckResult, LeaveSubmissionResult
- [x] Configuration variables: CONFLICT_CHECK_BUFFER_DAYS=0
- [x] Error scenarios identified: Date overlap, insufficient balance, system error
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 3: Request Leave (conflict check)
- 02-01-core-payroll-spec.md — Leave request validation

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/leave.ts` | update | Add submit and conflict procedures |
| `src/lib/leave/conflicts.ts` | create | Conflict detection logic |
| `src/components/leave/conflict-warning.tsx` | create | Overlap warning display |
| `src/components/leave/submit-confirmation.tsx` | create | Submit confirmation modal |

---

## Responsibilities

1. Check for date overlaps with existing approved/pending requests
2. Validate sufficient balance before submission
3. Create leave request record with pending status
4. Return request ID and status on success
5. Show balance after request in confirmation
6. Handle errors gracefully with user feedback

---

## Contracts

### leave.checkConflicts
- **Method:** tRPC query `leave.checkConflicts`
- **Input:**
  ```typescript
  {
    startDate: string;        // ISO date
    endDate: string;          // ISO date
    excludeRequestId?: string;  // For editing (MVP: not used)
  }
  ```
- **Output:**
  ```typescript
  {
    hasConflict: boolean;
    conflicts: [
      {
        requestId: string;
        leaveType: string;
        startDate: string;
        endDate: string;
        status: "approved" | "pending";
      }
    ];
  }
  ```
- **Auth:** Protected procedure

### leave.submitRequest
- **Method:** tRPC mutation `leave.submitRequest`
- **Input:**
  ```typescript
  {
    leaveType: string;
    startDate: string;
    endDate: string;
    days: number;
    reason?: string;
  }
  ```
- **Output:**
  ```typescript
  {
    success: boolean;
    request: {
      id: string;
      status: "pending";
      submittedAt: string;
      balanceAfter: number;
    };
  }
  ```
- **Errors:**
  - `CONFLICT` — Date overlap with existing request
  - `BAD_REQUEST` — Insufficient balance
  - `VALIDATION_ERROR` — Invalid dates or parameters
- **Auth:** Protected procedure

---

## Business Rules & Invariants

1. Conflict = any date overlap with approved or pending request
2. Submission creates request with pending status
3. Balance deducted immediately for approved requests only
4. Booked (pending) balance shown separately
5. Cannot submit if conflicts exist
6. System validates balance server-side (not client-only)

---

## Edge Cases

1. **Adjacent dates (no gap)** — Not a conflict (e.g., ends Monday, starts Tuesday)
2. **Same day range** — Conflict if any overlap
3. **Request while pending exists** — Block with conflict warning
4. **Concurrent submissions** — Database constraint prevents duplicates
5. **Balance race condition** — Server-side validation prevents overdraw

---

## Tests

### leave.router.test.ts
- checkConflicts finds overlapping requests
- submitRequest creates pending request
- Rejects submission with conflict
- Rejects submission with insufficient balance

### conflicts.test.ts
- Detects exact overlap
- Detects partial overlap
- Allows adjacent dates
- Handles multiple conflicts

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/leave.test.ts
npx vitest run src/lib/leave/conflicts.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Leave Request → Conflict checking
- 02-05-employee-portal-spec.md § Journey 3: Request Leave → System checks for conflicts
