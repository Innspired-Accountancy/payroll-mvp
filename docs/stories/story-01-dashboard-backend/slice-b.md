# Slice b: Dashboard Aggregation API Endpoints

**Story:** story-01-dashboard-backend
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement the tRPC procedures that provide aggregated dashboard data for bureau operations. These endpoints power the summary cards, status breakdowns, client lists, and upcoming deadlines widgets.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, Zod 3.22.x, Drizzle ORM 0.30.x
- [x] All SDK methods/API calls identified: tRPC router, protectedProcedure, zod validation
- [x] All external service endpoints specified: N/A (internal API)
- [x] All data contracts defined: DashboardSummary, ClientListItem, UpcomingDeadline, ExceptionSummary
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: TRPCError with appropriate codes
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § API Contracts — GET /api/v1/bureau/dashboard
- 02-04-bureau-operations-spec.md § API Contracts — GET /api/v1/bureau/clients
- 08-architecture-and-patterns.md — tRPC router structure and error handling

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/bureau-dashboard.ts` | create | tRPC router for dashboard endpoints |
| `src/lib/validation/dashboard.ts` | create | Zod schemas for dashboard inputs/outputs |
| `src/lib/services/dashboard-service.ts` | create | Business logic for dashboard aggregation |
| `src/tests/server/dashboard-router.test.ts` | create | tRPC procedure tests |

---

## Responsibilities

1. Implement dashboard.summary query for summary cards and status breakdown
2. Implement dashboard.clients query with filtering, sorting, and pagination
3. Implement dashboard.deadlines query for upcoming deadline widget
4. Implement dashboard.exceptions query for recent exceptions summary
5. Enforce bureau-level tenant isolation in all procedures

---

## Contracts

### dashboard.summary

- **Method:** tRPC query `dashboard.summary`
- **Input:** None (uses context for bureau_id)
- **Output:** DashboardSummaryOutput Zod schema

```typescript
export const DashboardSummaryOutput = z.object({
  summary: z.object({
    totalClients: z.number().int(),
    payrollsThisWeek: z.number().int(),
    overdueItems: z.number().int(),
    criticalExceptions: z.number().int(),
  }),
  statusBreakdown: z.object({
    notStarted: z.number().int(),
    dataCollection: z.number().int(),
    calculated: z.number().int(),
    inReview: z.number().int(),
    approved: z.number().int(),
    submitted: z.number().int(),
    complete: z.number().int(),
  }),
});
```

- **Errors:** UNAUTHORIZED (no bureau access), INTERNAL_SERVER_ERROR
- **Auth:** protectedProcedure with bureau:view permission

### dashboard.clients

- **Method:** tRPC query `dashboard.clients`
- **Input:** ClientListInput Zod schema

```typescript
export const ClientListInput = z.object({
  status: z.enum(['not_started', 'data_collection', 'calculated', 'in_review', 'approved', 'submitted', 'paid', 'complete']).optional(),
  assignedTo: z.string().uuid().optional(),
  overdue: z.boolean().optional(),
  hasExceptions: z.boolean().optional(),
  search: z.string().optional(),
  sortBy: z.enum(['name', 'status', 'payDate', 'cutOffDate', 'lastActivity']).default('lastActivity'),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
});
```

- **Output:** Paginated client list with ClientListItem schema

```typescript
export const ClientListItem = z.object({
  employerId: z.string().uuid(),
  name: z.string(),
  payeScheme: z.string().optional(),
  currentPeriod: z.string().optional(),
  status: z.enum(['not_started', 'data_collection', 'calculated', 'in_review', 'approved', 'submitted', 'paid', 'complete']),
  cutOffDate: z.string().date().optional(),
  payDate: z.string().date().optional(),
  assignedProcessor: z.object({ id: z.string().uuid(), name: z.string() }).optional(),
  daysOverdue: z.number().int().optional(),
  hasExceptions: z.boolean(),
});

export const ClientListOutput = z.object({
  clients: z.array(ClientListItem),
  total: z.number().int(),
  page: z.number().int(),
  pageSize: z.number().int(),
  totalPages: z.number().int(),
});
```

- **Errors:** UNAUTHORIZED, BAD_REQUEST (invalid filters)
- **Auth:** protectedProcedure with bureau:view permission

### dashboard.deadlines

- **Method:** tRPC query `dashboard.deadlines`
- **Input:** z.object({ days: z.number().int().min(1).max(90).default(14) })
- **Output:** z.array(UpcomingDeadline)

```typescript
export const UpcomingDeadline = z.object({
  employerId: z.string().uuid(),
  employerName: z.string(),
  deadlineType: z.enum(['cut_off', 'pay_date', 'filing_deadline']),
  deadlineDate: z.string().date(),
  daysRemaining: z.number().int(),
  status: z.enum(['on_track', 'attention_needed', 'overdue']),
  payrollStatus: z.enum(['not_started', 'data_collection', 'calculated', 'in_review', 'approved', 'submitted', 'paid', 'complete']),
});
```

- **Errors:** UNAUTHORIZED
- **Auth:** protectedProcedure with bureau:view permission

### dashboard.exceptions

- **Method:** tRPC query `dashboard.exceptions`
- **Input:** z.object({ limit: z.number().int().min(1).max(50).default(10) })
- **Output:** z.array(ExceptionSummary)

```typescript
export const ExceptionSummary = z.object({
  id: z.string().uuid(),
  type: z.enum(['hmrc_failure', 'pension_failure', 'payment_failure', 'missing_data', 'approval_pending', 'validation_error']),
  severity: z.enum(['low', 'medium', 'high', 'critical']),
  employerName: z.string(),
  description: z.string(),
  assignedTo: z.object({ id: z.string().uuid(), name: z.string() }).optional(),
  createdAt: z.string().datetime(),
});
```

- **Errors:** UNAUTHORIZED
- **Auth:** protectedProcedure with bureau:view permission

---

## Business Rules & Invariants

1. All queries are scoped to the user's bureau_id from tRPC context
2. Users can only see clients they have permission to access (via RBAC)
3. overdueItems count includes clients where days_overdue > 0
4. criticalExceptions count includes exceptions with severity = 'critical' and status != 'resolved'
5. payrollsThisWeek counts clients where pay_date is within next 7 days

---

## Edge Cases

1. **No clients in bureau** — Return empty results with 0 counts, not error
2. **Large result sets** — Pagination enforced, max 100 items per page
3. **Invalid sort field** — Fall back to default sort, log warning
4. **Database view stale** — Implement refresh mechanism on status changes

---

## Tests

### dashboard-router.test.ts

- dashboard.summary returns correct aggregation for bureau
- dashboard.clients with status filter returns only matching clients
- dashboard.clients with overdue=true returns only overdue clients
- dashboard.clients pagination works correctly
- dashboard.deadlines returns deadlines within specified days
- dashboard.exceptions returns most recent critical/high exceptions
- Unauthorized user receives UNAUTHORIZED error
- Query performance under 500ms for 1000 client portfolio

---

## Verification

```bash
npm run test:unit -- dashboard-router.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § API Contracts → GET /api/v1/bureau/dashboard
- 02-04-bureau-operations-spec.md § API Contracts → GET /api/v1/bureau/clients
