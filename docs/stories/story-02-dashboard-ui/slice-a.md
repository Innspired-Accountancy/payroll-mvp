# Slice a: Dashboard Layout and Summary Cards

**Story:** story-02-dashboard-ui
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** story-01-dashboard-backend

---

## Goal

Create the main dashboard layout framework with summary cards displaying key bureau metrics. Implement the page structure, loading states, and error boundaries for the bureau operations dashboard.

---

## Decision Checklist

- [x] All libraries/packages named: Next.js 14 App Router, React 18, Tailwind CSS 3.4, @tanstack/react-query 5.x, Recharts 2.x
- [x] All SDK methods/API calls identified: tRPC dashboard.summary query, React Query useQuery
- [x] All external service endpoints specified: tRPC dashboard.summary
- [x] All data contracts defined: DashboardSummaryOutput from story-01
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Loading, error, empty states
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 1: Multi-Client Dashboard Review
- 02-04-bureau-operations-spec.md § API Contracts — GET /api/v1/bureau/dashboard
- 08-architecture-and-patterns.md — Next.js App Router patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(bureau)/dashboard/page.tsx` | create | Main dashboard page component |
| `src/components/dashboard/dashboard-layout.tsx` | create | Layout wrapper with sidebar |
| `src/components/dashboard/summary-cards.tsx` | create | Summary metric cards component |
| `src/components/dashboard/status-breakdown-chart.tsx` | create | Status distribution chart |
| `src/components/dashboard/dashboard-skeleton.tsx` | create | Loading skeleton |
| `src/components/dashboard/dashboard-error.tsx` | create | Error boundary fallback |
| `src/tests/components/dashboard/summary-cards.test.tsx` | create | Component tests |

---

## Responsibilities

1. Create dashboard page layout with header and main content area
2. Implement summary cards for: total clients, this week's payrolls, overdue items, critical exceptions
3. Create status breakdown chart showing payroll state distribution
4. Handle loading states with skeleton UI
5. Implement error boundaries for graceful failure handling

---

## Contracts

### SummaryCards Component

```typescript
// src/components/dashboard/summary-cards.tsx
interface SummaryCardsProps {
  data: {
    totalClients: number;
    payrollsThisWeek: number;
    overdueItems: number;
    criticalExceptions: number;
  };
  isLoading?: boolean;
}

export function SummaryCards({ data, isLoading }: SummaryCardsProps): JSX.Element;
```

### StatusBreakdownChart Component

```typescript
// src/components/dashboard/status-breakdown-chart.tsx
interface StatusBreakdownChartProps {
  data: {
    notStarted: number;
    dataCollection: number;
    calculated: number;
    inReview: number;
    approved: number;
    submitted: number;
    complete: number;
  };
}

export function StatusBreakdownChart({ data }: StatusBreakdownChartProps): JSX.Element;
```

### tRPC Query Usage

```typescript
// In page component
const { data, isLoading, error } = api.dashboard.summary.useQuery();
```

---

## Business Rules & Invariants

1. Summary cards display numbers with thousands separators
2. Critical exceptions card shows red background/indicator when count > 0
3. Overdue items card shows amber background when count > 0
4. Status breakdown chart uses consistent colors for each status
5. Data refreshes automatically every 30 seconds via React Query

---

## Edge Cases

1. **All counts are zero** — Display empty state with helpful message
2. **API error** — Show error card with retry button
3. **Slow loading** — Show skeleton with animated pulse
4. **Large numbers** — Format with K/M suffix for thousands/millions

---

## Tests

### summary-cards.test.tsx

- Renders all four summary cards with correct labels
- Displays numbers with proper formatting
- Shows loading state skeletons
- Critical exceptions card shows alert styling when count > 0

### status-breakdown-chart.test.tsx

- Renders chart with correct data
- Displays legend with status labels
- Handles empty data gracefully

---

## Verification

```bash
npm run test:unit -- summary-cards.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 1: Multi-Client Dashboard Review
- 02-04-bureau-operations-spec.md § API Contracts → Dashboard output structure
