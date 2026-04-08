# Slice b: Approval Workflow and Status Management

**Story:** story-25-change-requests
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** slice-a (change request form)

---

## Goal

Implement the approval workflow for personal detail change requests. This includes status tracking, employer approval interface, and employee visibility into request status.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: changeRequests.getStatus(), changeRequests.cancel()
- [x] External service endpoints: None (internal workflow)
- [x] Data Contracts defined: ChangeRequestStatus, ApprovalAction
- [x] Configuration variables: APPROVAL_SLA_DAYS=5
- [x] Error scenarios identified: Already approved, already rejected, cancellation window expired
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 4: Update Bank Details (approval steps)
- 02-05-employee-portal-spec.md — Data Models: PersonalDetailChangeRequest
- 02-09-identity-access-spec.md — Approval workflow

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/change-requests.ts` | update | Add status and cancel |
| `src/app/(employee)/profile/requests/page.tsx` | create | Request status page |
| `src/components/change-requests/request-status.tsx` | create | Status display |
| `src/components/change-requests/cancel-button.tsx` | create | Cancel action |
| `src/lib/db/schema/change-requests.ts` | create | Database schema |

---

## Responsibilities

1. Store change request with pending_approval status
2. Display request status to employee
3. Allow cancellation of pending requests
4. Show approver notes on rejection
5. Update employee record on approval
6. Log all status changes
7. Enforce 5-day SLA target

---

## Contracts

### changeRequests.getMyRequests
- **Method:** tRPC query `changeRequests.getMyRequests`
- **Input:**
  ```typescript
  {
    status?: ("pending_approval" | "approved" | "rejected" | "cancelled")[];
  }
  ```
- **Output:**
  ```typescript
  {
    requests: [
      {
        id: string;
        changeType: "address" | "phone" | "bank_details";
        changeTypeLabel: string;
        oldValue: Record<string, unknown>;
        newValue: Record<string, unknown>;
        status: "pending_approval" | "approved" | "rejected" | "cancelled";
        submittedAt: string;
        reviewedAt: string | null;
        reviewedBy: string | null;
        approverNotes: string | null;
        canCancel: boolean;
        slaDeadline: string;      // 5 business days from submit
      }
    ];
  }
  ```
- **Auth:** Protected procedure

### changeRequests.cancel
- **Method:** tRPC mutation `changeRequests.cancel`
- **Input:**
  ```typescript
  {
    id: string;
    reason?: string;
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
  - `CONFLICT` — Already approved/rejected
  - `FORBIDDEN` — Not request owner
- **Auth:** Protected procedure

### Database Schema
```typescript
export const personalDetailChangeRequests = pgTable("personal_detail_change_requests", {
  id: uuid("id").primaryKey().defaultRandom(),
  employeeId: uuid("employee_id").references(() => employees.id).notNull(),
  changeType: varchar("change_type", { length: 20 }).notNull(), // address, phone, bank_details
  oldValue: jsonb("old_value").notNull(),
  newValue: jsonb("new_value").notNull(),
  status: varchar("status", { length: 20 }).notNull().default("pending_approval"),
  requestedAt: timestamp("requested_at").defaultNow().notNull(),
  approvedBy: uuid("approved_by").references(() => users.id),
  approvedAt: timestamp("approved_at"),
  approverNotes: text("approver_notes"),
  cancelledAt: timestamp("cancelled_at"),
  cancellationReason: text("cancellation_reason"),
});
```

---

## Business Rules & Invariants

1. Status flow: pending_approval → approved/rejected/cancelled
2. Only pending requests can be cancelled
3. Approved requests update employee record immediately
4. Rejected requests show approver notes to employee
5. SLA target: 5 business days from submission
6. All changes audited with before/after values

---

## Edge Cases

1. **Approver on leave** — Escalate after SLA breach
2. **Urgent change needed** — Employee can submit new request after cancel
3. **Partial approval** — Not supported (approve/reject entire request)
4. **Concurrent requests** — Block same-field requests, allow different fields
5. **Approval after employee leaves** — Reject with "Employee no longer active"

---

## Tests

### change-requests.router.test.ts
- getMyRequests returns employee's requests
- cancel updates status correctly
- Cannot cancel approved request
- Shows approver notes for rejected

### request-status.test.tsx
- Displays all requests
- Status badges color-coded
- Cancel button for pending
- Shows SLA deadline

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/change-requests.test.ts
npx vitest run src/components/change-requests/request-status.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Change Requests → Approval workflow
- 02-05-employee-portal-spec.md § Journey 4: Update Bank Details → Pending approval status
- 02-05-employee-portal-spec.md § Data Models → PersonalDetailChangeRequest
