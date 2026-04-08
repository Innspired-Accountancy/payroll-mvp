# Slice c: Tax Period Mapping and Week 53

**Story:** story-02-payroll-setup
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-b

---

## Goal

Implement HMRC tax period number mapping and Week 53 handling for weekly/four-weekly schedules according to HMRC rules.

---

## Decision Checklist

- [x] Algorithm: HMRC tax period rules from CWG2
- [x] Data contracts defined: TaxPeriodMapping type
- [x] Logic: Week 53 detection (April 5 falls on specific days)
- [x] Error scenarios: Week 53 validation, period alignment checks
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 Employer Further Guide — Week 53 rules
- 02-01-core-payroll-spec.md — Tax period field

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/tax-periods.ts` | create | Tax period mapping logic |
| `src/lib/config/tax-periods.ts` | create | Tax year period definitions |
| `src/tests/tax-periods.test.ts` | create | Tax period tests |

---

## Responsibilities

1. Map calendar dates to HMRC tax week numbers
2. Map calendar dates to HMRC tax month numbers
3. Detect Week 53 scenarios
4. Validate period alignment with tax periods
5. Handle period 12/13 splits for monthly

---

## Contracts

### getTaxWeekNumber
- **Method:** `getTaxWeekNumber(date: Date, taxYear: string): number`
- **Input:** Date and tax year string
- **Output:** Tax week number (1-53)
- **Logic:** Week 1 starts April 6, weekly boundaries

### getTaxMonthNumber
- **Method:** `getTaxMonthNumber(date: Date, taxYear: string): number`
- **Input:** Date and tax year string
- **Output:** Tax month number (1-12)
- **Logic:** Month 1 = April 6 - May 5, etc.

### isWeek53
- **Method:** `isWeek53(taxYear: string): boolean`
- **Input:** Tax year string
- **Output:** True if tax year has 53 weeks
- **Logic:** April 5 falls on specific weekday

---

## Business Rules & Invariants

1. Tax Week 1 starts on April 6
2. Week 53 occurs when April 5 is one of: Thursday (weekly), Wednesday (fortnightly), Tuesday (four-weekly)
3. Tax Month 1: April 6 - May 5, Month 2: May 6 - June 5, etc.
4. Periods must align with tax period boundaries for reporting

---

## Edge Cases

1. **Week 53 with 52-week schedule** — Handle per HMRC (use Week 52 rates or Week 1 of new year)
2. **Period spanning tax month boundary** — Use pay date to determine tax period
3. **Early year payroll start** — Handle pre-April 6 dates correctly

---

## Tests

### tax-periods.test.ts
- Tax week calculation for various dates
- Tax month calculation for period boundaries
- Week 53 detection for multiple years
- Period alignment validation
- Leap year tax year handling

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § Tax Weeks and Months
- 02-01-core-payroll-spec.md § Tax Period mapping
