# Slice d: Average Weekly Earnings Calculation

**Story:** story-05-statutory-payments
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a, slice-b, slice-c

---

## Goal

Implement AWE calculation for statutory payments using relevant period rules per HMRC guidelines.

---

## Decision Checklist

- [x] Rules: SSP 8 weeks, SMP/SPP 8 weeks ending qualifying week
- [x] Earnings: Gross pay subject to Class 1 NIC
- [x] Calculation: Total earnings ÷ number of weeks
- [x] Rounding: Round up to nearest penny
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC guidance — AWE calculation for each statutory payment
- 02-01-core-payroll-spec.md — Statutory payments

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/awe.ts` | create | AWE calculator |
| `src/tests/calculations/awe.test.ts` | create | AWE tests |

---

## Responsibilities

1. Determine relevant period for each payment type
2. Sum earnings subject to Class 1 NIC
3. Divide by number of weeks in period
4. Round up to nearest penny
5. Handle incomplete periods

---

## Contracts

### calculateAwe
- **Method:** `calculateAwe(input: AweInput): Decimal`
- **Input:** AweInput with earnings history
- **Output:** Average Weekly Earnings amount

### AweInput
| Field | Type | Description |
|-------|------|-------------|
| payment_type | enum | ssp, smp, spp |
| qualifying_date | Date | Reference date for calculation |
| earnings_history | EarningRecord[] | Weekly earnings for period |
| employment_start | Date | Employee start date |

### EarningRecord
| Field | Type | Description |
|-------|------|-------------|
| week_start | Date | Start of week |
| gross_pay | Decimal | Gross pay for week |
| nicable | boolean | Whether subject to Class 1 NIC |

---

## Business Rules & Invariants

1. SSP: 8 weeks before PIW starts
2. SMP: 8 weeks ending with qualifying week (week 17 before EWC)
3. SPP: 8 weeks ending with qualifying week
4. Only include earnings subject to Class 1 NIC
5. Divide by actual weeks (even if no earnings)
6. Round UP to nearest penny

---

## Edge Cases

1. **Employee started during period** — Use actual weeks employed
2. **No earnings in period** — AWE = 0, may not qualify
3. **Irregular pay periods** — Use weekly equivalent
4. **Sick pay in period** — Include if subject to NIC

---

## Tests

### awe.test.ts
- Standard 8-week AWE calculation
- Employee started mid-period
- No earnings in period
- Mixed regular and irregular pay
- Rounding verification (round up)

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC guidance § Average Weekly Earnings
- 02-01-core-payroll-spec.md § Statutory Payments
