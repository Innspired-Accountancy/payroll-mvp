# Slice c: Variance Reporting Engine

**Story:** story-08-pay-run-lifecycle
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement variance reporting comparing current pay run to previous period with amount and percentage changes.

---

## Decision Checklist

- [x] Comparison: Current vs previous pay run
- [x] Metrics: Gross, tax, NIC, net, headcount
- [x] Employee level: Individual variances
- [x] Thresholds: Flag variances above % threshold
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Variance reporting
- 02-04-bureau-operations-spec.md — Bureau review process

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/reports/variance.ts` | create | Variance calculator |
| `src/server/routers/variance-reports.ts` | create | tRPC endpoints |
| `src/components/reports/variance-report.tsx` | create | Variance UI |
| `src/tests/variance.test.ts` | create | Variance tests |

---

## Responsibilities

1. Compare current pay run to previous
2. Calculate absolute and percentage variances
3. Flag significant variances (>5% or £100)
4. Show new leavers/joiners
5. Generate variance report

---

## Contracts

### generateVarianceReport
- **Method:** tRPC query `varianceReports.generate`
- **Input:** `{ pay_run_id: uuid }`
- **Output:** VarianceReport

### VarianceReport
| Field | Type | Description |
|-------|------|-------------|
| current_period | PayPeriodRef | Current period |
| previous_period | PayPeriodRef | Previous period |
| summary | VarianceSummary | Totals variance |
| employee_variances | EmployeeVariance[] | Per-employee changes |
| new_employees | EmployeeRef[] | New this period |
| leavers | EmployeeRef[] | Left this period |

### VarianceSummary
| Field | Type | Description |
|-------|------|-------------|
| gross_change | Decimal | Gross pay change |
| gross_change_pct | Decimal | Percentage change |
| tax_change | Decimal | Tax change |
| nic_change | Decimal | NIC change |
| net_change | Decimal | Net pay change |
| headcount_change | int | Employee count change |

---

## Business Rules & Invariants

1. Compare to most recent finalised pay run
2. New employees show 100% increase (from zero)
3. Leavers show 100% decrease (to zero)
4. Flag variances > 5% or > £100 for review

---

## Edge Cases

1. **No previous period** — Show as new
2. **First pay run ever** — No variance possible
3. **Restated previous period** — Use original values

---

## Tests

### variance.test.ts
- Calculate variance between periods
- Flag significant variances
- Handle new employee
- Handle leaver
- No previous period handling

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Variance Reporting
- 02-04-bureau-operations-spec.md § Review Workflow
