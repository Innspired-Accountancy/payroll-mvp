# Slice b: Pay Period Generation

**Story:** story-02-payroll-setup
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement pay period generation engine that creates periods for a tax year based on schedule frequency with correct start/end dates and pay dates.

---

## Decision Checklist

- [x] All libraries/packages named: date-fns 3.x, date-fns-tz 2.x
- [x] Algorithm: Period generation with date-fns addWeeks/addMonths
- [x] Data contracts defined: PeriodGeneratorInput, GeneratedPeriod Zod schemas
- [x] tRPC procedures: payPeriods.generate, payPeriods.list, payPeriods.update
- [x] Error scenarios: TRPCError CONFLICT (overlapping), BAD_REQUEST (invalid dates)
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — PayPeriod entity definition
- HMRC CWG2 — Period boundaries and Week 53 rules

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/period-generator.ts` | create | Period generation logic |
| `src/server/routers/pay-periods.ts` | create | tRPC procedures |
| `src/lib/validation/pay-periods.ts` | create | Zod schemas |
| `src/tests/period-generator.test.ts` | create | Generation logic tests |

---

## Responsibilities

1. Generate periods for a full tax year (April 6 - April 5)
2. Calculate period start/end based on frequency
3. Calculate pay date using pay_day_offset
4. Detect and prevent overlapping periods
5. Store generated periods in database

---

## Contracts

### payPeriods.generate
- **Method:** tRPC mutation `payPeriods.generate`
- **Input:** `{ schedule_id: uuid, tax_year: string }`
- **Output:** `{ periods: PayPeriod[], count: number }`
- **Errors:** NOT_FOUND (schedule), CONFLICT (periods exist)
- **Auth:** Protected procedure with payroll:write permission

### PeriodGenerator
- **Method:** `generatePeriods(schedule: PayrollSchedule, taxYear: string): GeneratedPeriod[]`
- **Input:** Schedule config and tax year (e.g., "2026-27")
- **Output:** Array of periods with start, end, pay_date
- **Logic:** Weekly = 52/53 periods, Monthly = 12 periods

### GeneratedPeriod
| Field | Type | Description |
|-------|------|-------------|
| period_start | Date | Start of pay period |
| period_end | Date | End of pay period |
| pay_date | Date | Date employees are paid |
| tax_period | int | HMRC tax week/month number |

---

## Business Rules & Invariants

1. Tax year runs April 6 to April 5
2. Weekly schedules: 52 periods + Week 53 if applicable
3. Monthly schedules: 12 periods (tax months 1-12)
4. Periods must not overlap within same schedule
5. Pay date must be >= period_end date
6. Period 1 starts on or after April 6

---

## Edge Cases

1. **Week 53 detection** — Identify when 53rd week applies
2. **Schedule spans tax year boundary** — Generate only within tax year
3. **Existing periods** — Block generation, require explicit delete
4. **Leap year February** — Handle 29 days correctly

---

## Tests

### period-generator.test.ts
- Generate monthly periods for tax year (12 periods)
- Generate weekly periods for non-53 week year (52 periods)
- Generate weekly periods for 53 week year (53 periods)
- Detect overlapping periods (expect error)
- Calculate pay date with offset
- Handle leap year February

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Data Models → PayPeriod
- HMRC CWG2 § Tax Weeks and Months
