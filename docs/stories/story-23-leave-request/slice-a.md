# Slice a: Leave Request Form with Date Picker and Validation

**Story:** story-23-leave-request
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** story-22-leave-view (leave viewing)

---

## Goal

Create the leave request form with date picker, leave type selection, and comprehensive validation. The form provides real-time balance preview and validates all inputs before submission.

---

## Decision Checklist

- [x] All libraries/packages named: React Hook Form 7.51.x, Zod 3.22.x, date-fns 3.6.x, react-day-picker 8.10.x
- [x] SDK methods/API calls identified: leave.getTypes(), leave.calculateDays()
- [x] External service endpoints: None (internal calculations)
- [x] Data contracts defined: LeaveRequestInput, LeaveType, DateRange
- [x] Configuration variables: MAX_FUTURE_BOOKING_DAYS=365, MIN_NOTICE_DAYS=1
- [x] Error scenarios identified: Insufficient balance, date conflicts, past dates
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 3: Request Leave (submission form)
- 02-01-core-payroll-spec.md — Leave request validation rules

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(employee)/leave/request/page.tsx` | create | Request form page |
| `src/components/leave/leave-request-form.tsx` | create | Main form component |
| `src/components/leave/date-range-picker.tsx` | create | Date selection |
| `src/components/leave/leave-type-selector.tsx` | create | Type dropdown |
| `src/components/leave/balance-preview.tsx` | create | Real-time balance |
| `src/lib/validation/leave-request.ts` | create | Form validation schemas |

---

## Responsibilities

1. Display leave type dropdown (from employer configuration)
2. Provide date range picker with disabled past dates
3. Calculate days automatically based on date range
4. Validate sufficient balance before submission
5. Show real-time balance preview
6. Validate minimum notice period (configurable)
7. Provide reason textarea

---

## Contracts

### leave.getTypes
- **Method:** tRPC query `leave.getTypes`
- **Input:** None
- **Output:**
  ```typescript
  {
    types: [
      {
        id: string;
        name: string;           // "annual_leave"
        label: string;          // "Annual Leave"
        requiresApproval: boolean;
        maxConsecutiveDays: number | null;
        color: string;
      }
    ];
  }
  ```
- **Auth:** Protected procedure

### leave.calculateDays
- **Method:** tRPC query `leave.calculateDays`
- **Input:**
  ```typescript
  {
    leaveType: string;
    startDate: string;        // ISO date
    endDate: string;          // ISO date
  }
  ```
- **Output:**
  ```typescript
  {
    days: number;             // Working days
    dates: string[];          // Individual dates
    exceedsLimit: boolean;    // If > maxConsecutiveDays
  }
  ```
- **Auth:** Protected procedure

### LeaveRequestInput Schema
```typescript
export const leaveRequestInputSchema = z.object({
  leaveType: z.string().min(1, "Select a leave type"),
  startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, "Invalid date format"),
  endDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, "Invalid date format"),
  days: z.number().positive("Must be at least 1 day"),
  reason: z.string().max(500, "Reason must be under 500 characters").optional(),
}).refine((data) => {
  const start = new Date(data.startDate);
  const end = new Date(data.endDate);
  return start <= end;
}, {
  message: "End date must be after start date",
  path: ["endDate"],
});
```

---

## Business Rules & Invariants

1. Start date must be today or future (no backdating)
2. End date must be on or after start date
3. Minimum 1 day notice (configurable by employer)
4. Maximum booking 365 days in future
5. Days calculated as working days (excluding weekends/holidays)
6. Real-time balance check before submission

---

## Edge Cases

1. **Weekend-only selection** — Calculate as 0 working days, show warning
2. **Public holidays** — Exclude from working day count
3. **Half-day request** — Not supported in MVP (full days only)
4. **Negative balance** — Show warning if request exceeds remaining
5. **Future leave year** — Allow booking into next year if configured

---

## Tests

### leave-request-form.test.tsx
- Renders all form fields
- Date picker disables past dates
- Days calculated on date change
- Validation shows errors
- Submit calls API

### leave.router.test.ts
- calculateDays returns correct count
- Excludes weekends correctly
- Validates against limits

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/components/leave/leave-request-form.test.tsx
npx vitest run src/server/routers/leave.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Leave Request → Request workflow
- 02-05-employee-portal-spec.md § Journey 3: Request Leave → Select dates and type
