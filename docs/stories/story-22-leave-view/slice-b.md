# Slice b: Leave Request History List with Status Filtering

**Story:** story-22-leave-view
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** slice-a (leave balance summary)

---

## Goal

Create the leave request history list that displays all past and upcoming leave requests with their status. Employees can view request details, filter by status, and sort by date to track their leave usage.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: leave.getRequests(query), leave.cancelRequest(id)
- [x] External service endpoints: None (internal database queries)
- [x] Data contracts defined: LeaveRequestItem, LeaveRequestStatus, LeaveRequestQuery
- [x] Configuration variables: REQUEST_PAGE_SIZE=20
- [x] Error scenarios identified: No requests found, unauthorized cancel, already processed
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 3: Request Leave (history view)
- 02-01-core-payroll-spec.md — Leave request lifecycle
- 08-architecture-and-patterns.md — Status patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/leave.ts` | update | Add getRequests procedure |
| `src/components/leave/leave-history.tsx` | create | Leave history list component |
| `src/components/leave/leave-request-row.tsx` | create | Individual request row |
| `src/components/leave/status-badge.tsx` | create | Status indicator component |
| `src/components/leave/leave-filters.tsx` | create | Filter controls |
| `src/app/(employee)/leave/history/page.tsx` | create | History page |

---

## Responsibilities

1. Display paginated list of leave requests
2. Show date range, leave type, days, status per request
3. Support filtering by status (pending, approved, rejected, cancelled)
4. Support sorting by date (newest first default)
5. Color-code status badges (green=approved, yellow=pending, red=rejected)
6. Allow cancellation of pending requests
7. Show approver name and notes for processed requests

---

## Contracts

### leave.getRequests
- **Method:** tRPC query `leave.getRequests`
- **Input:**
  ```typescript
  {
    page?: number;              // Default: 1
    pageSize?: number;          // Default: 20
    status?: ("pending" | "approved" | "rejected" | "cancelled")[];
    leaveType?: string;         // Filter by type
    year?: number;              // Filter by leave year
    sortBy?: "startDate" | "submittedAt" | "status";
    sortOrder?: "asc" | "desc";
  }
  ```
- **Output:**
  ```typescript
  {
    requests: [
      {
        id: string;                 // UUID
        leaveType: string;          // "annual_leave"
        leaveTypeLabel: string;     // "Annual Leave"
        startDate: string;          // ISO date
        endDate: string;            // ISO date
        days: number;               // 5.0
        status: "pending" | "approved" | "rejected" | "cancelled";
        reason: string;             // "Holiday"
        submittedAt: string;        // ISO timestamp
        approvedBy: string | null;  // Approver name
        approvedAt: string | null;  // ISO timestamp
        approverNotes: string | null;
        canCancel: boolean;         // true if pending
      }
    ];
    pagination: {
      page: number;
      pageSize: number;
      totalCount: number;
      totalPages: number;
    };
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `BAD_REQUEST` — Invalid filter parameters
- **Auth:** Protected procedure with employee session

### leave.cancelRequest
- **Method:** tRPC mutation `leave.cancelRequest`
- **Input:**
  ```typescript
  {
    id: string;                 // Request UUID
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
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — Request doesn't exist
  - `FORBIDDEN` — Request belongs to different employee
  - `CONFLICT` — Request not pending (already processed)
- **Auth:** Protected procedure with employee session validation

### LeaveRequestItem Schema
```typescript
export const leaveRequestStatusSchema = z.enum([
  "pending",
  "approved",
  "rejected",
  "cancelled",
]);

export const leaveRequestItemSchema = z.object({
  id: z.string().uuid(),
  leaveType: z.string(),
  leaveTypeLabel: z.string(),
  startDate: z.string().datetime(),
  endDate: z.string().datetime(),
  days: z.number().positive(),
  status: leaveRequestStatusSchema,
  reason: z.string(),
  submittedAt: z.string().datetime(),
  approvedBy: z.string().nullable(),
  approvedAt: z.string().datetime().nullable(),
  approverNotes: z.string().nullable(),
  canCancel: z.boolean(),
});
```

---

## Business Rules & Invariants

1. Employee can only view their own leave requests
2. Cancellation only allowed for pending requests
3. Approved requests update balance (taken/booked)
4. History retained indefinitely for audit
5. Past approved requests cannot be modified
6. Rejected requests show reason for transparency

---

## Edge Cases

1. **No requests** — Show empty state with "Request Leave" CTA
2. **Multi-day spanning years** — Count days in appropriate leave year
3. **Partial day** — Display as 0.5 days
4. **Cancelled after approval** — Requires admin intervention (can't self-cancel)
5. **Bulk cancellation** — Not supported, cancel individually

---

## Tests

### leave.router.test.ts
- getRequests returns paginated results
- Filtering by status works
- Cancel request updates status
- Cannot cancel non-pending request

### leave-history.test.tsx
- Renders list of requests
- Filter controls functional
- Cancel button appears for pending
- Status badges color-coded
- Pagination works

### status-badge.test.tsx
- Renders correct color per status
- Accessible label for screen readers

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/leave.test.ts
npx vitest run src/components/leave/leave-history.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Leave View → History
- 02-05-employee-portal-spec.md § Journey 3: Request Leave → History tracking
- 02-01-core-payroll-spec.md § Leave request lifecycle
