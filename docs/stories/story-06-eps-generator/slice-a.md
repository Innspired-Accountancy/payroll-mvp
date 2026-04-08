# Slice a: EPS Data Structures and Recovery Calculations

**Story:** story-06-eps-generator
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** None

---

## Goal

Define EPS data models and implement statutory payment recovery calculations. Support SSP, SMP, SPP, SAP recovery and NIC compensation calculations for monthly EPS submissions.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, decimal.js 10.4.x
- [x] SDK methods identified: db.query.statutoryPayments.findMany(), aggregate functions
- [x] External service endpoints: N/A (internal calculations)
- [x] Data contracts defined: EPSData, RecoveryAmounts, StatutoryPayment interfaces
- [x] Configuration: N/A
- [x] Error scenarios: Missing statutory payment records, calculation rounding errors
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 3: Submit EPS for Adjustments
- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/paye-schemes/{id}/generate-eps

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/eps/types.ts` | create | EPS data type definitions |
| `src/lib/hmrc/eps/calculator.ts` | create | Recovery calculation logic |
| `src/lib/hmrc/eps/recovery-rates.ts` | create | HMRC recovery rate constants |

---

## Responsibilities

1. Define EPS data structures for recovery amounts
2. Calculate SSP recovery (92% or 103% for small employers)
3. Calculate SMP/SPP/SAP recovery (92% or 103%)
4. Calculate NIC compensation on statutory payments
5. Determine small employer eligibility

---

## Contracts

### EPSRecoveryCalculator
- **Method:** `calculate(taxYear: string, taxMonth: number, employerId: string): Promise<EPSRecoveryData>`
- **Input:**
  - `taxYear: string` — e.g., "2026-27"
  - `taxMonth: number` — 1-12
  - `employerId: string` — Employer UUID
- **Output:** EPSRecoveryData
  ```typescript
  interface EPSRecoveryData {
    taxYear: string;
    taxMonth: number;
    employerId: string;
    recoveries: {
      ssp: {
        paid: Decimal;
        recoverable: Decimal;
        recoveryRate: 0.92 | 1.03;
      };
      smp: {
        paid: Decimal;
        recoverable: Decimal;
        recoveryRate: 0.92 | 1.03;
      };
      spp: {
        paid: Decimal;
        recoverable: Decimal;
        recoveryRate: 0.92 | 1.03;
      };
      sap: {
        paid: Decimal;
        recoverable: Decimal;
        recoveryRate: 0.92 | 1.03;
      };
    };
    nicCompensation: {
      total: Decimal;
      breakdown: {
        ssp: Decimal;
        smp: Decimal;
        spp: Decimal;
        sap: Decimal;
      };
    };
    totalRecovery: Decimal;
    isSmallEmployer: boolean;
    // CIS deductions (if applicable)
    cisDeductionsSuffered?: Decimal;
  }
  ```

### Recovery Rates (2026-27)
```typescript
const RECOVERY_RATES = {
  standard: 0.92,      // 92% for standard employers
  smallEmployer: 1.03  // 103% for small employers
};

// Small employer threshold: £45,000 or less in Class 1 NICs
// in the previous tax year
const SMALL_EMPLOYER_THRESHOLD = 45000;
```

### NIC Compensation Calculation
```typescript
function calculateNICCompensation(
  statutoryPaymentAmount: Decimal,
  paymentType: 'ssp' | 'smp' | 'spp' | 'sap'
): Decimal {
  // NIC compensation is the employer NICs that would have
  // been paid on the statutory payment
  // Rate varies by payment type and year
  const rate = getNICCompensationRate(paymentType, taxYear);
  return statutoryPaymentAmount.times(rate);
}
```

---

## Business Rules & Invariants

1. Recovery rates: 92% standard, 103% for small employers
2. Small employer determined by previous tax year NIC liability
3. Recovery claims must be made within tax year or following year
4. NIC compensation claimed separately from payment recovery
5. CIS deductions suffered can be offset against PAYE liability

---

## Edge Cases

1. **No statutory payments** — Allow "no payment to declare" EPS
2. **Employer crosses small employer threshold mid-year** — Use rate from determination
3. **Partial month employment** — Pro-rata calculations
4. **Negative recovery (over-recovery)** — Carry forward to next period
5. **Recovery exceeds PAYE liability** — Carry forward or claim refund

---

## Tests

### eps-calculator.test.ts
- Calculate recovery for small employer (103%)
- Calculate recovery for standard employer (92%)
- Calculate NIC compensation
- Handle zero statutory payments
- Handle CIS deductions

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/eps/calculator.test.ts
npm run lint src/lib/hmrc/eps/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § API Contracts → POST /api/v1/paye-schemes/{id}/generate-eps
- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 3: System shows recoverable amounts
