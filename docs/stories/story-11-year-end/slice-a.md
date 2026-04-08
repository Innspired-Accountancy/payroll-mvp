# Slice a: Year End Data Aggregation

**Story:** story-11-year-end
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** None

---

## Goal

Aggregate year-end data for all employees including final YTD totals, statutory payments, and leaver status.

---

## Decision Checklist

- [x] Aggregation: Sum all payslips for tax year
- [x] Leavers: Exclude if P45 already issued
- [x] Statutory: SSP, SMP, SPP totals
- [x] Validation: Check for missing data
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Year-end processing
- HMRC guidance — P60 requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/year-end/aggregation.ts` | create | Year-end aggregator |
| `src/server/routers/year-end.ts` | create | Year-end tRPC |
| `src/tests/year-end-aggregation.test.ts` | create | Aggregation tests |

---

## Responsibilities

1. Aggregate all payslips for tax year
2. Calculate final YTD for each employee
3. Identify employees needing P60
4. Validate data completeness

---

## Contracts

### aggregateYearEndData
- **Method:** `aggregateYearEndData(taxYear: string, employerId: uuid): YearEndData`
- **Input:** Tax year and employer
- **Output:** Aggregated year-end data

### YearEndEmployeeData
| Field | Type | Description |
|-------|------|-------------|
| employee_id | uuid | Employee FK |
| tax_code | string | Final tax code |
| total_pay | Decimal | Total gross |
| total_tax | Decimal | Total tax deducted |
| total_nic | Decimal | Total employee NIC |
| ssp_received | Decimal | Total SSP |
| smp_received | Decimal | Total SMP |
| needs_p60 | boolean | P60 required |

---

## Business Rules & Invariants

1. Only employees with payslips in tax year
2. Exclude if P45 issued before April 5
3. Final tax code from last payslip
4. SSP/SMP shown if received > 0

---

## Edge Cases

1. **Mid-year employer** — Only since registration
2. **No payslips** — No P60 required
3. **Leaver rejoiner** — Separate employments

---

## Tests

### year-end-aggregation.test.ts
- Aggregate full tax year
- Exclude P45 issued leavers
- Calculate YTD correctly
- Handle mid-year employer

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Year End Processing
