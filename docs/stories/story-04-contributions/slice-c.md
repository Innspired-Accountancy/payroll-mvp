# Slice c: Payslip Integration

**Story:** story-04-contributions
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Integrate pension contributions with payslip generation to display employee deductions and employer contributions.

---

## Decision Checklist

- [x] Payslip display: Employee deduction, employer contribution, total
- [x] YTD tracking: Year-to-date contribution totals
- [x] Format: Standard payslip line items with descriptions
- [x] Integration point: Payroll calculation → payslip generation
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — PensionContribution entity
- 02-01-core-payroll-spec.md — Payslip generation

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/payslipIntegration.ts` | create | Pension line items for payslip |
| `src/lib/pension/ytdTracker.ts` | create | Year-to-date contribution tracking |
| `src/tests/pension/payslipIntegration.test.ts` | create | Integration tests |

---

## Responsibilities

1. Generate payslip line items for pension deductions
2. Display employee contribution (net deduction)
3. Display employer contribution (informational)
4. Show scheme name on payslip
5. Track YTD contributions per employee
6. Include YTD totals on payslip

---

## Contracts

### getPensionLineItems()
- **Method:** `getPensionLineItems(payslipId: UUID): PayslipLineItem[]`
- **Returns:** Array of line items for payslip display
- **Items:**
  - Pension (Employee): Negative amount (deduction)
  - Employer Pension: Positive amount (informational, not deducted)

### getPensionYtd()
- **Method:** `getPensionYtd(employeeId: UUID, taxYear: string): YtdTotals`
- **Returns:** { employeeTotal, employerTotal, grandTotal }
- **Calculation:** Sum all contributions from tax year start to date

### PayslipLineItem
| Field | Type | Description |
|-------|------|-------------|
| description | string | "Pension (Employee) - NEST" |
| amount | Decimal | Negative for deduction |
| ytd_amount | Decimal | Year-to-date total |

---

## Business Rules & Invariants

1. Employee deduction shown as negative amount
2. Employer contribution shown as positive (informational only)
3. YTD totals updated with each payslip
4. Tax year boundaries reset YTD counters
5. Multiple schemes shown as separate line items

---

## Edge Cases

1. **Multiple pension schemes** — Separate line item per scheme
2. **No pension this period** — No line items shown
3. **First payslip of tax year** — YTD = current period amount
4. **Leaver final payslip** — Final YTD shown

---

## Tests

### payslipIntegration.test.ts
- Line items generated for pension contribution
- YTD calculation across multiple periods
- Multiple schemes displayed correctly
- Zero contribution (no line items)

---

## Verification

```bash
npm run test:unit src/tests/pension/payslipIntegration.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-04-contributions/slice-a.md → Contribution data source
- story-10-payslip-generation → Payslip integration point
