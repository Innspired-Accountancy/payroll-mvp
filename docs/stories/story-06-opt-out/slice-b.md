# Slice b: Refund Calculation and Processing

**Story:** story-06-opt-out
**Epic:** epic-03-pension-ae
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Calculate and process refunds of employee contributions for opted-out employees via the next payroll run.

---

## Decision Checklist

- [x] Refund scope: Employee contributions only (employer keeps theirs)
- [x] Calculation: Sum of employee contributions from enrolment date to opt-out date
- [x] Payroll integration: Add as negative deduction on next payslip
- [x] Tax treatment: Refund is tax-free (already taxed via payroll)
- [x] YTD adjustment: Reduce YTD contribution totals
- [x] NEST reconciliation: Notify NEST of refund
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Journey 3, refund processing
- HMRC: Pension refunds after opt-out

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pension/refundCalculator.ts` | create | Refund amount calculation |
| `src/lib/pension/refundProcessor.ts` | create | Payroll integration for refunds |
| `src/lib/db/schema/pensionRefunds.ts` | create | Refund record table |
| `src/tests/pension/refundCalculator.test.ts` | create | Calculation tests |

---

## Responsibilities

1. Calculate total employee contributions since enrolment
2. Create refund record with breakdown
3. Queue refund for next payroll processing
4. Add refund to payslip as negative deduction
5. Update YTD contribution totals
6. Record refund completion

---

## Contracts

### calculateRefund()
- **Method:** `calculateRefund(enrolmentId: UUID, optOutDate: Date): RefundCalculation`
- **Returns:** RefundCalculation with amount and breakdown
- **Calculation:** Sum of employee_contribution where contribution_date between enrolment and opt-out

### RefundCalculation
| Field | Type | Description |
|-------|------|-------------|
| total_refund | Decimal | Total amount to refund |
| contribution_breakdown | ContributionItem[] | Per-period breakdown |
| tax_year | string | Tax year for refund |
| enrolment_date | Date | Original enrolment date |
| opt_out_date | Date | Opt-out date |

### processRefund()
- **Method:** `processRefund(refundId: UUID, payslipId: UUID): Promise<void>`
- **Action:** 
  1. Add refund line to payslip (negative amount)
  2. Update YTD contribution totals
  3. Mark refund as processed
  4. Notify NEST of refund

### RefundRecord
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Refund ID |
| enrolment_id | UUID | Source enrolment |
| amount | Decimal | Refund amount |
| status | enum | "pending" / "processed" |
| payslip_id | UUID | Payslip where processed |
| processed_at | DateTime | When processed |

---

## Business Rules & Invariants

1. Only employee contributions refunded (employer contributions stay in scheme)
2. Refund calculated from enrolment date to opt-out date
3. Refund processed on next available payroll run
4. Refund shown as negative "Pension Refund" on payslip
5. YTD contribution totals reduced by refund amount
6. Tax already paid on contributions is adjusted via payroll

---

## Edge Cases

1. **Zero contributions to refund** — Record opt-out, no refund needed
2. **Multiple pay periods** — Sum all contributions across periods
3. **Leaver before refund processed** — Process in final payslip or manual payment
4. **Partial period opt-out** — Refund full contributions from completed periods

---

## Tests

### refundCalculator.test.ts
- Single period refund calculation
- Multiple period refund calculation
- Zero contribution handling
- Date range boundary
- YTD adjustment calculation

---

## Verification

```bash
npm run test:unit src/tests/pension/refundCalculator.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-06-opt-out/slice-a.md → Opt-out data source
- story-04-contributions → Contribution data for calculation
