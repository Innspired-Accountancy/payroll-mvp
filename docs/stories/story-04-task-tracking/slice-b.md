# Slice b: Task Queue UI and Management

**Story:** story-04-task-tracking
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Create the task queue UI for individual users to view and manage their assigned tasks, and for managers to view team workload. Support task status updates, reassignment, and filtering by various criteria.

---

## Decision Checklist

- [x] All libraries/packages named: @tanstack/react-table 8.x, @tanstack/react-query 5.x, date-fns 3.x, Radix UI 1.x
- [x] All SDK methods/API calls identified: tRPC tasks.list, tRPC tasks.myTasks, tRPC tasks.update
- [x] All external service endpoints specified: tRPC task queries and mutations
- [x] All data contracts defined: TaskListOutput, TaskUpdateInput from slice-a
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Loading, error, empty states
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 1: Reassign overdue payroll
- 08-architecture-and-patterns.md — UI component patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(bureau)/tasks/page.tsx` | create | Task queue page |
| `src/components/tasks/task-queue.tsx` | create | Main task queue component |
| `src/components/tasks/task-table.tsx` | create | Task list table |
| `src/components/tasks/task-filters.tsx` | create | Filter panel |
| `src/components/tasks/task-status-dropdown.tsx` | create | Status change dropdown |
| `src/components/tasks/priority-badge.tsx` | create | Priority indicator |
| `src/components/tasks/overdue-indicator.tsx` | create | Overdue warning |
| `src/components/tasks/task-counts.tsx` | create | Summary counts |
| `src/tests/components/tasks/task-queue.test.tsx` | create | Component tests |

---

## Responsibilities

1. Create task queue page with personal and team views
2. Display tasks with priority, due date, status, employer, and type
3. Implement inline status updates
4. Provide reassignment capability
5. Show overdue indicators and urgency
6. Support filtering by status, type, and date range

---

## Contracts

### TaskQueue Component

```typescript
// src/components/tasks/task-queue.tsx
interface TaskQueueProps {
  view: 'my_tasks' | 'team_tasks';
  bureauId: string;
}

// Uses:
// - view === 'my_tasks': api.tasks.myTasks.useQuery()
// - view === 'team_tasks': api.tasks.list.useQuery()
export function TaskQueue({ view, bureauId }: TaskQueueProps): JSX.Element;
```

### TaskTable Component

```typescript
// src/components/tasks/task-table.tsx
interface TaskTableProps {
  tasks: Task[];
  onStatusChange: (taskId: string, status: TaskStatus) => void;
  onReassign: (taskId: string, userId: string) => void;
  showAssignee?: boolean;
}

// Columns:
// - Priority (with icon)
// - Title (with link to task)
// - Employer (with link)
// - Type
// - Due Date (with overdue indicator)
// - Status (dropdown)
// - Assignee (if team view)
// - Actions
export function TaskTable(props: TaskTableProps): JSX.Element;
```

### TaskStatusDropdown Component

```typescript
// src/components/tasks/task-status-dropdown.tsx
interface TaskStatusDropdownProps {
  value: TaskStatus;
  onChange: (status: TaskStatus) => void;
  disabled?: boolean;
}

// Statuses:
// - pending (gray)
// - in_progress (blue)
// - blocked (red)
// - complete (green)
export function TaskStatusDropdown(props: TaskStatusDropdownProps): JSX.Element;
```

### PriorityBadge Component

```typescript
// src/components/tasks/priority-badge.tsx
interface PriorityBadgeProps {
  priority: 'low' | 'normal' | 'high' | 'urgent';
}

// Colors:
// - low: gray
// - normal: blue
// - high: amber
// - urgent: red with animation
export function PriorityBadge({ priority }: PriorityBadgeProps): JSX.Element;
```

### OverdueIndicator Component

```typescript
// src/components/tasks/overdue-indicator.tsx
interface OverdueIndicatorProps {
  dueDate: Date;
  status: TaskStatus;
}

// Shows:
// - Nothing if complete
// - "Due today" in amber if due today
// - "X days overdue" in red if overdue
// - "Due in X days" in green if future
export function OverdueIndicator({ dueDate, status }: OverdueIndicatorProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Default view is "My Tasks" for processors, "Team Tasks" for managers
2. Overdue tasks sort to top by default
3. Urgent priority shows pulsing indicator
4. Status change updates in real-time via React Query cache update
5. Complete tasks remain visible for 24 hours then archived
6. Task count badges update on all navigation items

---

## Edge Cases

1. **No tasks assigned** — Show empty state with "All caught up!"
2. **Many overdue tasks** — Show warning banner
3. **Blocked status** — Require reason notes when setting blocked
4. **Bulk status update** — Multi-select with bulk actions

---

## Tests

### task-queue.test.tsx

- My Tasks view shows only current user's tasks
- Team Tasks view shows all tasks
- Status dropdown updates task status
- Overdue indicator shows correct message
- Priority badge displays correct color
- Filter by status updates list

---

## Verification

```bash
npm run test:unit -- task-queue.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 1: Manager reassigns overdue payroll
