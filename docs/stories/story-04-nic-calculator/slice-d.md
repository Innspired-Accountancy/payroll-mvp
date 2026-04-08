# Slice d: Director Annual Calculation

**Story:** story-04-nic-calculator
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement director NIC calculation using the annual method with pro-rata thresholds for appointment/cessation and election handling.

---

## Decision Checklist

- [x] Algorithm: HMRC director annual calculation (CWG2)
- [x] Data types: Decimal.js, annual threshold values
- [x] Logic: Pro-rata based on appointment date
- [x] Election: Alternative method flag support
- [x] Error handling: Invalid dates, YTD tracking
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — Director NIC calculation
- 02-01-core-payroll-spec.md — Director handling

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/nic-director.ts` | create | Director NIC calculator |
| `src/lib/types/director.ts` | create | Director types |
| `src/tests/calculations/nic-director.test.ts` | create | Director NIC tests |

---

## Responsibilities

1. Calculate director NIC using annual thresholds
2. Apply pro-rata for appointment/cessation mid-year
3. Support annual earnings period method
4. Handle director election (alternative method)
5. Accumulate NIC throughout year

---

## Contracts

### calculateDirectorNic
- **Method:** `calculateDirectorNic(input: DirectorNicInput): NicCalculationResult`
- **Input:** DirectorNicInput with YTD and appointment info
- **Output:** NicCalculationResult with director NIC

### DirectorNicInput
| Field | Type | Description |
|-------|------|-------------|
| gross_pay_ytd | Decimal | Year-to-date gross |
| nic_paid_ytd | Decimal | Year-to-date NIC paid |
| appointment_date | Date | Director appointment date |
| cessation_date | Date | Director cessation date (optional) |
| use_alternative_method | boolean | Election for alternative method |
| category | enum | NIC category |
| current_period_gross | Decimal | This period's gross |

---

## Business Rules & Invariants

1. Directors use annual thresholds regardless of pay frequency
2. Pro-rata thresholds if appointed mid-year
3. Recalculate NIC each period based on cumulative earnings
4. Alternative method: calculate NIC per period, reconcile at year-end
5. Regular method: cumulative calculation each period

---

## Edge Cases

1. **Appointed mid-year** — Pro-rata annual thresholds
2. **Ceased mid-year** — Final calculation with actual periods
3. **Irregular payments** — Cumulative calculation handles this
4. **Category change** — Recalculate from appointment

---

## Tests

### nic-director.test.ts
- Director annual calculation (regular method)
- Director appointed mid-year (pro-rata)
- Director alternative method (per-period)
- Year-end reconciliation
- Cessation calculation

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § Directors
- 02-01-core-payroll-spec.md § NIC Calculation
