# Slice c: SPP Calculation Engine

**Story:** story-05-statutory-payments
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** None

---

## Goal

Implement Statutory Paternity Pay calculation with flat rate for 1-2 weeks and AWE qualification.

---

## Decision Checklist

- [x] Rate: £184.03/week flat rate (2026-27)
- [x] Duration: 1 or 2 consecutive weeks
- [x] Logic: AWE qualification, 26 weeks service
- [x] NIC: Employer pays NIC on SPP
- [x] Error handling: Ineligible employee, AWE below threshold
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC E19 — SPP employer guide
- 02-01-core-payroll-spec.md — Statutory payments

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/spp.ts` | create | SPP calculator |
| `src/lib/db/schema/paternity-leave.ts` | create | SPP record storage |
| `src/tests/calculations/spp.test.ts` | create | SPP tests |

---

## Responsibilities

1. Calculate AWE for qualification
2. Apply flat rate for SPP weeks
3. Support 1 or 2 week duration
4. Handle NIC exemption for employee
5. Support recovery calculations (92%/103%)

---

## Contracts

### calculateSpp
- **Method:** `calculateSpp(input: SppInput): SppResult`
- **Input:** SppInput with paternity details
- **Output:** SppResult with SPP amount

### SppInput
| Field | Type | Description |
|-------|------|-------------|
| paternity_start | Date | SPP start date |
| weeks_duration | int | 1 or 2 weeks |
| awe | Decimal | Average Weekly Earnings |
| is_small_employer | boolean | For recovery rate |
| tax_year | string | Tax year for rates |

### SppResult
| Field | Type | Description |
|-------|------|-------------|
| spp_amount | Decimal | Total SPP for period |
| weekly_rate | Decimal | Rate per week |
| weeks_paid | int | Number of weeks paid |
| recovery_amount | Decimal | Recoverable amount |

---

## Business Rules & Invariants

1. Qualifying conditions: 26 weeks service, AWE >= LEL
2. Flat rate: £184.03/week (2026-27)
3. Duration: 1 or 2 consecutive weeks
4. Must be taken within 8 weeks of birth/adoption
5. Employee NIC not deducted from SPP
6. Small employer relief: recover 103%, otherwise 92%

---

## Edge Cases

1. **Partial week** — Calculate daily rate if period ends mid-week
2. **Birth/adoption date change** — Recalculate window
3. **Employee leaves** — SPP continues if already started
4. **Overlapping tax year** — Continue into new tax year

---

## Tests

### spp.test.ts
- 1-week SPP calculation
- 2-week SPP calculation
- AWE qualification check
- Partial week calculation
- Recovery calculation

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC E19 § SPP calculation
- 02-01-core-payroll-spec.md § Statutory Payments
