# Slice A: Deduction Types Data Model

**Story:** story-07-deductions
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** none

## Goal

Create the database schema and data models for deduction types (salary sacrifice and AEO), employee deduction assignments, and deduction history tracking. This slice establishes the foundation that all deduction calculations depend on.

## Decision Checklist

- [x] All libraries/packages for this slice named with versions (from epic Decisions)
  - Drizzle ORM 0.30.x for schema definitions
  - Zod 3.22.x for validation schemas
- [x] All SDK methods/API calls identified (not just "uses X API" — actual method signatures)
  - `db.insert(deductionTypes).values(data)` — create deduction type
  - `db.query.deductionTypes.findMany()` — list deduction types
  - `db.insert(employeeDeductions).values(data)` — assign deduction to employee
  - `db.query.employeeDeductions.findMany({ where: eq(employeeDeductions.employeeId, id) })` — get employee deductions
- [x] All external service endpoints specified (paths, methods, auth, payload shapes)
  - No external service calls in this slice (internal data layer only)
- [x] All data contracts defined (input/output types, schemas, models)
  - See Contracts section below
- [x] All configuration/environment variables listed
  - None required for this slice
- [x] All error scenarios identified with handling strategy
  - Duplicate deduction type code: 409 Conflict with error message
  - Invalid deduction parameters: 400 with field-level validation errors
  - Referential integrity violations: 409 with constraint details
- [x] No "TBD", slash-notation, or `{placeholder}` text remaining

## Spec References

- epic-plan.md § Data Contracts → PayCalculationInput, PayCalculationResult
- epic-plan.md § Decisions → Data Types (Decimal.js for money)

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/db/schema/deductions.ts` | create | Deduction types, employee deductions, and deduction history tables |
| `src/db/schema/index.ts` | update | Export new deduction schemas |
| `src/lib/validations/deductions.ts` | create | Zod schemas for deduction validation |
| `src/server/api/routers/deductions.ts` | create | tRPC router for deduction management |
| `src/server/api/routers/_app.ts` | update | Register deductions router |
| `tests/unit/deductions/schema.test.ts` | create | Schema validation tests |

## Responsibilities

1. Define deduction type taxonomy (salary sacrifice vs post-tax vs AEO)
2. Store deduction configuration parameters (percentage, fixed amount, limits)
3. Link deductions to employees with effective date ranges
4. Track deduction history for audit and YTD calculations
5. Support soft deletion/archiving of deduction assignments

## Contracts

### DeductionType Model

```typescript
interface DeductionType {
  id: string;                    // UUID primary key
  code: string;                  // Unique code (e.g., "PENSION_SS", "AEO_COUNCIL_TAX")
  name: string;                  // Display name
  category: 'salary_sacrifice' | 'post_tax' | 'aeo';
  subType: string;               // Specific type (pension, cycle_to_work, childcare, csa, court_fine, council_tax)
  taxTreatment: 'reduces_taxable' | 'reduces_nic' | 'reduces_both' | 'no_reduction';
  nicTreatment: 'reduces_employer' | 'reduces_employee' | 'reduces_both' | 'no_reduction';
  calculationMethod: 'percentage' | 'fixed_amount' | 'sliding_scale';
  defaultValue: Decimal | null;  // Default percentage or amount
  minAmount: Decimal | null;     // Minimum deduction
  maxAmount: Decimal | null;     // Maximum deduction or cap
  isPension: boolean;            // For AE compliance tracking
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### EmployeeDeduction Model

```typescript
interface EmployeeDeduction {
  id: string;                    // UUID primary key
  employeeId: string;            // FK to employees
  deductionTypeId: string;       // FK to deduction_types
  value: Decimal;                // Percentage or fixed amount
  reference: string | null;      // AEO reference number, pension policy number
  priority: number;              // For multiple AEO ordering (lower = higher priority)
  protectedEarningsRate: Decimal | null; // For AEOs: percentage of net pay protected
  effectiveFrom: Date;           // Start date
  effectiveTo: Date | null;      // End date (null = ongoing)
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}
```

### DeductionHistory Model

```typescript
interface DeductionHistory {
  id: string;                    // UUID primary key
  employeeDeductionId: string;   // FK to employee_deductions
  payRunId: string;              // FK to pay_runs
  payPeriodId: string;           // FK to pay_periods
  amount: Decimal;               // Actual amount deducted
  taxablePayBefore: Decimal;     // Taxable pay before this deduction
  taxablePayAfter: Decimal;      // Taxable pay after this deduction
  nicablePayBefore: Decimal;     // NICable pay before this deduction
  nicablePayAfter: Decimal;      // NICable pay after this deduction
  ytdAmount: Decimal;            // Cumulative for this deduction type
  createdAt: Date;
}
```

### tRPC Router Endpoints

**deduction.createType**
- Input: `CreateDeductionTypeInput` (Zod schema)
- Output: `DeductionType`
- Auth: Admin/Employer
- Errors: 409 duplicate code, 400 validation error

**deduction.listTypes**
- Input: `{ category?: string, isActive?: boolean }`
- Output: `DeductionType[]`
- Auth: Authenticated user

**deduction.assignToEmployee**
- Input: `AssignDeductionInput` (Zod schema)
- Output: `EmployeeDeduction`
- Auth: Admin/Employer
- Errors: 404 employee not found, 404 deduction type not found, 400 invalid dates

**deduction.getEmployeeDeductions**
- Input: `{ employeeId: string, activeOnly?: boolean }`
- Output: `EmployeeDeductionWithType[]`
- Auth: Own employee data or admin

## Business Rules & Invariants

1. **Deduction type codes must be unique** across the system (enforced at DB level)
2. **AEO deductions require reference number** — validation must ensure reference is provided for AEO category
3. **Effective date ranges cannot overlap** for the same employee and deduction type (DB constraint)
4. **Inactive deduction types cannot be assigned** to employees (application validation)
5. **Salary sacrifice cannot reduce pay below NMW** — this will be validated at calculation time, not assignment
6. **AEO priority defaults**: CSA/CMS = 1, Court fines = 2, Council tax = 3 (enforced if not specified)

## Edge Cases

1. **Employee has multiple salary sacrifices** — all apply cumulatively, order does not matter for pre-tax
2. **AEO assigned mid-period** — effectiveFrom date determines first applicable period
3. **Deduction type deactivated** — existing employee assignments remain but new assignments blocked
4. **Employee deduction end date reached** — automatically excluded from future pay runs
5. **Zero amount deduction** — allowed (no-op) but logged for audit trail

## Tests

### schema.test.ts
- `deduction type creation validates required fields`
- `deduction type code uniqueness enforced`
- `employee deduction effective date validation`
- `overlapping date ranges rejected`
- `AEO requires reference number`
- `inactive deduction type cannot be assigned`

### deductions.router.test.ts
- `createType creates valid deduction type`
- `listTypes filters by category`
- `assignToEmployee links deduction to employee`
- `getEmployeeDeductions returns with type details`
- `unauthorized access returns 403`

## Verification

```bash
# Lint check
npm run lint

# Type check
npm run typecheck

# Run deduction-related tests
npm run test -- deductions/schema
npm run test -- deductions/router

# Database migration dry-run
npm run db:generate
```

## Source Sections

- epic-plan.md § Decisions → Data Types (Decimal for money)
- epic-plan.md § Scope & Deliverables → Salary sacrifice and pre/post-tax deductions, Basic AEO support
