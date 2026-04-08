# Slice a: Client Payroll Status Data Model and Views

**Story:** story-01-dashboard-backend
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** none

---

## Goal

Create the database schema and views that support real-time payroll status aggregation across all clients in a bureau. This includes the ClientPayrollStatus entity, materialized views for performance, and indexing strategy for dashboard queries.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, PostgreSQL 16
- [x] All SDK methods/API calls identified: Drizzle schema definition, index creation, view creation
- [x] All external service endpoints specified: N/A (database layer only)
- [x] All data contracts defined: ClientPayrollStatus schema, PayrollStatus enum, DashboardSummary type
- [x] All configuration/environment variables listed: DATABASE_URL
- [x] All error scenarios identified with handling strategy: constraint violations, concurrent updates
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Data Models — ClientPayrollStatus entity definition
- 02-04-bureau-operations-spec.md § Data Models — ExceptionQueueItem entity
- 08-architecture-and-patterns.md — Multi-tenant schema design

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/client-payroll-status.ts` | create | Drizzle table schema for client payroll status |
| `src/lib/db/schema/bureau-dashboard.ts` | create | Bureau dashboard views and related tables |
| `src/lib/db/migrations/0014_add_dashboard_views.sql` | create | Materialized view for dashboard aggregation |
| `src/lib/db/seed/dashboard-test-data.ts` | create | Test data for dashboard queries |
| `src/tests/db/dashboard-views.test.ts` | create | Database view unit tests |

---

## Responsibilities

1. Define ClientPayrollStatus table with all status fields and foreign keys
2. Create database indexes for efficient filtering and aggregation
3. Implement materialized view for dashboard summary data
4. Ensure tenant isolation via bureau_id and employer_id relationships
5. Support soft deletes and audit timestamps

---

## Contracts

### ClientPayrollStatus Table Schema

```typescript
// src/lib/db/schema/client-payroll-status.ts
export const payrollStatusEnum = pgEnum('payroll_status', [
  'not_started',
  'data_collection',
  'calculated',
  'in_review',
  'approved',
  'submitted',
  'paid',
  'complete'
]);

export const clientPayrollStatus = pgTable('client_payroll_status', {
  id: uuid('id').primaryKey().defaultRandom(),
  bureauId: uuid('bureau_id').notNull().references(() => bureaus.id),
  employerId: uuid('employer_id').notNull().references(() => employers.id),
  currentPeriodId: uuid('current_period_id').references(() => payPeriods.id),
  status: payrollStatusEnum('status').notNull().default('not_started'),
  cutOffDate: date('cut_off_date'),
  payDate: date('pay_date'),
  filingDeadline: date('filing_deadline'),
  assignedProcessorId: uuid('assigned_processor_id').references(() => users.id),
  assignedReviewerId: uuid('assigned_reviewer_id').references(() => users.id),
  lastActivityAt: timestamp('last_activity_at').notNull().defaultNow(),
  daysOverdue: integer('days_overdue'),
  hasExceptions: boolean('has_exceptions').notNull().default(false),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});
```

### DashboardSummary View

```sql
-- Materialized view for dashboard aggregation
CREATE MATERIALIZED VIEW bureau_dashboard_summary AS
SELECT 
  bureau_id,
  COUNT(*) as total_clients,
  COUNT(*) FILTER (WHERE status = 'not_started') as not_started_count,
  COUNT(*) FILTER (WHERE status = 'data_collection') as data_collection_count,
  COUNT(*) FILTER (WHERE status = 'calculated') as calculated_count,
  COUNT(*) FILTER (WHERE status = 'in_review') as in_review_count,
  COUNT(*) FILTER (WHERE status = 'approved') as approved_count,
  COUNT(*) FILTER (WHERE status = 'submitted') as submitted_count,
  COUNT(*) FILTER (WHERE status = 'complete') as complete_count,
  COUNT(*) FILTER (WHERE days_overdue > 0) as overdue_count,
  COUNT(*) FILTER (WHERE has_exceptions = true) as has_exceptions_count,
  MAX(updated_at) as last_updated
FROM client_payroll_status
GROUP BY bureau_id;
```

### Indexes

```sql
CREATE INDEX idx_cps_bureau_id ON client_payroll_status(bureau_id);
CREATE INDEX idx_cps_status ON client_payroll_status(status);
CREATE INDEX idx_cps_assigned_processor ON client_payroll_status(assigned_processor_id);
CREATE INDEX idx_cps_cut_off_date ON client_payroll_status(cut_off_date);
CREATE INDEX idx_cps_overdue ON client_payroll_status(days_overdue) WHERE days_overdue > 0;
CREATE INDEX idx_cps_has_exceptions ON client_payroll_status(has_exceptions) WHERE has_exceptions = true;
CREATE INDEX idx_cps_last_activity ON client_payroll_status(last_activity_at DESC);
```

---

## Business Rules & Invariants

1. Every employer must have exactly one ClientPayrollStatus record per bureau
2. Status transitions follow the workflow: not_started → data_collection → calculated → in_review → approved → submitted → paid → complete
3. days_overdue is calculated as: current_date - cut_off_date when cut_off_date < current_date and status < 'complete'
4. has_exceptions is true when any ExceptionQueueItem exists for the employer with status != 'resolved'
5. last_activity_at updates on any status change or assignment change

---

## Edge Cases

1. **Missing cut-off date** — days_overdue is NULL, not included in overdue count
2. **Status rollback** — Allowed for corrections but logged in audit trail
3. **Concurrent updates** — Optimistic locking via updated_at timestamp
4. **Employer transferred between bureaus** — Archive old record, create new with reset status

---

## Tests

### dashboard-views.test.ts

- Create ClientPayrollStatus with all required fields
- Update status triggers last_activity_at update
- Materialized view returns correct aggregation counts
- Query with bureau filter returns only that bureau's data
- Index usage verification for common queries
- Concurrent update handling with optimistic locking

---

## Verification

```bash
npm run db:migrate
npm run db:seed:dashboard
npm run test:unit -- dashboard-views.test.ts
npm run typecheck
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Data Models → ClientPayrollStatus entity
- 02-04-bureau-operations-spec.md § API Contracts → GET /api/v1/bureau/dashboard
