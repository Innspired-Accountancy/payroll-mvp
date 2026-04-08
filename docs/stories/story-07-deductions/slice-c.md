# Slice C: Basic AEO Calculation Engine

**Story:** story-07-deductions
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a

## Goal

Implement basic Attachment of Earnings Order (AEO) calculation that operates on post-tax net pay. Calculate deduction amounts based on protected earnings thresholds and enforce priority ordering when multiple AEOs apply to the same employee.

## Decision Checklist

- [x] All libraries/packages for this slice named with versions (from epic Decisions)
  - Decimal.js 10.4.x for precise financial calculations
  - Drizzle ORM 0.30.x for database queries
- [x] All SDK methods/API calls identified (not just "uses X API" — actual method signatures)
  - `calculateAEOs(params: AEOInput): AEOResult` — core AEO calculation
  - `calculateProtectedEarnings(netPay: Decimal, rate: Decimal): Decimal` — protected amount
  - `prioritizeAEOs(aeos: EmployeeDeduction[]): EmployeeDeduction[]` — sort by priority
  - `distributeAvailableFunds(aeos: AEOToCalc[], available: Decimal): AEOCalculation[]` — handle insufficient funds
- [x] All external service endpoints specified (paths, methods, auth, payload shapes)
  - No external service calls in this slice (internal calculation engine)
- [x] All data contracts defined (input/output types, schemas, models)
  - See Contracts section below
- [x] All configuration/environment variables listed
  - None required for this slice
- [x] All error scenarios identified with handling strategy
  - Invalid AEO reference format: 400 validation error
  - Multiple AEOs exceed net pay: Proportional distribution applied, logged
  - Protected earnings calculation overflow: Capped at net pay, warning logged
  - Zero or negative net pay: All AEOs zero, flagged for review
- [x] No "TBD", slash-notation, or `{placeholder}` text remaining

## Spec References

- epic-plan.md § Data Contracts → PayCalculationResult (net_pay field)
- epic-plan.md § Scope & Deliverables → Basic AEO support
- story-03-tax-calculator/slice files for net pay dependency

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/aeo.ts` | create | Core AEO calculation engine |
| `src/lib/calculations/aeo-priorities.ts` | create | AEO priority constants and sorting |
| `src/lib/validations/aeo.ts` | create | Zod schemas for AEO inputs and reference formats |
| `src/server/api/routers/aeos.ts` | create | tRPC router for AEO management |
| `src/server/api/routers/calculations.ts` | update | Add AEO calculation endpoint |
| `tests/unit/calculations/aeo.test.ts` | create | Unit tests for AEO calculations |
| `tests/integration/aeo-priority.test.ts` | create | Integration tests for multiple AEOs |

## Responsibilities

1. Calculate protected earnings threshold for each AEO based on net pay
2. Determine available amount for AEO deduction (net pay minus protected earnings)
3. Apply priority ordering: CSA/CMS first, then court fines, then council tax
4. Calculate individual AEO amounts based on fixed value or percentage
5. Handle insufficient funds by applying priority order until funds exhausted
6. Support proportional distribution for same-priority AEOs when funds limited
7. Generate AEO payment schedule data for remittance reporting
8. Track AEO deduction history for compliance reporting

## Contracts

### AEOInput

```typescript
interface AEOInput {
  employeeId: string;
  netPay: Decimal;               // Post-tax, post-NIC net pay
  payPeriod: PayPeriod;          // Period dates
  aeoDeductions: EmployeeDeduction[]; // Active AEO assignments
}
```

### AEOResult

```typescript
interface AEOResult {
  totalAEODeductions: Decimal;   // Sum of all AEO amounts
  finalNetPay: Decimal;          // Net pay after AEOs
  availableForDeduction: Decimal; // Net pay minus protected earnings
  individualAEOs: IndividualAEOResult[];
  distributionMethod: 'full' | 'priority_capped' | 'proportional';
  warnings: string[];            // Non-blocking warnings (e.g., partial payment)
}
```

### IndividualAEOResult

```typescript
interface IndividualAEOResult {
  deductionId: string;           // Employee deduction reference
  aeoType: 'csa_cms' | 'court_fine' | 'council_tax';
  reference: string;             // AEO reference number
  priority: number;              // 1, 2, or 3
  protectedEarningsRate: Decimal; // e.g., 0.60 for 60% protected
  protectedAmount: Decimal;      // Net pay × protected rate
  requestedAmount: Decimal;      // Full calculated amount
  actualAmount: Decimal;         // Actual deducted (may be less due to funds)
  shortfall: Decimal | null;     // Amount not deducted
  ytdAmount: Decimal;            // Running total for this AEO
  remittanceData: {
    payee: string;               // Court, council, CSA
    reference: string;
    amount: Decimal;
  };
}
```

### AEO Priority Constants

```typescript
const AEOPriorities = {
  CSA_CMS: 1,        // Child Support Agency / Child Maintenance Service
  COURT_FINE: 2,     // Magistrates' court fines
  COUNCIL_TAX: 3,    // Council tax AEOs
  // Priority 4+ reserved for future expansion (not in basic scope)
} as const;

const ProtectedEarningsRates = {
  CSA_CMS: 0.60,     // 60% of net pay protected
  COURT_FINE: 0.75,  // 75% of net pay protected
  COUNCIL_TAX: 0.75, // 75% of net pay protected
} as const;
```

### AEO Calculation Rules

**Basic AEO Calculation:**
```
available = netPay - (netPay × protectedRate)
aeoAmount = min(requestedAmount, available)
```

**Multiple AEOs (Priority Order):**
```
1. Sort AEOs by priority (ascending)
2. remainingFunds = netPay - protectedEarnings(total)
3. For each AEO in priority order:
   - Calculate protected amount for this specific AEO
   - available = remainingFunds
   - actual = min(requested, available)
   - remainingFunds -= actual
   - If remainingFunds <= 0, stop processing
```

**Same Priority Distribution:**
- When multiple AEOs share priority (rare), distribute proportionally
- Each receives: (aeo.requested / totalRequested) × availableFunds

### tRPC Endpoints

**aeo.create**
- Input: `CreateAEOInput` (employeeId, type, reference, amount/percentage, priority)
- Output: `EmployeeDeduction`
- Auth: Employer/Admin
- Errors: 400 invalid reference format, 409 duplicate reference active

**aeo.listByEmployee**
- Input: `{ employeeId: string, activeOnly?: boolean }`
- Output: `EmployeeDeduction[]`
- Auth: Own data or admin

**calculation.calculateAEOs**
- Input: `{ employeeId: string, payPeriodId: string, netPay: Decimal }`
- Output: `AEOResult`
- Auth: Employer/Admin
- Errors: 404 employee not found, 404 pay period not found

## Business Rules & Invariants

1. **AEOs never reduce pay below protected earnings** — hard floor enforced
2. **CSA/CMS takes absolute priority** — must be paid in full before other AEOs
3. **Protected earnings are calculated per-AEO** — each has its own threshold
4. **Council tax AEOs limited to council tax arrears only** — current year handled separately
5. **AEO reference numbers must be validated** — format: court refs (e.g., "CT12345678"), CSA refs
6. **Partial payments must be tracked** — shortfall amounts recorded for catch-up
7. **Employer administration fee** — £1 per AEO payment (not deducted from employee in basic implementation)

## Edge Cases

1. **Zero net pay** — all AEOs return zero, flagged for manual review
2. **Net pay below protected earnings** — no AEO deduction possible, full shortfall recorded
3. **Multiple AEOs same priority** — proportional distribution (rare edge case)
4. **AEO amount exceeds available funds** — capped at available, shortfall tracked
5. **Mid-period AEO activation** — apply from effective date, no pro-rata (AEOs are per-period)
6. **AEO reference number change** — require new deduction record, old one ended
7. **Protected earnings rate > 100%** — validation error on creation
8. **Negative net pay (tax rebate scenario)** — AEOs suspended, flagged for review

## Tests

### aeo.test.ts
- `single aeo calculates correct deduction amount`
- `protected earnings threshold enforced`
- `csa priority higher than council tax`
- `multiple aeos in priority order`
- `insufficient funds applies priority order`
- `proportional distribution for same priority`
- `zero net pay returns zero deductions`
- `net pay below protected earnings returns zero`
- `negative net pay suspends deductions`
- `ytd accumulation tracked correctly`
- `shortfall amounts recorded for catch-up`

### aeo-priority.test.ts
- `csa paid in full before court fine`
- `court fine paid before council tax`
- `partial csa prevents other aeo payments`
- `multiple council tax aeos share proportionally`

### validations.test.ts
- `valid court reference accepted`
- `valid csa reference accepted`
- `invalid reference format rejected`
- `protected rate > 100% rejected`
- `negative amount rejected`

## Verification

```bash
# Lint check
npm run lint

# Type check
npm run typecheck

# Run AEO calculation tests
npm run test -- calculations/aeo
npm run test -- aeo-priority

# Build check
npm run build
```

## Source Sections

- epic-plan.md § Scope & Deliverables → Basic AEO support
- epic-plan.md § Out of Scope → Advanced AEO edge cases (phase 2)
- epic-plan.md § Data Types → Decimal.js for all monetary values
