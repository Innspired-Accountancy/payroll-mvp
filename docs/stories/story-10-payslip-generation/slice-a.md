# Slice a: Payslip Data Aggregation

**Story:** story-10-payslip-generation
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** None

---

## Goal

Aggregate all payslip data from calculation results including earnings, deductions, YTD totals, and metadata.

---

## Decision Checklist

- [x] Aggregation: Sum from calculation results
- [x] YTD: Cumulative from previous payslips
- [x] Elements: Earnings and deductions breakdown
- [x] Storage: Payslip table with all fields
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Payslip entity
- 02-01-core-payroll-spec.md — PayElement entity

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/payslip-aggregation.ts` | create | Data aggregator |
| `src/lib/db/schema/payslips.ts` | update | Payslip schema |
| `src/tests/payslip-aggregation.test.ts` | create | Aggregation tests |

---

## Responsibilities

1. Aggregate calculation results
2. Calculate YTD from history
3. Create pay elements (earnings/deductions)
4. Store complete payslip record

---

## Contracts

### aggregatePayslipData
- **Method:** `aggregatePayslipData(input: AggregationInput): PayslipData`
- **Input:** Calculation results and employee data
- **Output:** Complete payslip data

### AggregationInput
| Field | Type | Description |
|-------|------|-------------|
| pay_run_id | uuid | FK to pay run |
| employee_id | uuid | FK to employee |
| calculation_result | CalculationResult | From orchestrator |
| previous_ytd | YtdTotals | From last payslip |

### PayslipData
| Field | Type | Description |
|-------|------|-------------|
| gross_pay | Decimal | Total gross |
| taxable_pay | Decimal | Subject to tax |
| tax_deducted | Decimal | PAYE |
| nic_employee | Decimal | Employee NIC |
| nic_employer | Decimal | Employer NIC |
| net_pay | Decimal | Final net |
| ytd_totals | YtdTotals | Cumulative |
| elements | PayElement[] | Breakdown |

---

## Business Rules & Invariants

1. YTD = previous YTD + current period
2. Elements sum to gross and deductions
3. Taxable pay = gross - tax-free
4. Net = gross - deductions

---

## Edge Cases

1. **First payslip** — YTD = current period
2. **New starter** — Pro-rata YTD
3. **Leaver** — Final payslip, complete YTD

---

## Tests

### payslip-aggregation.test.ts
- Aggregate standard payslip
- First payslip YTD
- YTD accumulation
- Element summation

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Payslip Data Model
