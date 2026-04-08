# Story: Client Dashboard View

**Epic:** epic-07-employer-portal
**Priority:** 2 of 8
**Sprint:** Sprint 4
**Sprint:** Sprint 4
**Dependencies:** story-26-client-auth, story-02-dashboard-ui

## Goal

Provide a role-appropriate dashboard for client portal users showing employer-specific payroll status, pending actions, deadlines, and recent activity. The dashboard must be mobile-responsive and load within 2 seconds.

## Acceptance Criteria

- [ ] Dashboard displays employer name and PAYE reference
- [ ] Current payroll period status with visual indicator
- [ ] Days remaining until deadline with countdown
- [ ] Pending actions list (payroll approval, data submission required)
- [ ] Recent activity feed (payroll paid, submissions made)
- [ ] Role-based view differences (admin sees more actions than viewer)
- [ ] Mobile-responsive layout
- [ ] Dashboard data refreshes automatically or on manual refresh
- [ ] Loading states and error handling
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Slow dashboard load with many pending items | Medium | Medium | Pagination; lazy loading; caching |
| Data leakage between employers | Low | Critical | Strict employer_id filtering on all queries |
| Stale data showing incorrect deadlines | Medium | High | Real-time calculation; cache invalidation |
| Mobile layout issues on small screens | Medium | Low | Responsive testing; progressive disclosure |
| Notification timing delays | Medium | Medium | Event-driven updates; polling fallback |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Dashboard API and data aggregation | M | story-26-client-auth |
| slice-b | Dashboard UI components | M | slice-a |
| slice-c | Mobile responsiveness and polish | S | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Dashboard API with employer-scoped data
- [slice-b.md](./slice-b.md) — Dashboard React components
- [slice-c.md](./slice-c.md) — Mobile responsiveness and loading states
