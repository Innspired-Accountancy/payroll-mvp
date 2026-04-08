# Slice a: Leave Balance Summary with Real-Time Calculation

**Story:** story-22-leave-view
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** story-18-portal-auth (authentication), Leave Module (balance data)

---

## Goal

Create the leave balance summary API and UI that displays current entitlement, taken, booked, and remaining days for all leave types. Balance data is fetched in real-time from the Leave Module to ensure accuracy.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: leave.getBalances(), leave.calculateBalance()
- [x] External service endpoints: None (internal Leave Module API)
- [x] Data contracts defined: LeaveBalance, LeaveTypeBalance, LeaveYearConfig
- [x] Configuration variables: LEAVE_YEAR_START=1 (January 1st)
- [x] Error scenarios identified: Leave module unavailable, invalid leave year, calculation errors
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 3: Request Leave (balance view)
- 02-01-core-payroll-spec.md — Leave types and entitlements
- 08-architecture-and-patterns.md — Integration with Leave Module

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/leave.ts` | create | Leave tRPC procedures |
| `src/app/(employee)/leave/page.tsx` | create | Leave page with balance |
| `src/components/leave/leave-balance.tsx` | create | Balance summary component |
| `src/components/leave/balance-card.tsx` | create | Individual leave type card |
| `src/lib/validation/leave.ts` | create | Leave schemas |
| `src/lib/calculations/leave.ts` | create | Leave calculation utilities |

---

## Responsibilities

1. Fetch real-time leave balances from Leave Module
2. Display entitlement, taken, booked, remaining per leave type
3. Show annual leave, sick leave, and other configured types
4. Calculate remaining = entitlement - taken - booked
5. Display leave year (e.g., "2026 leave year")
6. Show carry-over days separately if applicable
7. Handle prorated entitlements for new starters

---

## Contracts

### leave.getBalances
- **Method:** tRPC query `leave.getBalances`
- **Input:**
  ```typescript
  {
    leaveYear?: number;     // Default: current leave year
  }
  ```
- **Output:**
  ```typescript
  {
    leaveYear: number;      // 2026
    yearStartDate: string;  // ISO date
    yearEndDate: string;    // ISO date
    balances: [
      {
        leaveType: string;         // "annual_leave"
        leaveTypeLabel: string;    // "Annual Leave"
        entitlement: number;       // 25.0 days
        carriedOver: number;       // 5.0 days
        taken: number;             // 10.0 days
        booked: number;            // 5.0 days (pending approval)
        remaining: number;         // 15.0 days
        unit: "days" | "hours";
        color: string;             // UI color code
        icon: string;              // Lucide icon name
      }
    ];
    totalRemaining: number;   // Sum across all types
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — Leave year not found
  - `SERVICE_UNAVAILABLE` — Leave Module error
- **Auth:** Protected procedure with employee session

### LeaveBalance Schema
```typescript
export const leaveTypeBalanceSchema = z.object({
  leaveType: z.string(),
  leaveTypeLabel: z.string(),
  entitlement: z.number().nonnegative(),
  carriedOver: z.number().nonnegative(),
  taken: z.number().nonnegative(),
  booked: z.number().nonnegative(),
  remaining: z.number(),  // Can be negative if overdrawn
  unit: z.enum(["days", "hours"]),
  color: z.string(),
  icon: z.string(),
});

export const leaveBalancesResponseSchema = z.object({
  leaveYear: z.number(),
  yearStartDate: z.string().datetime(),
  yearEndDate: z.string().datetime(),
  balances: z.array(leaveTypeBalanceSchema),
  totalRemaining: z.number(),
});
```

---

## Business Rules & Invariants

1. Leave year typically runs January-December (configurable per employer)
2. Remaining = entitlement + carriedOver - taken - booked
3. Negative remaining indicates overdrawn balance
4. Booked = approved future leave + pending requests
5. Taken = approved past leave days
6. Carried over from previous year (if employer allows)
7. Prorated for new starters based on start date

---

## Edge Cases

1. **New starter** — Prorated entitlement based on start date
2. **Leaver** — No new bookings, show final balance
3. **Overdrawn balance** — Show negative remaining in red
4. **Multiple employments** — Separate balances per employment
5. **Zero entitlement** — Show "No entitlement" message
6. **Hours-based leave** — Display in hours, not days

---

## Tests

### leave.router.test.ts
- getBalances returns correct calculations
- Handles prorated entitlements
- Returns error for invalid leave year
- Validates employee ownership

### leave-balance.test.tsx
- Renders all leave type cards
- Displays correct remaining days
- Shows carry-over if applicable
- Color-codes negative balances
- Loading and error states

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/leave.test.ts
npx vitest run src/components/leave/leave-balance.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Leave View → Balance, history
- 02-05-employee-portal-spec.md § Journey 3: Request Leave → Display balance
- 02-05-employee-portal-spec.md § API Contracts → Leave balance response
