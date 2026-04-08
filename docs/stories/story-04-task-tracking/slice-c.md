# Slice c: Automatic Task Creation Rules

**Story:** story-04-task-tracking
**Epic:** epic-04-bureau-operations
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Implement automatic task creation rules that generate tasks based on payroll events and deadlines. Reduce manual task creation by automatically creating tasks for recurring payroll activities.

---

## Decision Checklist

- [x] All libraries/packages named: Node.js EventEmitter, Drizzle ORM 0.30.x, date-fns 3.x
- [x] All SDK methods/API calls identified: EventEmitter.on(), taskService.create()
- [x] All external service endpoints specified: Internal event bus
- [x] All data contracts defined: PayrollEvent types, TaskRule configuration
- [x] All configuration/environment variables listed: AUTO_TASK_CREATION_ENABLED (default: true)
- [x] All error scenarios identified with handling strategy: Event parsing failures, duplicate prevention
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Integration Points — Core Payroll events
- 08-architecture-and-patterns.md — Event-driven architecture

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/rules/task-creation-rules.ts` | create | Rules for automatic task creation |
| `src/lib/services/auto-task-service.ts` | create | Service for creating tasks from events |
| `src/lib/events/task-event-handlers.ts` | create | Event handler implementations |
| `src/tests/services/auto-task-service.test.ts` | create | Auto-task service tests |

---

## Responsibilities

1. Define rules for automatic task creation from payroll events
2. Create tasks when pay period is created (data collection)
3. Create tasks when calculation is complete (review)
4. Create tasks when review is complete (approval)
5. Create tasks when approval is complete (submission)
6. Prevent duplicate task creation

---

## Contracts

### Task Creation Rules

```typescript
// src/lib/rules/task-creation-rules.ts
export interface TaskCreationRule {
  eventType: string;
  condition: (event: unknown) => boolean;
  taskTemplate: {
    taskType: TaskType;
    title: string;
    description: string;
    priority: TaskPriority;
    dueDateOffset: number; // days from event
  };
}

export const taskCreationRules: TaskCreationRule[] = [
  {
    eventType: 'payroll.period_created',
    condition: () => true,
    taskTemplate: {
      taskType: 'data_collection',
      title: 'Collect payroll data',
      description: 'Collect timesheets, variable pay, and other payroll data from client',
      priority: 'normal',
      dueDateOffset: -2, // 2 days before cut-off
    },
  },
  {
    eventType: 'payroll.calculated',
    condition: () => true,
    taskTemplate: {
      taskType: 'review',
      title: 'Review payroll calculation',
      description: 'Review calculated payroll for accuracy before approval',
      priority: 'high',
      dueDateOffset: 1, // 1 day after calculation
    },
  },
  {
    eventType: 'payroll.review_complete',
    condition: () => true,
    taskTemplate: {
      taskType: 'approval',
      title: 'Approve payroll',
      description: 'Final approval required before submission',
      priority: 'high',
      dueDateOffset: 0, // same day
    },
  },
  {
    eventType: 'payroll.approved',
    condition: () => true,
    taskTemplate: {
      taskType: 'submission',
      title: 'Submit to HMRC',
      description: 'Submit FPS to HMRC before filing deadline',
      priority: 'urgent',
      dueDateOffset: 0, // same day
    },
  },
];
```

### AutoTaskService

```typescript
// src/lib/services/auto-task-service.ts
export class AutoTaskService {
  async handlePayrollPeriodCreated(event: PayrollPeriodCreatedEvent): Promise<void>;
  async handlePayrollCalculated(event: PayrollCalculatedEvent): Promise<void>;
  async handlePayrollReviewComplete(event: PayrollReviewCompleteEvent): Promise<void>;
  async handlePayrollApproved(event: PayrollApprovedEvent): Promise<void>;
  
  private async createTaskFromRule(
    rule: TaskCreationRule,
    event: unknown
  ): Promise<void>;
  
  private async shouldCreateTask(
    employerId: string,
    payPeriodId: string,
    taskType: TaskType
  ): Promise<boolean>;
  
  private calculateDueDate(
    baseDate: Date,
    offsetDays: number,
    cutOffDate?: Date
  ): Date;
}
```

### Event Handler Mappings

```typescript
// src/lib/events/task-event-handlers.ts
export function registerTaskEventHandlers(eventBus: EventEmitter): void {
  eventBus.on('payroll.period_created', (event) => {
    autoTaskService.handlePayrollPeriodCreated(event);
  });
  
  eventBus.on('payroll.calculated', (event) => {
    autoTaskService.handlePayrollCalculated(event);
  });
  
  eventBus.on('payroll.review_complete', (event) => {
    autoTaskService.handlePayrollReviewComplete(event);
  });
  
  eventBus.on('payroll.approved', (event) => {
    autoTaskService.handlePayrollApproved(event);
  });
}
```

---

## Business Rules & Invariants

1. Only one task of each type per pay period (duplicate prevention)
2. Data collection task due date is 2 days before payroll cut-off
3. Review task created 1 day after calculation (allows buffer)
4. Approval and submission tasks are same-day priority
5. Tasks assigned to default processor for employer
6. Manual task creation takes precedence over auto-creation

---

## Edge Cases

1. **No default processor** — Assign to bureau manager
2. **Cut-off date not set** — Use pay date minus 5 days as fallback
3. **Weekend due dates** — Roll back to Friday
4. **Event replay** — Check for existing task before creating

---

## Tests

### auto-task-service.test.ts

- Period created event generates data collection task
- Calculated event generates review task
- Duplicate event doesn't create second task
- Due date calculates correctly from cut-off
- Weekend due date rolls to Friday
- No default processor assigns to manager

---

## Verification

```bash
npm run test:unit -- auto-task-service.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Integration Points → Core Payroll events
- 02-04-bureau-operations-spec.md § User Journeys → Payroll workflow
