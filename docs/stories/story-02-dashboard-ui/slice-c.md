# Slice c: Filter Panel and Search

**Story:** story-02-dashboard-ui
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement the advanced filter panel for the dashboard allowing users to filter clients by status, assigned processor, overdue status, and free-text search. Filters apply in real-time with debounced inputs.

---

## Decision Checklist

- [x] All libraries/packages named: React 18, Radix UI Select 2.x, use-debounce 10.x, Tailwind CSS 3.4
- [x] All SDK methods/API calls identified: tRPC user list for processor dropdown, tRPC dashboard.clients
- [x] All external service endpoints specified: tRPC users.list for processor assignment
- [x] All data contracts defined: ClientFilters interface, User list output
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Validation errors, empty filter results
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 1: Filter by team member
- 08-architecture-and-patterns.md — Form state management with React Hook Form

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/dashboard/filter-panel.tsx` | create | Main filter panel component |
| `src/components/dashboard/status-filter.tsx` | create | Status multi-select filter |
| `src/components/dashboard/processor-filter.tsx` | create | Assigned processor dropdown |
| `src/components/dashboard/overdue-toggle.tsx` | create | Overdue only toggle |
| `src/components/dashboard/client-search.tsx` | create | Free-text search input |
| `src/components/dashboard/filter-badge.tsx` | create | Active filter indicator |
| `src/hooks/use-dashboard-filters.ts` | create | Filter state management |
| `src/tests/components/dashboard/filter-panel.test.tsx` | create | Component tests |

---

## Responsibilities

1. Create filter panel with collapsible sections
2. Implement status multi-select with checkboxes
3. Implement processor dropdown populated from user list
4. Add overdue-only toggle switch
5. Implement search input with debounced filtering
6. Show active filter count and clear-all button
7. Persist filter preferences to URL query params

---

## Contracts

### FilterPanel Component

```typescript
// src/components/dashboard/filter-panel.tsx
interface FilterPanelProps {
  filters: ClientFilters;
  onChange: (filters: ClientFilters) => void;
  bureauId: string;
}

export interface ClientFilters {
  status?: PayrollStatus[];
  assignedTo?: string[];
  overdue?: boolean;
  hasExceptions?: boolean;
  search?: string;
  dateRange?: { from?: Date; to?: Date };
}

export function FilterPanel({ filters, onChange, bureauId }: FilterPanelProps): JSX.Element;
```

### useDashboardFilters Hook

```typescript
// src/hooks/use-dashboard-filters.ts
interface UseDashboardFiltersReturn {
  filters: ClientFilters;
  setFilter: <K extends keyof ClientFilters>(key: K, value: ClientFilters[K]) => void;
  clearFilter: (key: keyof ClientFilters) => void;
  clearAll: () => void;
  activeCount: number;
  isDefault: boolean;
}

export function useDashboardFilters(): UseDashboardFiltersReturn;
```

### ProcessorFilter Component

```typescript
// src/components/dashboard/processor-filter.tsx
interface ProcessorFilterProps {
  bureauId: string;
  selected: string[];
  onChange: (processorIds: string[]) => void;
}

// Uses tRPC users.list with bureau filter:
// api.users.list.useQuery({ bureauId, role: 'processor' })
export function ProcessorFilter(props: ProcessorFilterProps): JSX.Element;
```

### SearchInput with Debounce

```typescript
// src/components/dashboard/client-search.tsx
interface ClientSearchProps {
  value: string;
  onChange: (value: string) => void;
  debounceMs?: number; // default: 300
}

// Uses use-debounce library:
// const debouncedValue = useDebounce(value, debounceMs);
export function ClientSearch(props: ClientSearchProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Search input debounces at 300ms to prevent excessive API calls
2. Multiple status filters are ORed together (show clients matching ANY selected status)
3. Multiple processor filters are ORed together
4. URL query params sync with filter state for shareable filtered views
5. Clear all button appears when any filter is active

---

## Edge Cases

1. **No processors in bureau** — Show empty state in processor dropdown
2. **All statuses selected** — Optimize query by not filtering on status
3. **Special characters in search** — Escape for SQL LIKE clause
4. **Long search term** — Truncate display, show full in tooltip

---

## Tests

### filter-panel.test.tsx

- Status checkboxes update filter state correctly
- Processor dropdown loads and displays users
- Search input debounces correctly
- URL params sync with filter state
- Clear all button resets all filters
- Active filter count displays correctly

---

## Verification

```bash
npm run test:unit -- filter-panel.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § API Contracts → Query parameters for filtering
- 02-04-bureau-operations-spec.md § Non-Functional Requirements → Filter and search <2 seconds
