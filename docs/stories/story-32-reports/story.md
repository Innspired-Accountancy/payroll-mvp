# Story: Employer Cost Reports and Summaries

**Epic:** epic-07-employer-portal
**Priority:** 7 of 8
**Sprint:** Sprint 5
**Dependencies:** story-26-client-auth, story-04-task-tracking

## Goal

Provide self-service reporting for employers to view payroll costs, departmental breakdowns, variance analysis, and historical trends. Reports must be employer-scoped and available as both online views and downloadable PDF/Excel.

## Acceptance Criteria

- [ ] Payroll summary report (gross, deductions, net by period)
- [ ] Department/location cost breakdown
- [ ] Employee cost summary report
- [ ] Variance report (period-over-period comparison)
- [ ] Tax and NI summary report
- [ ] Pension contributions report
- [ ] Date range selection for all reports
- [ ] Online report viewing with charts
- [ ] PDF download option
- [ ] Excel/CSV export option
- [ ] Report generation under 10 seconds
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Slow report generation for large employers | Medium | High | Query optimization; materialized views; caching |
| Data leakage in multi-employer reports | Low | Critical | Strict employer_id filtering; parameterized queries |
| Report timeouts on complex queries | Medium | High | Query limits; background generation; pagination |
| Incorrect totals due to aggregation errors | Medium | High | Automated reconciliation; test data validation |
| Export format compatibility issues | Low | Low | Standard formats; testing with common tools |
| Historical data archival affecting reports | Medium | Medium | Archival strategy; report data retention policy |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Report API and data aggregation | M | story-26-client-auth |
| slice-b | Report UI with charts | M | slice-a |
| slice-c | PDF and Excel export | M | slice-b |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Report API with aggregation logic
- [slice-b.md](./slice-b.md) — Report UI with visualization
- [slice-c.md](./slice-c.md) — Export functionality (PDF/Excel)
