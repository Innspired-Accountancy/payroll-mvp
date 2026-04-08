# Slice b: Payroll Calculation Orchestrator

**Story:** story-08-pay-run-lifecycle
**Epic:** epic-01-core-payroll
**Effort:** L
**Dependencies:** slice-a

---

## Goal

Implement the payroll calculation orchestrator that coordinates tax, NIC, statutory payments, and deductions for all employees in a pay run.

---

## Decision Checklist

- [x] Orchestration: Sequential calculation per employee
- [x] Components: Tax calc, NIC calc, statutory calc, deductions
- [x] Aggregation: Sum totals for pay run
- [x] Warnings: NMW check, negative net, missing data
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Calculation orchestration
- 02-01-core-payroll-spec.md — API Contracts

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/orchestrator.ts` | create | Calculation orchestrator |
| `src/server/routers/calculate-payroll.ts` | create | Calculation tRPC |
| `src/tests/calculations/orchestrator.test.ts` | create | Orchestrator tests |

---

## Responsibilities

1. Load all employees for pay period
2. Calculate each employee's payslip
3. Aggregate totals for pay run
4. Generate warnings
5. Create payslip records

---

## Contracts

### calculatePayroll
- **Method:** tRPC mutation `payRuns.calculate`
- **Input:** `{ pay_period_id: uuid, employee_data?: EmployeeInput[] }`
- **Output:** PayRunCalculationResult

### EmployeeInput
| Field | Type | Description |
|-------|------|-------------|
| employee_id | uuid | Employee FK |
| hours_worked | Decimal | Variable hours |
| overtime_hours | Decimal | Overtime |
| bonus | Decimal | Bonus amount |
| ssp_days | int | SSP days to claim |

### PayRunCalculationResult
| Field | Type | Description |
|-------|------|-------------|
| pay_run_id | uuid | Created/updated pay run |
| total_employees | int | Count processed |
| total_gross | Decimal | Sum of gross pay |
| total_tax | Decimal | Sum of tax |
| total_nic_employee | Decimal | Sum employee NIC |
| total_nic_employer | Decimal | Sum employer NIC |
| total_net | Decimal | Sum of net pay |
| warnings | Warning[] | Generated warnings |

---

## Business Rules & Invariants

1. Calculate in order: gross → salary sacrifice → taxable → tax → NIC → statutory → deductions → net
2. Track YTD for cumulative calculations
3. Each employee calculated independently
4. Warnings don't block calculation
5. Failed employee calculations logged but don't stop others

---

## Edge Cases

1. **Employee with no data** — Use default/zero values
2. **Leaver in period** — Calculate up to leaving date
3. **New starter** — Calculate from start date
4. **Zero hours** — Handle variable pay

---

## Tests

### orchestrator.test.ts
- Calculate pay run for multiple employees
- Aggregation totals correct
- Warning generation (NMW, negative net)
- New starter calculation
- Leaver calculation

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Calculation Engine Design
