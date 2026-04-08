# Slice c: Email Notifications and Request Cancellation

**Story:** story-23-leave-request
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** slice-b (conflict detection and submission)

---

## Goal

Implement email notifications for leave request submissions and approvals, plus employee cancellation functionality for pending requests. This closes the loop on the leave request workflow.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, nodemailer 6.9.x (or existing notification service)
- [x] SDK methods/API calls identified: notifications.send(), leave.cancelRequest()
- [x] External service endpoints: Internal notification service
- [x] Data contracts defined: NotificationPayload, CancelRequestInput
- [x] Configuration variables: NOTIFICATION_FROM_EMAIL="notifications@payroll.local"
- [x] Error scenarios identified: Notification failure, already processed, unauthorized cancel
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 3: Request Leave (notifications)
- 02-10-audit-compliance-spec.md — Notification audit logging

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/leave.ts` | update | Update cancel logic |
| `src/lib/notifications/leave.ts` | create | Leave notification templates |
| `src/components/leave/cancel-button.tsx` | create | Cancel action component |
| `src/lib/jobs/leave-notifications.ts` | create | Background notification jobs |

---

## Responsibilities

1. Send confirmation email to employee on submission
2. Send notification to approver on submission
3. Send status update to employee on approval/rejection
4. Allow employee to cancel pending requests
5. Log all notifications for audit
6. Handle notification failures gracefully

---

## Contracts

### notifications.sendLeaveRequestSubmitted
- **Method:** Internal service call
- **Input:**
  ```typescript
  {
    to: string;               // Employee email
    requestId: string;
    leaveType: string;
    startDate: string;
    endDate: string;
    days: number;
  }
  ```
- **Output:**
  ```typescript
  {
    sent: boolean;
    messageId: string | null;
  }
  ```

### notifications.sendLeaveStatusChanged
- **Method:** Internal service call
- **Input:**
  ```typescript
  {
    to: string;               // Employee email
    requestId: string;
    status: "approved" | "rejected";
    approverNotes: string | null;
  }
  ```
- **Output:**
  ```typescript
  {
    sent: boolean;
    messageId: string | null;
  }
  ```

### leave.cancelRequest (updated)
- **Method:** tRPC mutation `leave.cancelRequest`
- **Input:**
  ```typescript
  {
    id: string;
    reason?: string;          // Optional cancellation reason
  }
  ```
- **Output:**
  ```typescript
  {
    success: boolean;
    request: {
      id: string;
      status: "cancelled";
      cancelledAt: string;
    };
  }
  ```
- **Errors:**
  - `CONFLICT` — Request already approved/rejected
  - `FORBIDDEN` — Not request owner

---

## Business Rules & Invariants

1. Notification sent within 60 seconds of status change
2. Employee can only cancel own pending requests
3. Cancellation reason optional but logged if provided
4. Approver notified of cancellation
5. Balance updated on cancellation (booked days released)

---

## Edge Cases

1. **Notification service down** — Queue for retry, log failure
2. **Approver has no email** — Log warning, in-app notification only
3. **Cancel after approver viewed** — Still allowed if pending
4. **Bulk cancel** — Not supported in MVP
5. **Cancellation after leave date passed** — Block, contact admin

---

## Tests

### leave-notifications.test.ts
- Submission sends employee confirmation
- Submission sends approver notification
- Status change sends update
- Logs all notifications

### cancel-button.test.tsx
- Cancels pending request
- Shows confirmation dialog
- Error on already processed

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/lib/notifications/leave.test.ts
npx vitest run src/components/leave/cancel-button.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Leave Request → Notifications and cancellation
- 02-05-employee-portal-spec.md § Journey 3: Request Leave → System notifies employer and employee
