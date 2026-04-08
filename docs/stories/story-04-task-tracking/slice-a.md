# Slice a: Task Data Model and API

**Story:** story-04-task-tracking
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** story-02-dashboard-ui

---

## Goal

Create the task tracking data model and API for payroll processing workflows. Implement CRUD operations, assignment capabilities, and status tracking for tasks linked to payroll periods.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x, tRPC 11.x, date-fns 3.x
- [x] All SDK methods/API calls identified: tRPC procedures, Drizzle transactions
- [x] All external service endpoints specified: N/A (internal API)
- [x] All data contracts defined: Task schema, TaskCreateInput, TaskUpdateInput, TaskStatus enum
- [x] All configuration/environment variables listed: DATABASE_URL
- [x] All error scenarios identified with handling strategy: Validation, not found, concurrency
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Data Models — Task patterns
- 08-architecture-and-patterns.md — Multi-tenant schema design

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/tasks.ts` | create | Task table schema |
| `src/server/routers/tasks.ts` | create | tRPC router for task management |
| `src/lib/validation/tasks.ts` | create | Zod schemas for task inputs |
| `src/lib/services/task-service.ts` | create | Business logic for task operations |
| `src/tests/server/tasks-router.test.ts` | create | Task router tests |

---

## Responsibilities

1. Define Task table with fields for payroll workflow tracking
2. Implement task.create for manual task creation
3. Implement task.list with filtering by assignee, status, due date
4. Implement task.update for status changes and reassignment
5. Implement task.delete for task removal
6. Implement task.getById for detail view

---

## Contracts

### Task Table Schema

```typescript
// src/lib/db/schema/tasks.ts
export const taskTypeEnum = pgEnum('task_type', [
  'data_collection',
  'review',
  'approval',
  'submission',
  'follow_up',
  'client_chase'
]);

export const taskStatusEnum = pgEnum('task_status', [
  'pending',
  'in_progress',
  'blocked',
  'complete'
]);

export const taskPriorityEnum = pgEnum('task_priority', [
  'low',
  'normal',
  'high',
  'urgent'
]);

export const tasks = pgTable('tasks', {
  id: uuid('id').primaryKey().defaultRandom(),
  bureauId: uuid('bureau_id').notNull().references(() => bureaus.id),
  employerId: uuid('employer_id').notNull().references(() => employers.id),
  payPeriodId: uuid('pay_period_id').references(() => payPeriods.id),
  taskType: taskTypeEnum('task_type').notNull(),
  title: varchar('title', { length: 255 }).notNull(),
  description: text('description'),
  status: taskStatusEnum('task_status').notNull().default('pending'),
  priority: taskPriorityEnum('task_priority').notNull().default('normal'),
  assignedTo: uuid('assigned_to').references(() => users.id),
  dueDate: date('due_date'),
  completedAt: timestamp('completed_at'),
  completedBy: uuid('completed_by').references(() => users.id),
  createdBy: uuid('created_by').notNull().references(() => users.id),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});
```

### tasks.create

- **Method:** tRPC mutation `tasks.create`
- **Input:** TaskCreateInput

```typescript
export const TaskCreateInput = z.object({
  employerId: z.string().uuid(),
  payPeriodId: z.string().uuid().optional(),
  taskType: z.enum(['data_collection', 'review', 'approval', 'submission', 'follow_up', 'client_chase']),
  title: z.string().min(1).max(255),
  description: z.string().optional(),
  priority: z.enum(['low', 'normal', 'high', 'urgent']).default('normal'),
  assignedTo: z.string().uuid().optional(),
  dueDate: z.string().date().optional(),
});
```

- **Output:** Task
- **Errors:** BAD_REQUEST (validation), NOT_FOUND (employer), UNAUTHORIZED
- **Auth:** protectedProcedure with task:create permission

### tasks.list

- **Method:** tRPC query `tasks.list`
- **Input:** TaskListInput

```typescript
export const TaskListInput = z.object({
  status: z.enum(['pending', 'in_progress', 'blocked', 'complete']).optional(),
  assignedTo: z.string().uuid().optional(),
  employerId: z.string().uuid().optional(),
  taskType: z.enum(['data_collection', 'review', 'approval', 'submission', 'follow_up', 'client_chase']).optional(),
  overdue: z.boolean().optional(),
  dueBefore: z.string().date().optional(),
  dueAfter: z.string().date().optional(),
  sortBy: z.enum(['dueDate', 'priority', 'createdAt']).default('dueDate'),
  sortOrder: z.enum(['asc', 'desc']).default('asc'),
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
});
```

- **Output:** Paginated task list
- **Errors:** UNAUTHORIZED
- **Auth:** protectedProcedure with task:view permission

### tasks.update

- **Method:** tRPC mutation `tasks.update`
- **Input:** z.object({ id: z.string().uuid(), data: TaskUpdateInput })

```typescript
export const TaskUpdateInput = z.object({
  title: z.string().min(1).max(255).optional(),
  description: z.string().optional(),
  status: z.enum(['pending', 'in_progress', 'blocked', 'complete']).optional(),
  priority: z.enum(['low', 'normal', 'high', 'urgent']).optional(),
  assignedTo: z.string().uuid().optional(),
  dueDate: z.string().date().optional(),
});
```

- **Output:** Task
- **Errors:** NOT_FOUND, UNAUTHORIZED, CONFLICT (if already complete)
- **Auth:** protectedProcedure with task:update permission

### tasks.myTasks

- **Method:** tRPC query `tasks.myTasks`
- **Input:** z.object({ status: z.enum(['pending', 'in_progress', 'blocked', 'complete']).optional(), overdue: z.boolean().optional() })
- **Output:** Array of Task (for current user)
- **Errors:** UNAUTHORIZED
- **Auth:** protectedProcedure

---

## Business Rules & Invariants

1. Task status flows: pending → in_progress → complete OR pending → blocked → in_progress → complete
2. Setting status to 'complete' sets completedAt and completedBy automatically
3. Overdue tasks have dueDate < current_date and status != 'complete'
4. Task completion can trigger payroll status updates via event
5. Reassignment is allowed at any point before completion
6. Urgent priority tasks notify assignee immediately

---

## Edge Cases

1. **Complete to incomplete** — Not allowed, create follow-up task instead
2. **Assign to inactive user** — Validation error
3. **Due date in past** — Allowed but flagged as overdue
4. **Bulk task operations** — Batch update API for efficiency

---

## Tests

### tasks-router.test.ts

- Create task with all fields
- Create task without optional fields
- List tasks with assignee filter
- List tasks with overdue filter
- Update task status to complete sets completedAt
- Cannot update completed task
- Unauthorized user cannot create task

---

## Verification

```bash
npm run test:unit -- tasks-router.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Task assignment patterns
- 08-architecture-and-patterns.md → Multi-tenant data model
