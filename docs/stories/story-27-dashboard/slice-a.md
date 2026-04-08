# Slice a: Dashboard API and Data Aggregation

**Story:** story-27-dashboard
**Epic:** epic-07-employer-portal
**Effort:** M
**Dependencies:** story-26-client-auth

## Goal

Implement the employer dashboard API endpoint that aggregates payroll status, pending actions, deadlines, and recent activity scoped to the authenticated user's employer.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `date-fns@3.x`
- [x] SDK methods identified: `db.query`, `db.select`, Drizzle relational queries
- [x] External service endpoints: N/A
- [x] Data contracts defined: DashboardResponse, PendingAction, RecentActivity
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ API Contracts → Dashboard endpoint
- 02-06-employer-portal-spec.md:§ Data Models → EmployerPortalUser

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/api/routers/dashboard.ts` | create | Dashboard tRPC router |
| `src/lib/dashboard/queries.ts` | create | Dashboard data aggregation |
| `src/lib/dashboard/types.ts` | create | Dashboard type definitions |
| `src/lib/dashboard/calculations.ts` | create | Deadline and status calculations |

## Responsibilities
1. Fetch employer details (name, PAYE reference)
2. Determine current pay period and status
3. Calculate days remaining until deadline
4. Identify pending actions for the user
5. Retrieve recent activity history
6. Enforce employer scoping on all queries
7. Optimize queries for <2 second response time

## Contracts

### DashboardResponse Type
```typescript
// src/lib/dashboard/types.ts
export interface DashboardResponse {
  employer: {
    id: string;
    name: string;
    payeReference: string;
  };
  currentPayroll: {
    periodId: string;
    period: string;  // e.g., "April 2026"
    status: 'draft' | 'awaiting_data' | 'awaiting_approval' | 'approved' | 'paid';
    payDate: Date;
    deadline: Date;
    daysRemaining: number;
    isOverdue: boolean;
  } | null;
  pendingActions: PendingAction[];
  recentActivity: RecentActivity[];
  stats: {
    totalEmployees: number;
    activePayPeriods: number;
  };
}

export interface PendingAction {
  id: string;
  type: 'payroll_approval' | 'variable_pay_input' | 'starter_pending' | 'document_required';
  title: string;
  description: string;
  dueDate: Date;
  priority: 'high' | 'medium' | 'low';
  actionUrl: string;
}

export interface RecentActivity {
  id: string;
  type: 'payroll_paid' | 'payroll_approved' | 'variable_pay_submitted' | 'starter_added' | 'leaver_processed';
  description: string;
  date: Date;
  actorName?: string;
}
```

### Dashboard Router
```typescript
// src/server/api/routers/dashboard.ts
import { router, protectedProcedure } from '@/server/api/trpc';
import { getDashboardData } from '@/lib/dashboard/queries';

export const dashboardRouter = router({
  getEmployerDashboard: protectedProcedure
    .query(async ({ ctx }): Promise<DashboardResponse> => {
      const employerId = ctx.session.user.employerId;
      
      if (!employerId) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'User not associated with an employer',
        });
      }
      
      return getDashboardData(employerId, ctx.session.user.id);
    }),
});
```

### Query Implementation
```typescript
// src/lib/dashboard/queries.ts
export async function getDashboardData(
  employerId: string,
  userId: string
): Promise<DashboardResponse> {
  // Parallel fetch of all dashboard components
  const [
    employer,
    currentPayroll,
    pendingActions,
    recentActivity,
    stats
  ] = await Promise.all([
    fetchEmployerDetails(employerId),
    fetchCurrentPayroll(employerId),
    fetchPendingActions(employerId, userId),
    fetchRecentActivity(employerId),
    fetchStats(employerId),
  ]);

  return {
    employer,
    currentPayroll,
    pendingActions,
    recentActivity,
    stats,
  };
}

async function fetchCurrentPayroll(employerId: string): Promise<DashboardResponse['currentPayroll']> {
  const payPeriod = await db.query.payPeriods.findFirst({
    where: and(
      eq(payPeriods.employerId, employerId),
      gte(payPeriods.payDate, new Date())
    ),
    orderBy: [asc(payPeriods.payDate)],
    with: {
      payRun: true,
    },
  });

  if (!payPeriod) return null;

  const now = new Date();
  const daysRemaining = differenceInDays(payPeriod.cutOffDate, now);

  return {
    periodId: payPeriod.id,
    period: format(payPeriod.periodStart, 'MMMM yyyy'),
    status: determinePayrollStatus(payPeriod),
    payDate: payPeriod.payDate,
    deadline: payPeriod.cutOffDate,
    daysRemaining: Math.max(0, daysRemaining),
    isOverdue: daysRemaining < 0,
  };
}
```

## Business Rules & Invariants
1. Dashboard data is strictly scoped to the user's employer
2. Current payroll shows the nearest upcoming pay period
3. Days remaining calculated from cut-off date (not pay date)
4. Pending actions filtered by user role permissions
5. Recent activity limited to last 30 days, max 10 items
6. Status determined from pay run state machine

## Edge Cases
1. **No active pay period** — Return null for currentPayroll with message
2. **New employer with no history** — Show empty states with onboarding guidance
3. **Deadline on weekend/bank holiday** — Use previous business day
4. **Multiple pending pay periods** — Show most urgent first
5. **Activity feed empty** — Show "No recent activity" message
6. **Database timeout** — Return partial data with error indicator

## Tests

### src/lib/dashboard/queries.test.ts
- `should return employer details scoped to user`: Scoping
- `should calculate days remaining correctly`: Date math
- `should identify pending payroll approval`: Action detection
- `should identify missing variable pay data`: Action detection
- `should return recent activity chronologically`: Sorting
- `should handle employer with no pay periods`: Empty state
- `should enforce employer isolation`: Security

### src/server/api/routers/dashboard.test.ts
- `should reject unauthenticated requests`: Auth
- `should reject user without employer`: Validation
- `should complete within 2 seconds`: Performance

## Verification
```bash
npm run test:unit src/lib/dashboard/queries.test.ts
npm run test:unit src/server/api/routers/dashboard.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-06-employer-portal-spec.md § API Contracts → GET /api/v1/employer-portal/dashboard
- 02-06-employer-portal-spec.md § User Journeys → Journey 1
