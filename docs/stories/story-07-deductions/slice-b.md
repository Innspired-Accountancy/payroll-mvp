# Slice B: Salary Sacrifice Calculation

**Story:** story-07-deductions
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a

## Goal

Implement salary sacrifice calculation logic that reduces taxable and/or NICable pay before tax and NIC calculations. Integrate with the existing tax and NIC calculation pipeline to ensure correct PAYE and NIC amounts are computed on post-sacrifice earnings.

## Decision Checklist

- [x] All libraries/packages for this slice named with versions (from epic Decisions)
  - Decimal.js 10.4.x for precise financial calculations
  - Drizzle ORM 0.30.x for database queries
  - date-fns 3.x for period calculations
- [x] All SDK methods/API calls identified (not just "uses X API" — actual method signatures)
  - `calculateSalarySacrifice(params: SacrificeInput): SacrificeResult` — core calculation
  - `adjustTaxablePay(gross: Decimal, sacrifices: SacrificeResult[]): Decimal` — reduce taxable
  - `adjustNICablePay(gross: Decimal, sacrifices: SacrificeResult[]): Decimal` — reduce NICable
  - `taxCalculator.calculate(taxablePay: Decimal, ...): TaxResult` — existing tax calc (from story-03)
  - `nicCalculator.calculate(nicablePay: Decimal, ...): NICResult` — existing NIC calc (from story-04)
- [x] All external service endpoints specified (paths, methods, auth, payload shapes)
  - No external service calls in this slice (internal calculation engine)
- [x] All data contracts defined (input/output types, schemas, models)
  - See Contracts section below
- [x] All configuration/environment variables listed
  - None required for this slice
- [x] All error scenarios identified with handling strategy
  - NMW violation after sacrifice: Warning returned, calculation proceeds with flag
  - Invalid sacrifice configuration: 400 with field-level errors
  - Pension AE compliance breach: Logged, employer notification triggered
- [x] No "TBD", slash-notation, or `{placeholder}` text remaining

## Spec References

- epic-plan.md § Calculation Engine Design → Tax Calculator, NIC Calculator
- epic-plan.md § Data Contracts → PayCalculationInput, PayCalculationResult
- story-03-tax-calculator/slice files for tax integration points
- story-04-nic-calculator/slice files for NIC integration points

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/salary-sacrifice.ts` | create | Core salary sacrifice calculation engine |
| `src/lib/calculations/deduction-pipeline.ts` | create | Integration layer with tax/NIC calculators |
| `src/lib/validations/sacrifice.ts` | create | Zod schemas for sacrifice inputs |
| `src/server/api/routers/calculations.ts` | update | Add sacrifice calculation endpoint |
| `tests/unit/calculations/salary-sacrifice.test.ts` | create | Unit tests for sacrifice calculations |
| `tests/integration/sacrifice-tax-integration.test.ts` | create | Integration tests with tax/NIC |

## Responsibilities

1. Calculate salary sacrifice amounts based on percentage or fixed configuration
2. Determine tax and NIC treatment based on deduction type (pension, cycle-to-work, childcare)
3. Reduce taxable pay for applicable sacrifice types before tax calculation
4. Reduce NICable pay for applicable sacrifice types before NIC calculation
5. Validate NMW compliance post-sacrifice and flag violations
6. Track AE pension compliance for qualifying earnings calculations
7. Return detailed breakdown for payslip display and audit trail

## Contracts

### SacrificeInput

```typescript
interface SacrificeInput {
  employeeId: string;
  grossSalary: Decimal;          // Pre-sacrifice gross
  payPeriod: PayPeriod;           // Period dates for pro-rating
  deductions: EmployeeDeduction[]; // Active salary sacrifice deductions
  nmwRate: Decimal;               // Applicable NMW rate for age bracket
  hoursWorked: number;            // For NMW calculation
}
```

### SacrificeResult

```typescript
interface SacrificeResult {
  deductionId: string;            // Employee deduction reference
  deductionTypeCode: string;      // Type code (e.g., "PENSION_SS")
  name: string;                   // Display name
  amount: Decimal;                // Sacrifice amount this period
  reducesTaxable: boolean;        // Affects PAYE calculation
  reducesNICable: boolean;        // Affects NIC calculation
  reducesEmployerNIC: boolean;    // Affects employer NIC
  taxablePayImpact: Decimal;      // Amount subtracted from taxable pay
  nicablePayImpact: Decimal;      // Amount subtracted from NICable pay
  ytdAmount: Decimal;             // Running total for this deduction type
}
```

### SacrificeCalculationOutput

```typescript
interface SacrificeCalculationOutput {
  originalGross: Decimal;
  totalSacrificeAmount: Decimal;
  adjustedTaxablePay: Decimal;    // For input to tax calculator
  adjustedNICablePay: Decimal;    // For input to NIC calculator
  individualResults: SacrificeResult[];
  nmwCompliant: boolean;          // Post-sacrifice NMW check
  nmwShortfall: Decimal | null;   // Amount below NMW if applicable
  aeCompliant: boolean;           // Auto-enrolment qualifying earnings check
  warnings: string[];             // Non-blocking warnings
}
```

### Integration with Tax/NIC Calculators

The sacrifice calculation must run **before** tax and NIC calculations:

```
Pay Calculation Flow:
1. Get gross salary
2. Calculate salary sacrifices (this slice)
3. adjustedTaxablePay = gross - sum(taxable sacrifices)
4. adjustedNICablePay = gross - sum(nicable sacrifices)
5. Call taxCalculator.calculate(adjustedTaxablePay, ...)
6. Call nicCalculator.calculate(adjustedNICablePay, ...)
7. Continue with post-tax deductions (AEOs)
```

### tRPC Endpoint

**calculation.calculateSalarySacrifices**
- Input: `{ employeeId: string, payPeriodId: string }`
- Output: `SacrificeCalculationOutput`
- Auth: Employer/Admin
- Errors: 404 employee not found, 404 pay period not found, 400 invalid configuration

## Business Rules & Invariants

1. **Pension salary sacrifice** reduces both taxable pay and employee NICable pay
2. **Cycle-to-work scheme** reduces taxable pay but NOT NICable pay
3. **Childcare vouchers** (legacy) reduce taxable pay but NOT NICable pay (subject to limits)
4. **Total sacrifices cannot exceed gross pay** — enforced calculation limit
5. **NMW compliance**: Post-sacrifice pay must be >= NMW rate × hours worked
6. **AE pension qualifying earnings**: Sacrifices that reduce NICable pay also reduce qualifying earnings
7. **Pro-rata for mid-period changes**: Sacrifices apply from effectiveFrom date within period

## Edge Cases

1. **Multiple pension sacrifices** — combine and apply before other sacrifices (pension takes precedence)
2. **Sacrifice exceeds taxable pay** — cap at 100% of taxable pay, log warning
3. **Mid-period sacrifice start** — calculate daily rate and apply for remaining days
4. **NMW threshold breach** — flag violation but allow calculation (employer responsibility)
5. **Zero gross period** — zero sacrifices, no errors
6. **Negative sacrifice amount** — validation error (absolute value required)

## Tests

### salary-sacrifice.test.ts
- `pension sacrifice reduces taxable and nicable pay`
- `cycle to work reduces taxable but not nicable pay`
- `multiple sacrifices combined correctly`
- `percentage-based sacrifice calculated correctly`
- `fixed amount sacrifice applied correctly`
- `pro-rata calculation for mid-period start`
- `nmw compliance check flags violation`
- `sacrifice capped at gross pay amount`
- `ytd accumulation tracked correctly`

### sacrifice-tax-integration.test.ts
- `tax calculated on post-sacrifice taxable pay`
- `employee nic calculated on post-sacrifice nicable pay`
- `employer nic reduced by applicable sacrifices`
- `full calculation pipeline produces correct net pay`
- `calculation trace includes sacrifice details`

## Verification

```bash
# Lint check
npm run lint

# Type check
npm run typecheck

# Run sacrifice calculation tests
npm run test -- calculations/salary-sacrifice
npm run test -- sacrifice-tax-integration

# Build check
npm run build
```

## Source Sections

- epic-plan.md § Scope & Deliverables → Salary sacrifice and pre/post-tax deductions
- epic-plan.md § Calculation Engine Design → Tax Calculator (HMRC exact percentage), NIC Calculator (category-based)
