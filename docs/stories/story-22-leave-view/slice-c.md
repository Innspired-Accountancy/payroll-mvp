# Slice c: Leave Request Detail View with Approver Notes

**Story:** story-22-leave-view
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** slice-b (leave request history list)

---

## Goal

Create the detailed view for individual leave requests that shows complete information including approver notes, timeline, and related details. This provides transparency into the approval process.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: leave.getRequestById(id)
- [x] External service endpoints: None (internal database queries)
- [x] Data contracts defined: LeaveRequestDetail, LeaveRequestTimeline
- [x] Configuration variables: None
- [x] Error scenarios identified: Request not found, unauthorized access
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 3: Request Leave
- 02-01-core-payroll-spec.md — Leave request approval workflow

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(employee)/leave/requests/[id]/page.tsx` | create | Request detail page |
| `src/components/leave/leave-request-detail.tsx` | create | Detail view component |
| `src/components/leave/request-timeline.tsx` | create | Approval timeline |
| `src/server/routers/leave.ts` | update | Add getRequestById procedure |

---

## Responsibilities

1. Display full request details including reason and notes
2. Show approval timeline (submitted → reviewed → approved/rejected)
3. Display approver name and notes for transparency
4. Show date breakdown (individual days if multi-day)
5. Display leave balance impact
6. Provide cancel action for pending requests
7. Show related requests if part of series

---

## Contracts

### leave.getRequestById
- **Method:** tRPC query `leave.getRequestById`
- **Input:**
  ```typescript
  {
    id: string;                 // Request UUID
  }
  ```
- **Output:**
  ```typescript
  {
    id: string;
    leaveType: string;
    leaveTypeLabel: string;
    startDate: string;
    endDate: string;
    days: number;
    status: "pending" | "approved" | "rejected" | "cancelled";
    reason: string;
    submittedAt: string;
    submittedBy: string;        // Employee name
    timeline: [
      {
        status: "submitted" | "pending" | "approved" | "rejected" | "cancelled";
        timestamp: string;
        actor: string;          // Person who actioned
        notes: string | null;
      }
    ];
    approval: {
      approvedBy: string | null;
      approvedAt: string | null;
      approverNotes: string | null;
    } | null;
    balanceImpact: {
      year: number;
      previousRemaining: number;
      newRemaining: number;
    };
    canCancel: boolean;
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — Request doesn't exist or not owned by employee
- **Auth:** Protected procedure with employee session validation

### LeaveRequestDetail Schema
```typescript
export const timelineEventSchema = z.object({
  status: z.enum(["submitted", "pending", "approved", "rejected", "cancelled"]),
  timestamp: z.string().datetime(),
  actor: z.string(),
  notes: z.string().nullable(),
});

export const leaveRequestDetailSchema = z.object({
  id: z.string().uuid(),
  leaveType: z.string(),
  leaveTypeLabel: z.string(),
  startDate: z.string().datetime(),
  endDate: z.string().datetime(),
  days: z.number().positive(),
  status: leaveRequestStatusSchema,
  reason: z.string(),
  submittedAt: z.string().datetime(),
  submittedBy: z.string(),
  timeline: z.array(timelineEventSchema),
  approval: z.object({
    approvedBy: z.string().nullable(),
    approvedAt: z.string().datetime().nullable(),
    approverNotes: z.string().nullable(),
  }).nullable(),
  balanceImpact: z.object({
    year: z.number(),
    previousRemaining: z.number(),
    newRemaining: z.number(),
  }),
  canCancel: z.boolean(),
});
```

---

## Business Rules & Invariants

1. Timeline shows all status changes in chronological order
2. Approver notes visible to employee for transparency
3. Balance impact calculated at time of viewing
4. Cancel action only available for pending status
5. Rejection reason must be provided by approver

---

## Edge Cases

1. **Pending for long time** — Show days since submission
2. **No approver notes** — Show "No notes provided"
3. **Multi-stage approval** — Show all approval steps in timeline
4. **Auto-approved** — Mark timeline event as "auto-approved"
5. **Cancelled by admin** — Show as cancelled with admin note

---

## Tests

### leave.router.test.ts
- getRequestById returns full details
- Timeline in correct order
- Rejects unauthorized access

### leave-request-detail.test.tsx
- Renders all request details
- Timeline displayed correctly
- Cancel button for pending
- Balance impact shown

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/leave.test.ts
npx vitest run src/components/leave/leave-request-detail.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Leave View → Detail view
- 02-05-employee-portal-spec.md § Journey 3: Request Leave → Approval notification
