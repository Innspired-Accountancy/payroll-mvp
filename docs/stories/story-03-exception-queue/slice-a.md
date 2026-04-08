# Slice a: Exception Data Model and API

**Story:** story-03-exception-queue
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** story-01-dashboard-backend

---

## Goal

Create the exception data model and management API that supports tracking payroll-related issues. Implement CRUD operations, filtering, and assignment workflows for the exception queue.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x, tRPC 11.x
- [x] All SDK methods/API calls identified: tRPC procedures, Drizzle transactions
- [x] All external service endpoints specified: N/A (internal API)
- [x] All data contracts defined: ExceptionQueueItem schema, ExceptionCreateInput, ExceptionUpdateInput
- [x] All configuration/environment variables listed: DATABASE_URL
- [x] All error scenarios identified with handling strategy: Validation errors, not found, concurrency
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Data Models — ExceptionQueueItem entity
- 02-04-bureau-operations-spec.md § API Contracts — POST /api/v1/exceptions/{id}/assign

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/exceptions.ts` | create | Exception queue table schema |
| `src/server/routers/exceptions.ts` | create | tRPC router for exception management |
| `src/lib/validation/exceptions.ts` | create | Zod schemas for exception inputs |
| `src/lib/services/exception-service.ts` | create | Business logic for exception operations |
| `src/tests/server/exceptions-router.test.ts` | create | Exception router tests |

---

## Responsibilities

1. Define ExceptionQueueItem table with all fields
2. Implement exception.create for manual exception creation
3. Implement exception.list with filtering and pagination
4. Implement exception.assign for assignment workflow
5. Implement exception.resolve for resolution workflow
6. Implement exception.getById for detail view

---

## Contracts

### ExceptionQueueItem Schema

```typescript
// src/lib/db/schema/exceptions.ts
export const exceptionTypeEnum = pgEnum('exception_type', [
  'hmrc_failure',
  'pension_failure',
  'payment_failure',
  'missing_data',
  'approval_pending',
  'validation_error'
]);

export const exceptionSeverityEnum = pgEnum('exception_severity', [
  'low',
  'medium',
  'high',
  'critical'
]);

export const exceptionStatusEnum = pgEnum('exception_status', [
  'open',
  'assigned',
  'resolved',
  'escalated'
]);

export const exceptionQueueItem = pgTable('exception_queue_items', {
  id: uuid('id').primaryKey().defaultRandom(),
  bureauId: uuid('bureau_id').notNull().references(() => bureaus.id),
  employerId: uuid('employer_id').notNull().references(() => employers.id),
  exceptionType: exceptionTypeEnum('exception_type').notNull(),
  severity: exceptionSeverityEnum('severity').notNull(),
  status: exceptionStatusEnum('status').notNull().default('open'),
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description').notNull(),
  sourceModule: varchar('source_module', { length: 100 }).notNull(),
  sourceReference: uuid('source_reference'),
  assignedTo: uuid('assigned_to').references(() => users.id),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  resolvedAt: timestamp('resolved_at'),
  resolutionNotes: text('resolution_notes'),
  createdBy: uuid('created_by').references(() => users.id),
});
```

### exceptions.create

- **Method:** tRPC mutation `exceptions.create`
- **Input:** ExceptionCreateInput

```typescript
export const ExceptionCreateInput = z.object({
  employerId: z.string().uuid(),
  exceptionType: z.enum(['hmrc_failure', 'pension_failure', 'payment_failure', 'missing_data', 'approval_pending', 'validation_error']),
  severity: z.enum(['low', 'medium', 'high', 'critical']),
  title: z.string().min(1).max(255),
  description: z.string().min(1),
  sourceModule: z.string().min(1).max(100),
  sourceReference: z.string().uuid().optional(),
});
```

- **Output:** ExceptionQueueItem
- **Errors:** BAD_REQUEST (validation), NOT_FOUND (employer), UNAUTHORIZED
- **Auth:** protectedProcedure with exception:create permission

### exceptions.list

- **Method:** tRPC query `exceptions.list`
- **Input:** ExceptionListInput

```typescript
export const ExceptionListInput = z.object({
  status: z.enum(['open', 'assigned', 'resolved', 'escalated']).optional(),
  type: z.enum(['hmrc_failure', 'pension_failure', 'payment_failure', 'missing_data', 'approval_pending', 'validation_error']).optional(),
  severity: z.enum(['low', 'medium', 'high', 'critical']).optional(),
  assignedTo: z.string().uuid().optional(),
  employerId: z.string().uuid().optional(),
  sortBy: z.enum(['createdAt', 'severity', 'status']).default('createdAt'),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
});
```

- **Output:** Paginated exception list
- **Errors:** UNAUTHORIZED
- **Auth:** protectedProcedure with exception:view permission

### exceptions.assign

- **Method:** tRPC mutation `exceptions.assign`
- **Input:** z.object({ id: z.string().uuid(), assignedTo: z.string().uuid(), notes: z.string().optional() })
- **Output:** ExceptionQueueItem
- **Errors:** NOT_FOUND, CONFLICT (already resolved), UNAUTHORIZED
- **Auth:** protectedProcedure with exception:assign permission

### exceptions.resolve

- **Method:** tRPC mutation `exceptions.resolve`
- **Input:** z.object({ id: z.string().uuid(), resolutionNotes: z.string().min(1) })
- **Output:** ExceptionQueueItem
- **Errors:** NOT_FOUND, CONFLICT (already resolved), UNAUTHORIZED
- **Auth:** protectedProcedure with exception:resolve permission

---

## Business Rules & Invariants

1. Exception status flows: open → assigned → resolved OR open → escalated
2. Only open or assigned exceptions can be resolved
3. Resolution requires notes explaining how it was resolved
4. Critical severity exceptions notify assigned user immediately
5. resolvedAt is set automatically when status changes to resolved
6. Creating an exception sets has_exceptions = true on ClientPayrollStatus

---

## Edge Cases

1. **Duplicate exception** — Check for existing open exception on same employer/type/source
2. **Assign to self** — Allowed for all users with exception:assign permission
3. **Resolve without notes** — Validation error, notes required
4. **Concurrent assignment** — Last write wins, notify first assignee

---

## Tests

### exceptions-router.test.ts

- Create exception with valid data
- Create exception with invalid employer returns 404
- List exceptions with status filter
- Assign exception updates status to assigned
- Resolve exception requires notes
- Cannot resolve already resolved exception
- Unauthorized user cannot create exception

---

## Verification

```bash
npm run test:unit -- exceptions-router.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Data Models → ExceptionQueueItem entity
- 02-04-bureau-operations-spec.md § API Contracts → Exception assignment endpoint
