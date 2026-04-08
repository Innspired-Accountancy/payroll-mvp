# Slice a: SSP Calculation Engine

**Story:** story-05-statutory-payments
**Epic:** epic-01-core-payroll
**Effort:** L
**Dependencies:** None

---

## Goal

Implement Statutory Sick Pay calculation with PIW (Period of Incapacity for Work), qualifying days, waiting days, and linked PIW handling.

---

## Decision Checklist

- [x] Libraries: date-fns 3.x for date arithmetic
- [x] Data: SSP weekly rate from config (£116.75 for 2026-27)
- [x] Logic: PIW detection, qualifying days, waiting days
- [x] Linked PIW: 8-week linking window
- [x] Error handling: Invalid dates, AWE below LEL
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC E14 — SSP employer guide
- 02-01-core-payroll-spec.md — Statutory payments

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/ssp.ts` | create | SSP calculator |
| `src/lib/types/statutory-pay.ts` | create | Statutory payment types |
| `src/lib/db/schema/sick-leave.ts` | create | SSP record storage |
| `src/tests/calculations/ssp.test.ts` | create | SSP calculation tests |

---

## Responsibilities

1. Determine PIW (4+ consecutive days of sickness)
2. Calculate qualifying days (days normally worked)
3. Apply 3 waiting days before SSP payable
4. Handle linked PIWs (no re-qualifying within 8 weeks)
5. Calculate daily SSP rate based on qualifying days

---

## Contracts

### calculateSsp
- **Method:** `calculateSsp(input: SspInput): SspResult`
- **Input:** SspInput with sickness details
- **Output:** SspResult with SSP amount and days

### SspInput
| Field | Type | Description |
|-------|------|-------------|
| sickness_start | Date | First day of sickness |
| sickness_end | Date | Last day of sickness (or null if ongoing) |
| qualifying_days | string[] | Days of week normally worked ["Mon", "Tue"...] |
| previous_piw_end | Date | End date of previous PIW (for linking) |
| awe | Decimal | Average Weekly Earnings |
| tax_year | string | Tax year for rates |

### SspResult
| Field | Type | Description |
|-------|------|-------------|
| ssp_payable | boolean | Whether SSP is payable |
| ssp_amount | Decimal | Total SSP for period |
| ssp_days | int | Number of SSP days paid |
| waiting_days | int | Number of waiting days applied |
| daily_rate | Decimal | Daily SSP rate |
| rejection_reason | string | Reason if not payable |

---

## Business Rules & Invariants

1. PIW = 4 or more consecutive days of sickness
2. SSP only payable from day 4 (3 waiting days)
3. Linked PIW: within 8 weeks, no new waiting days
4. AWE must be >= LEL (£125/week for 2026-27) to qualify
5. Max SSP: 28 weeks per PIW
6. Daily rate = weekly rate ÷ qualifying days per week

---

## Edge Cases

1. **Linked PIW** — No waiting days, continue from previous
2. **AWE below LEL** — SSP not payable, record reason
3. **PIW spans tax year** — Continue into new tax year
4. **Ongoing sickness** — Calculate to period end date
5. **Max 28 weeks reached** — SSP stops

---

## Tests

### ssp.test.ts
- Standard 5-day sickness (2 waiting, 2 payable)
- Linked PIW within 8 weeks
- AWE below LEL threshold
- PIW exactly 4 days (1 payable day)
- Ongoing sickness (partial period)
- Week 53 handling
- Max 28 weeks reached

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC E14 § SSP calculation
- 02-01-core-payroll-spec.md § Statutory Payments
