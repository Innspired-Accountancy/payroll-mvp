# Slice d: Widgets (Deadlines and Exceptions)

**Story:** story-02-dashboard-ui
**Epic:** epic-04-bureau-operations
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Create the sidebar widgets for upcoming deadlines and recent exceptions. These provide at-a-glance visibility into urgent items without requiring full table navigation.

---

## Decision Checklist

- [x] All libraries/packages named: React 18, @tanstack/react-query 5.x, date-fns 3.x, Tailwind CSS 3.4
- [x] All SDK methods/API calls identified: tRPC dashboard.deadlines, tRPC dashboard.exceptions
- [x] All external service endpoints specified: tRPC dashboard.deadlines, tRPC dashboard.exceptions
- [x] All data contracts defined: UpcomingDeadline, ExceptionSummary from story-01
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Widget loading, empty states
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 4: Deadline Calendar Management
- 02-04-bureau-operations-spec.md § API Contracts — Upcoming deadlines output

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/dashboard/widgets-section.tsx` | create | Widget container layout |
| `src/components/dashboard/deadlines-widget.tsx` | create | Upcoming deadlines widget |
| `src/components/dashboard/exceptions-widget.tsx` | create | Recent exceptions widget |
| `src/components/dashboard/deadline-item.tsx` | create | Single deadline row component |
| `src/components/dashboard/exception-item.tsx` | create | Single exception row component |
| `src/components/dashboard/severity-badge.tsx` | create | Exception severity indicator |
| `src/tests/components/dashboard/widgets.test.tsx` | create | Widget component tests |

---

## Responsibilities

1. Create deadlines widget showing next 5 upcoming deadlines with urgency
2. Create exceptions widget showing latest 5 critical/high exceptions
3. Color-code items by urgency/severity
4. Add click navigation to relevant client/payroll
5. Show "View All" links to full pages
6. Auto-refresh every 60 seconds

---

## Contracts

### DeadlinesWidget Component

```typescript
// src/components/dashboard/deadlines-widget.tsx
interface DeadlinesWidgetProps {
  bureauId: string;
  maxItems?: number; // default: 5
}

// Uses: api.dashboard.deadlines.useQuery({ days: 14 })
export function DeadlinesWidget({ bureauId, maxItems }: DeadlinesWidgetProps): JSX.Element;
```

### DeadlineItem Component

```typescript
// src/components/dashboard/deadline-item.tsx
interface DeadlineItemProps {
  deadline: {
    employerId: string;
    employerName: string;
    deadlineType: 'cut_off' | 'pay_date' | 'filing_deadline';
    deadlineDate: string;
    daysRemaining: number;
    status: 'on_track' | 'attention_needed' | 'overdue';
  };
}

// Color coding:
// on_track: green
// attention_needed: amber
// overdue: red
export function DeadlineItem({ deadline }: DeadlineItemProps): JSX.Element;
```

### ExceptionsWidget Component

```typescript
// src/components/dashboard/exceptions-widget.tsx
interface ExceptionsWidgetProps {
  bureauId: string;
  maxItems?: number; // default: 5
}

// Uses: api.dashboard.exceptions.useQuery({ limit: 5 })
export function ExceptionsWidget({ bureauId, maxItems }: ExceptionsWidgetProps): JSX.Element;
```

### ExceptionItem Component

```typescript
// src/components/dashboard/exception-item.tsx
interface ExceptionItemProps {
  exception: {
    id: string;
    type: string;
    severity: 'low' | 'medium' | 'high' | 'critical';
    employerName: string;
    description: string;
    createdAt: string;
  };
}

// Severity colors:
// low: gray
// medium: blue
// high: amber
// critical: red
export function ExceptionItem({ exception }: ExceptionItemProps): JSX.Element;
```

### SeverityBadge Component

```typescript
// src/components/dashboard/severity-badge.tsx
interface SeverityBadgeProps {
  severity: 'low' | 'medium' | 'high' | 'critical';
  showLabel?: boolean;
}

export function SeverityBadge({ severity, showLabel }: SeverityBadgeProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Deadlines sorted by date (soonest first)
2. Exceptions sorted by createdAt desc (newest first), severity desc
3. Critical severity exceptions show alert icon
4. Overdue deadlines show in red with negative days
5. Clicking item navigates to relevant client detail
6. Widgets show skeleton loading state

---

## Edge Cases

1. **No upcoming deadlines** — Show "No upcoming deadlines" message
2. **No active exceptions** — Show "No active exceptions" with checkmark
3. **Deadline today** — Show "Today" label instead of "0 days"
4. **Many exceptions** — Truncate list with "+X more" link

---

## Tests

### widgets.test.tsx

- Deadlines widget renders list of upcoming deadlines
- Exceptions widget renders list of recent exceptions
- Critical exceptions show alert styling
- Clicking item navigates to correct page
- Empty states display correctly
- Auto-refresh updates data every 60 seconds

---

## Verification

```bash
npm run test:unit -- widgets.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 4: Deadline Calendar Management
- 02-04-bureau-operations-spec.md § API Contracts → Deadlines and exceptions output
