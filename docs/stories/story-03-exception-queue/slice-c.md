# Slice c: Exception Queue UI with Filters

**Story:** story-03-exception-queue
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Create the exception queue interface allowing bureau staff to view, filter, and manage exceptions. Provide a clear overview of all active issues with visual indicators for severity and status.

---

## Decision Checklist

- [x] All libraries/packages named: @tanstack/react-table 8.x, @tanstack/react-query 5.x, date-fns 3.x, Tailwind CSS 3.4
- [x] All SDK methods/API calls identified: tRPC exceptions.list query
- [x] All external service endpoints specified: tRPC exceptions.list
- [x] All data contracts defined: ExceptionListOutput from slice-a
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Loading, error, empty states
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 2: Exception Queue Management
- 02-04-bureau-operations-spec.md § API Contracts — Exception filtering

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(bureau)/exceptions/page.tsx` | create | Exception queue page |
| `src/components/exceptions/exception-queue-table.tsx` | create | Main exception table |
| `src/components/exceptions/exception-filters.tsx` | create | Filter panel component |
| `src/components/exceptions/severity-icon.tsx` | create | Severity visual indicator |
| `src/components/exceptions/type-badge.tsx` | create | Exception type badge |
| `src/components/exceptions/status-badge.tsx` | create | Status badge component |
| `src/components/exceptions/exception-counts.tsx` | create | Summary counts by severity |
| `src/tests/components/exceptions/exception-queue.test.tsx` | create | Component tests |

---

## Responsibilities

1. Create exception queue page with table and filters
2. Display exceptions with severity, type, employer, assigned to, created date
3. Implement filters for status, type, severity, and assigned to
4. Show summary counts (critical, high, medium, low)
5. Support sorting by severity, date, status
6. Enable row click to view exception detail

---

## Contracts

### ExceptionQueueTable Component

```typescript
// src/components/exceptions/exception-queue-table.tsx
interface ExceptionQueueTableProps {
  filters: ExceptionFilters;
}

export interface ExceptionFilters {
  status?: ExceptionStatus;
  type?: ExceptionType;
  severity?: ExceptionSeverity;
  assignedTo?: string;
  employerId?: string;
}

// Uses: api.exceptions.list.useQuery(filters)
export function ExceptionQueueTable({ filters }: ExceptionQueueTableProps): JSX.Element;
```

### ExceptionFilters Component

```typescript
// src/components/exceptions/exception-filters.tsx
interface ExceptionFiltersProps {
  filters: ExceptionFilters;
  onChange: (filters: ExceptionFilters) => void;
}

export function ExceptionFilters({ filters, onChange }: ExceptionFiltersProps): JSX.Element;
```

### SeverityIcon Component

```typescript
// src/components/exceptions/severity-icon.tsx
interface SeverityIconProps {
  severity: 'low' | 'medium' | 'high' | 'critical';
  size?: 'sm' | 'md' | 'lg';
  showLabel?: boolean;
}

// Icons:
// critical: ExclamationCircle (red)
// high: ExclamationTriangle (amber)
// medium: InformationCircle (blue)
// low: InformationCircle (gray)
export function SeverityIcon({ severity, size, showLabel }: SeverityIconProps): JSX.Element;
```

### TypeBadge Component

```typescript
// src/components/exceptions/type-badge.tsx
interface TypeBadgeProps {
  type: 'hmrc_failure' | 'pension_failure' | 'payment_failure' | 'missing_data' | 'approval_pending' | 'validation_error';
}

// Type colors:
// hmrc_failure: red
// pension_failure: orange
// payment_failure: red
// missing_data: amber
// approval_pending: blue
// validation_error: purple
export function TypeBadge({ type }: TypeBadgeProps): JSX.Element;
```

### ExceptionCounts Component

```typescript
// src/components/exceptions/exception-counts.tsx
interface ExceptionCountsProps {
  counts: {
    critical: number;
    high: number;
    medium: number;
    low: number;
    total: number;
  };
}

export function ExceptionCounts({ counts }: ExceptionCountsProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Default sort is createdAt desc (newest first)
2. Critical severity items appear at top when sorting by severity
3. Resolved exceptions hidden by default (filter toggle to show)
4. Counts update in real-time as exceptions are created/resolved
5. Row highlighting for items assigned to current user
6. Quick filter buttons for "My Exceptions" and "Critical Only"

---

## Edge Cases

1. **Many exceptions** — Virtualized table for performance
2. **All resolved** — Show empty state with celebration message
3. **Filter returns no results** — Show clear filters button
4. **Long exception titles** — Truncate with tooltip

---

## Tests

### exception-queue.test.tsx

- Table renders with correct columns
- Severity icons display correctly for each level
- Filter by status shows only matching exceptions
- Filter by type shows only matching exceptions
- Sort by severity orders critical first
- Clicking row navigates to detail
- Exception counts display correctly

---

## Verification

```bash
npm run test:unit -- exception-queue.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 2: Exception Queue Management
- 02-04-bureau-operations-spec.md § API Contracts → Exception filtering parameters
