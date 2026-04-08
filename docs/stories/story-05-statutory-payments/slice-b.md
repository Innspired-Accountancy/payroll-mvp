# Slice b: SMP Calculation Engine

**Story:** story-05-statutory-payments
**Epic:** epic-01-core-payroll
**Effort:** L
**Dependencies:** None

---

## Goal

Implement Statutory Maternity Pay calculation with 90% for 6 weeks, flat rate for 33 weeks, AWE qualification, and NIC/tax treatment.

---

## Decision Checklist

- [x] Rates: 90% for 6 weeks, £184.03 flat rate (2026-27)
- [x] Logic: AWE qualification, 26 weeks service
- [x] Phases: Higher rate (6w) + Lower rate (33w)
- [x] NIC: Employer pays NIC on SMP
- [x] Error handling: Ineligible employee, AWE below threshold
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC E15 — SMP employer guide
- 02-01-core-payroll-spec.md — Statutory payments

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/smp.ts` | create | SMP calculator |
| `src/lib/db/schema/maternity-leave.ts` | create | SMP record storage |
| `src/tests/calculations/smp.test.ts` | create | SMP tests |

---

## Responsibilities

1. Calculate AWE for qualification and rate determination
2. Determine 90% rate (higher rate phase)
3. Apply flat rate for remaining weeks (lower rate phase)
4. Handle NIC exemption for employee
5. Support recovery calculations (92%/103%)

---

## Contracts

### calculateSmp
- **Method:** `calculateSmp(input: SmpInput): SmpResult`
- **Input:** SmpInput with maternity details
- **Output:** SmpResult with SMP amount breakdown

### SmpInput
| Field | Type | Description |
|-------|------|-------------|
| maternity_start | Date | Maternity leave start date |
| expected_week | Date | Expected week of childbirth |
| actual_week | Date | Actual week of childbirth (optional) |
| awe | Decimal | Average Weekly Earnings |
| weeks_claimed | int | Weeks already claimed (for continuing) |
| is_small_employer | boolean | For recovery rate |
| tax_year | string | Tax year for rates |

### SmpResult
| Field | Type | Description |
|-------|------|-------------|
| smp_amount | Decimal | Total SMP for period |
| higher_rate_amount | Decimal | 90% portion |
| lower_rate_amount | Decimal | Flat rate portion |
| weeks_at_higher | int | Weeks at 90% rate |
| weeks_at_lower | int | Weeks at flat rate |
| recovery_amount | Decimal | Recoverable amount |

---

## Business Rules & Invariants

1. Qualifying conditions: 26 weeks service by qualifying week, AWE >= LEL
2. Higher rate: 90% of AWE for first 6 weeks
3. Lower rate: £184.03/week (2026-27) for next 33 weeks
4. Max 39 weeks total SMP
5. Employee NIC not deducted from SMP
6. Small employer relief: recover 103%, otherwise 92%

---

## Edge Cases

1. **AWE exactly at LEL** — Qualifies for flat rate only
2. **Baby born early** — Use actual week, recalculate if needed
3. **Employee leaves** — SMP continues if already started
4. **Overlapping tax years** — Continue into new tax year
5. **90% < flat rate** — Pay flat rate instead (higher rate floor)

---

## Tests

### smp.test.ts
- Standard 39-week SMP calculation
- Higher rate (90%) for first 6 weeks
- Lower rate (£184.03) for remaining weeks
- AWE exactly at threshold
- Early childbirth adjustment
- Recovery calculation (92% vs 103%)
- Week boundary calculations

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC E15 § SMP calculation
- 02-01-core-payroll-spec.md § Statutory Payments
