# Slice c: Real-Time Status Update Events

**Story:** story-01-dashboard-backend
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement the event-driven update mechanism that keeps dashboard data current. Subscribe to payroll events (calculations, submissions, approvals) and update ClientPayrollStatus records accordingly. Refresh materialized views and notify connected clients of changes.

---

## Decision Checklist

- [x] All libraries/packages named: Next.js 14 App Router, Server-Sent Events (EventSource), Drizzle ORM 0.30.x
- [x] All SDK methods/API calls identified: EventEmitter, fetch EventSource, Drizzle transaction
- [x] All external service endpoints specified: Internal event bus, SSE endpoint at /api/sse/dashboard
- [x] All data contracts defined: DashboardEvent types, StatusChangePayload
- [x] All configuration/environment variables listed: EVENT_BUS_ENABLED (feature flag)
- [x] All error scenarios identified with handling strategy: Event delivery failures, stale data detection
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Non-Functional Requirements — Real-Time Updates
- 08-architecture-and-patterns.md — Event-driven architecture

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/events/dashboard-events.ts` | create | Event definitions and payload types |
| `src/lib/services/status-updater.ts` | create | Service to update status from events |
| `src/app/api/sse/dashboard/route.ts` | create | Server-Sent Events endpoint for dashboard |
| `src/lib/hooks/use-dashboard-events.ts` | create | React hook for consuming SSE |
| `src/tests/events/status-updater.test.ts` | create | Unit tests for status update logic |

---

## Responsibilities

1. Define dashboard event types and payload schemas
2. Implement event handlers that update ClientPayrollStatus records
3. Create SSE endpoint for real-time client notifications
4. Implement materialized view refresh trigger
5. Handle event delivery failures with retry logic

---

## Contracts

### DashboardEvent Types

```typescript
// src/lib/events/dashboard-events.ts
export type DashboardEventType = 
  | 'payroll.status_changed'
  | 'payroll.calculated'
  | 'payroll.approved'
  | 'payroll.submitted'
  | 'exception.created'
  | 'exception.resolved'
  | 'task.assigned'
  | 'deadline.approaching';

export interface PayrollStatusChangedEvent {
  type: 'payroll.status_changed';
  payload: {
    employerId: string;
    bureauId: string;
    previousStatus: PayrollStatus;
    newStatus: PayrollStatus;
    changedBy: string;
    changedAt: Date;
    payPeriodId: string;
  };
}

export interface ExceptionCreatedEvent {
  type: 'exception.created';
  payload: {
    exceptionId: string;
    employerId: string;
    bureauId: string;
    severity: 'low' | 'medium' | 'high' | 'critical';
    type: string;
    title: string;
  };
}

export type DashboardEvent = PayrollStatusChangedEvent | ExceptionCreatedEvent | /* ... */;
```

### SSE Endpoint

- **Path:** `/api/sse/dashboard`
- **Method:** GET
- **Headers:** Authorization: Bearer <token>
- **Response:** text/event-stream

```typescript
// Event format
event: payroll.status_changed
data: {"employerId": "...", "newStatus": "calculated", ...}

event: exception.created
data: {"exceptionId": "...", "severity": "critical", ...}
```

- **Auth:** Validates JWT and bureau membership
- **Errors:** 401 Unauthorized, 403 Forbidden

### StatusUpdater Service

```typescript
// src/lib/services/status-updater.ts
export class StatusUpdater {
  async handlePayrollCalculated(event: PayrollCalculatedEvent): Promise<void>;
  async handlePayrollApproved(event: PayrollApprovedEvent): Promise<void>;
  async handlePayrollSubmitted(event: PayrollSubmittedEvent): Promise<void>;
  async handleExceptionCreated(event: ExceptionCreatedEvent): Promise<void>;
  async handleExceptionResolved(event: ExceptionResolvedEvent): Promise<void>;
  private async refreshMaterializedView(bureauId: string): Promise<void>;
}
```

### React Hook

```typescript
// src/lib/hooks/use-dashboard-events.ts
export function useDashboardEvents(bureauId: string) {
  const [events, setEvents] = useState<DashboardEvent[]>([]);
  const [connected, setConnected] = useState(false);
  
  useEffect(() => {
    const eventSource = new EventSource(`/api/sse/dashboard?bureauId=${bureauId}`);
    // ... connection handling
    return () => eventSource.close();
  }, [bureauId]);
  
  return { events, connected };
}
```

---

## Business Rules & Invariants

1. Events are processed idempotently (duplicate events have no effect)
2. Materialized view refreshes are debounced (max once per 5 seconds per bureau)
3. SSE connections are authenticated and scoped to user's bureau
4. Event handlers run in transactions (all-or-nothing updates)
5. Failed event processing is retried 3 times before logging error

---

## Edge Cases

1. **SSE connection drops** — Client auto-reconnects with exponential backoff
2. **Missed events during disconnect** — Client requests full refresh on reconnect
3. **Event storm from bulk operation** — Debounce view refresh, batch notifications
4. **Multiple concurrent updates** — Optimistic locking prevents lost updates

---

## Tests

### status-updater.test.ts

- handlePayrollCalculated updates status to 'calculated'
- handleExceptionCreated sets has_exceptions = true
- handleExceptionResolved sets has_exceptions = false when no other exceptions
- Materialized view refresh debounces correctly
- Duplicate event processing is idempotent
- Transaction rollback on update failure

### dashboard-events.integration.test.ts

- SSE endpoint authenticates valid tokens
- SSE endpoint rejects invalid tokens
- Events are broadcast to connected clients
- Reconnection receives missed events

---

## Verification

```bash
npm run test:unit -- status-updater.test.ts
npm run test:integration -- dashboard-events.integration.test.ts
npm run typecheck
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Non-Functional Requirements → Real-Time Updates
- 02-04-bureau-operations-spec.md § Integration Points → Events
