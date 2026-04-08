# Slice b: Client Status Table with Sorting and Pagination

**Story:** story-02-dashboard-ui
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement the client status table that displays all bureau clients with their current payroll status, deadlines, and assignments. Support column sorting, pagination, and row-level actions for quick client navigation.

---

## Decision Checklist

- [x] All libraries/packages named: @tanstack/react-table 8.x, @tanstack/react-query 5.x, Tailwind CSS 3.4
- [x] All SDK methods/API calls identified: tRPC dashboard.clients query, React Table hooks
- [x] All external service endpoints specified: tRPC dashboard.clients with pagination
- [x] All data contracts defined: ClientListItem, ClientListOutput from story-01
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Pagination edge cases, sort errors
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § API Contracts — GET /api/v1/bureau/clients
- 02-04-bureau-operations-spec.md § Data Models — ClientPayrollStatus

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/dashboard/client-status-table.tsx` | create | Main table component |
| `src/components/dashboard/client-table-columns.tsx` | create | Column definitions |
| `src/components/dashboard/pagination-controls.tsx` | create | Pagination UI component |
| `src/components/dashboard/status-badge.tsx` | create | Status indicator badge |
| `src/components/dashboard/urgency-indicator.tsx` | create | Overdue/urgency visual |
| `src/hooks/use-client-table.ts` | create | Table state management hook |
| `src/tests/components/dashboard/client-status-table.test.tsx` | create | Component tests |

---

## Responsibilities

1. Display client data in sortable table with columns: Name, Status, Pay Date, Cut-off, Assigned, Actions
2. Implement client-side sorting with visual indicators
3. Implement pagination with page size selector (25, 50, 100)
4. Show status badges with color coding
5. Display urgency indicators for overdue items
6. Provide row click to navigate to client detail

---

## Contracts

### ClientStatusTable Component

```typescript
// src/components/dashboard/client-status-table.tsx
interface ClientStatusTableProps {
  filters: {
    status?: PayrollStatus;
    assignedTo?: string;
    overdue?: boolean;
    search?: string;
  };
}

export function ClientStatusTable({ filters }: ClientStatusTableProps): JSX.Element;
```

### useClientTable Hook

```typescript
// src/hooks/use-client-table.ts
interface UseClientTableReturn {
  data: ClientListItem[];
  totalPages: number;
  currentPage: number;
  pageSize: number;
  sortColumn: string;
  sortDirection: 'asc' | 'desc';
  setPage: (page: number) => void;
  setPageSize: (size: number) => void;
  setSort: (column: string, direction: 'asc' | 'desc') => void;
  isLoading: boolean;
  error: Error | null;
}

export function useClientTable(filters: ClientFilters): UseClientTableReturn;
```

### StatusBadge Component

```typescript
// src/components/dashboard/status-badge.tsx
interface StatusBadgeProps {
  status: PayrollStatus;
  daysOverdue?: number;
}

// Status colors:
// not_started: gray
// data_collection: blue
// calculated: yellow
// in_review: purple
// approved: indigo
// submitted: cyan
// paid: teal
// complete: green
export function StatusBadge({ status, daysOverdue }: StatusBadgeProps): JSX.Element;
```

### tRPC Query Usage

```typescript
const { data, isLoading } = api.dashboard.clients.useQuery({
  status: filters.status,
  assignedTo: filters.assignedTo,
  overdue: filters.overdue,
  search: filters.search,
  sortBy: sortColumn,
  sortOrder: sortDirection,
  page: currentPage,
  pageSize: pageSize,
});
```

---

## Business Rules & Invariants

1. Table defaults to sorting by lastActivity desc (most recent first)
2. Overdue clients show amber/red urgency indicator
3. Clients with exceptions show warning icon
4. Clicking a row navigates to /clients/[employerId]
5. Page size preference is stored in localStorage

---

## Edge Cases

1. **Empty results** — Show empty state with "No clients match your filters"
2. **Single page** — Hide pagination controls
3. **Last page with few items** — Display correctly without extra empty rows
4. **Rapid sort clicks** — Debounce to prevent API spam

---

## Tests

### client-status-table.test.tsx

- Renders table with correct columns
- Sorting triggers API call with correct params
- Pagination controls work correctly
- Overdue clients show urgency indicator
- Row click navigates to client detail
- Empty state displays when no results

---

## Verification

```bash
npm run test:unit -- client-status-table.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § API Contracts → Client list output
- 02-04-bureau-operations-spec.md § User Journeys → Multi-Client Dashboard Review
