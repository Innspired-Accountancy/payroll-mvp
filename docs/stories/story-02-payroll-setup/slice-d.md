# Slice d: Payroll Calendar UI

**Story:** story-02-payroll-setup
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a, slice-b

---

## Goal

Create the payroll calendar visualization UI showing all pay periods for a tax year with status indicators and navigation.

---

## Decision Checklist

- [x] Libraries: React 18.x, Tailwind CSS 3.x, date-fns 3.x
- [x] Components: CalendarGrid, PeriodCard, ScheduleSelector
- [x] Data fetching: tRPC payPeriods.list query
- [x] Error handling: ErrorBoundary, loading states
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — PayPeriod status values
- 02-06-employer-portal-spec.md — Bureau UI patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(bureau)/payroll/calendar/page.tsx` | create | Calendar page |
| `src/components/payroll/calendar-grid.tsx` | create | Calendar grid component |
| `src/components/payroll/period-card.tsx` | create | Period card component |
| `src/components/payroll/schedule-selector.tsx` | create | Schedule dropdown |
| `src/components/payroll/period-status-badge.tsx` | create | Status indicator |

---

## Responsibilities

1. Display tax year calendar with all periods
2. Show period status (draft/calculated/approved/finalised)
3. Allow navigation to period detail
4. Support schedule switching
5. Highlight current period

---

## Contracts

### CalendarGrid Props
| Prop | Type | Description |
|------|------|-------------|
| periods | PayPeriod[] | Array of periods to display |
| taxYear | string | Tax year being displayed |
| onPeriodClick | (id: string) => void | Click handler |

### PeriodCard Props
| Prop | Type | Description |
|------|------|-------------|
| period | PayPeriod | Period data |
| isCurrent | boolean | Highlight if current period |
| onClick | () => void | Click handler |

---

## Business Rules & Invariants

1. Periods displayed in chronological order
2. Current period highlighted based on today's date
3. Status color coding: draft=gray, calculated=yellow, approved=green, finalised=blue
4. Clicking period navigates to pay run detail

---

## Edge Cases

1. **Many periods** — Virtual scrolling or pagination
2. **No periods generated** — Show empty state with generate button
3. **Multiple schedules** — Schedule selector dropdown

---

## Tests

### calendar-grid.test.tsx
- Render with periods
- Handle period click
- Display correct status colors
- Show empty state

---

## Verification

```bash
npm run test:unit
npm run typecheck
npm run build
```

---

## Source Sections

- 02-01-core-payroll-spec.md § User Journeys
