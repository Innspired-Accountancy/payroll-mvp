# Slice D: Payslip Integration and YTD Tracking

**Story:** story-07-deductions
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-b, slice-c

## Goal

Integrate deduction calculations into payslip generation and implement year-to-date (YTD) tracking for all deduction types. Ensure deductions are clearly categorised on payslips and cumulative totals are maintained accurately.

## Decision Checklist

- [x] All libraries/packages for this slice named with versions (from epic Decisions)
  - Drizzle ORM 0.30.x for database queries and updates
  - Decimal.js 10.4.x for YTD accumulation
- [x] All SDK methods/API calls identified (not just "uses X API" — actual method signatures)
  - `db.insert(deductionHistory).values(data)` — record deduction transaction
  - `db.update(employeeDeductions).set({ ytdAmount: newTotal })` — update YTD
  - `db.query.deductionHistory.findMany({ where: eq(deductionHistory.employeeId, id) })` — get history
  - `payslipGenerator.addDeductionSection(deductions: DeductionDisplayData[])` — payslip integration
- [x] All external service endpoints specified (paths, methods, auth, payload shapes)
  - No external service calls in this slice (internal integration)
- [x] All data contracts defined (input/output types, schemas, models)
  - See Contracts section below
- [x] All configuration/environment variables listed
  - None required for this slice
- [x] All error scenarios identified with handling strategy
  - YTD calculation overflow: Decimal precision error, logged
  - Payslip generation failure: Deduction data still persisted, retry mechanism
  - Concurrent YTD update: Database transaction isolation handles
- [x] No "TBD", slash-notation, or `{placeholder}` text remaining

## Spec References

- epic-plan.md § Data Contracts → PayCalculationResult, YTDRecord
- epic-plan.md § Scope & Deliverables → Payslip generation with calculation trace
- story-10-payslip-generation/slice files for payslip integration points

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/payslip/deduction-formatter.ts` | create | Format deductions for payslip display |
| `src/lib/calculations/ytd-manager.ts` | create | YTD accumulation and retrieval |
| `src/server/services/payroll-calculation.ts` | update | Integrate deduction history recording |
| `tests/unit/payslip/deduction-formatter.test.ts` | create | Payslip formatting tests |
| `tests/unit/calculations/ytd-manager.test.ts` | create | YTD tracking tests |
| `tests/integration/deduction-ytd-payslip.test.ts` | create | End-to-end integration tests |

## Responsibilities

1. Format salary sacrifice deductions for payslip display with appropriate categorisation
2. Format AEO deductions for payslip display with reference numbers
3. Calculate and persist YTD totals per deduction type per employee
4. Record deduction history for audit trail and reversal support
5. Integrate with payslip generation pipeline to include deduction sections
6. Support payslip reversal by recording pre-image data
7. Generate deduction summaries for reporting (P60, P45)

## Contracts

### DeductionDisplayData

```typescript
interface DeductionDisplayData {
  section: 'pre_tax' | 'post_tax' | 'aeo';
  category: 'pension' | 'salary_sacrifice' | 'aeo';
  label: string;                 // Display name (e.g., "Pension Salary Sacrifice")
  reference: string | null;      // AEO reference or pension policy number
  currentAmount: Decimal;        // This period amount
  ytdAmount: Decimal;            // Year to date
  impactDescription: string | null; // e.g., "Reduces taxable pay by £XXX"
}
```

### PayslipDeductionSection

```typescript
interface PayslipDeductionSection {
  title: string;
  items: DeductionDisplayItem[];
  sectionTotal: Decimal;
}

interface DeductionDisplayItem {
  description: string;
  reference: string | null;
  amount: Decimal;
  ytd: Decimal;
  notes: string | null;
}
```

### YTDUpdateInput

```typescript
interface YTDUpdateInput {
  employeeId: string;
  taxYear: string;               // e.g., "2024-25"
  deductionTypeId: string;
  employeeDeductionId: string;
  periodAmount: Decimal;
  payPeriodId: string;
  payRunId: string;
}
```

### YTDRecord

```typescript
interface YTDDeductionRecord {
  employeeId: string;
  taxYear: string;
  deductionTypeId: string;
  employeeDeductionId: string;
  ytdAmount: Decimal;
  lastPayPeriodId: string;
  lastPayRunId: string;
  lastUpdated: Date;
}
```

### DeductionHistoryRecord

```typescript
interface DeductionHistoryRecord {
  id: string;
  employeeDeductionId: string;
  payRunId: string;
  payPeriodId: string;
  amount: Decimal;
  taxablePayBefore: Decimal;
  taxablePayAfter: Decimal;
  nicablePayBefore: Decimal;
  nicablePayAfter: Decimal;
  netPayBefore: Decimal;         // For AEOs
  netPayAfter: Decimal;          // For AEOs
  ytdAmountAfter: Decimal;
  createdAt: Date;
}
```

### Payslip Integration Flow

```
Payslip Generation:
1. Get calculation results (tax, NIC, sacrifices, AEOs)
2. Format pre-tax deductions (salary sacrifice)
3. Format post-tax deductions (AEOs)
4. Calculate section totals
5. Add to payslip data structure
6. Record deduction history (transaction)
7. Update YTD accumulators
```

## Business Rules & Invariants

1. **YTD amounts reset at tax year end** — new tax year starts from zero
2. **Deduction history is immutable** — once recorded, cannot be changed (reversal creates offsetting record)
3. **Payslip reversal restores YTD** — deduction amount subtracted from YTD totals
4. **AEO references must appear on payslip** — legal requirement for transparency
5. **Salary sacrifice impact shown** — display how much taxable/NICable pay reduced
6. **YTD carried forward on P45** — leaver processing includes deduction YTD

## Edge Cases

1. **Zero deduction period** — YTD unchanged, history record with zero amount
2. **Tax year boundary** — YTD resets, final P60 captures closing totals
3. **Employee leaves mid-year** — P45 includes deduction YTD summary
4. **Payslip reversal** — YTD decremented, offsetting history record created
5. **Multiple pay runs same period** — YTD accumulates across runs (rare but supported)
6. **Deduction type changed mid-year** — separate YTD tracking per type
7. **Negative YTD (reversal scenario)** — floored at zero, warning logged

## Tests

### deduction-formatter.test.ts
- `salary sacrifice formatted with impact description`
- `pension sacrifice shows policy number`
- `aeo shows reference number and type`
- `multiple deductions grouped by section`
- `section totals calculated correctly`
- `ytd amounts displayed correctly`

### ytd-manager.test.ts
- `ytd accumulates period amounts`
- `ytd reset for new tax year`
- `ytd decremented on reversal`
- `ytd never goes negative`
- `concurrent updates handled safely`

### deduction-ytd-payslip.test.ts
- `full payslip includes deduction sections`
- `deduction history recorded on generation`
- `ytd updated after payslip generation`
- `payslip reversal restores ytd`
- `calculation trace includes deduction details`

## Verification

```bash
# Lint check
npm run lint

# Type check
npm run typecheck

# Run deduction YTD and payslip tests
npm run test -- payslip/deduction-formatter
npm run test -- calculations/ytd-manager
npm run test -- deduction-ytd-payslip

# Build check
npm run build
```

## Source Sections

- epic-plan.md § Scope & Deliverables → Payslip generation with calculation trace, Year-end P60 generation
- epic-plan.md § Data Contracts → PayCalculationResult (ytd_after field)
