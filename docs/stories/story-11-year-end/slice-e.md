# Slice e: Year-End Reconciliation Reports

**Story:** story-11-year-end
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Generate year-end reconciliation reports comparing expected vs actual totals and identifying discrepancies.

---

## Decision Checklist

- [x] Reports: Summary, variance, exception reports
- [x] Comparison: Expected vs actual totals
- [x] Discrepancies: Flag and explain differences
- [x] Export: CSV and PDF formats
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Reconciliation reports
- 02-04-bureau-operations-spec.md — Bureau reporting

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/reports/year-end-reconciliation.ts` | create | Report generator |
| `src/server/routers/year-end-reports.ts` | create | tRPC endpoints |
| `src/components/reports/year-end-summary.tsx` | create | Report UI |
| `src/tests/year-end-reports.test.ts` | create | Report tests |

---

## Responsibilities

1. Generate reconciliation summary
2. Compare expected vs actual
3. Identify discrepancies
4. Export reports

---

## Contracts

### generateReconciliationReport
- **Method:** `generateReconciliationReport(taxYear: string, employerId: uuid): ReconciliationReport`
- **Input:** Tax year and employer
- **Output:** Reconciliation report

### ReconciliationReport
| Field | Type | Description |
|-------|------|-------------|
| tax_year | string | Tax year |
| expected_totals | Totals | From projections |
| actual_totals | Totals | From payslips |
| variances | Variance[] | Differences |
| discrepancies | Discrepancy[] | Items to review |

### yearEndReports.reconciliation
- **Method:** tRPC query `yearEndReports.reconciliation`
- **Input:** `{ tax_year: string }`
- **Output:** ReconciliationReport

---

## Business Rules & Invariants

1. Compare FPS totals to P60 totals (must match)
2. Flag employees with no P60 but in FPS
3. Flag P60s with no FPS record
4. Tax and NIC totals must reconcile

---

## Edge Cases

1. **No expected data** — Use only actuals
2. **Large variances** — Flag for investigation
3. **Partial year** — Note in report

---

## Tests

### year-end-reports.test.ts
- Generate reconciliation
- Identify discrepancies
- Export formats
- Variance calculation

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Year End Reconciliation
