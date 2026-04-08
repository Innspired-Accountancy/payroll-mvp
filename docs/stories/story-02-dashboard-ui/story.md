# Story: Dashboard UI - Dashboard Views and Filters

**Epic:** epic-04-bureau-operations
**Priority:** 2 of 8
**Sprint:** Month 4
**Dependencies:** story-01-dashboard-backend

---

## Goal

Create the bureau dashboard UI with multiple views, filters, and interactive elements. Display client portfolio status, upcoming deadlines, exceptions queue summary, and team workload in an at-a-glance format that enables quick decision-making.

---

## Acceptance Criteria

- [ ] Summary cards display key metrics (total clients, this week's payrolls, overdue, critical exceptions)
- [ ] Status breakdown chart shows distribution across payroll states
- [ ] Client table displays sortable list with status, deadlines, and assignment
- [ ] Filter panel supports filtering by status, processor, date range, and overdue flag
- [ ] Upcoming deadlines widget shows urgent items with color-coded urgency
- [ ] Recent exceptions widget displays latest critical/high severity items
- [ ] Dashboard updates in real-time (or near real-time) as status changes
- [ ] Responsive layout works on desktop and tablet
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| UI performance with large datasets | Medium | Medium | Virtualized tables, pagination, optimistic updates |
| Complex filter combinations causing slow queries | Medium | Medium | Debounced search, query optimization, client-side caching |
| Mobile/tablet layout issues | Medium | Low | Responsive design testing, progressive enhancement |
| Real-time update conflicts with user interactions | Low | Medium | Conflict resolution strategy, user notification of updates |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Dashboard layout and summary cards | M | story-01-dashboard-backend |
| slice-b | Client status table with sorting and pagination | M | slice-a |
| slice-c | Filter panel and search | M | slice-a |
| slice-d | Widgets (deadlines, exceptions) | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Dashboard layout framework and summary cards
- [slice-b.md](./slice-b.md) — Client status table with interactive features
- [slice-c.md](./slice-c.md) — Advanced filter panel and search
- [slice-d.md](./slice-d.md) — Upcoming deadlines and exceptions widgets
