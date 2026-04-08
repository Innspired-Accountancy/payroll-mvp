# Module: Core Payroll Engine

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend Engineering Team
**Priority:** P0 (Critical Path)

---

## Purpose

Calculate accurate UK gross-to-net payroll for all standard scenarios across multiple pay frequencies, producing data for RTI, pensions, payments, documents, and reporting with full calculation transparency and immutable snapshots.

This is the foundational module — all other modules depend on its outputs.

---

## User Journeys

### Journey 1: Create and Calculate a Pay Run
1. **Payroll Processor** navigates to employer's payroll schedule
2. System creates new pay period with period dates, pay date, tax period mapping
3. Processor enters variable data (hours, overtime, bonuses, leave)
4. System validates data completeness (tax codes, NI categories, bank details)
5. Processor runs calculation
6. System generates draft payslips with full calculation breakdown
7. System flags any warnings (NMW alerts, negative net pay, etc.)

### Journey 2: Review and Approve Payroll
1. **Reviewer/Approver** views calculated pay run
2. System displays variance against previous period
3. System shows "explain this calculation" view with all inputs, thresholds, rates
4. Reviewer approves or rejects with comments
5. On approval, system locks pay run and creates immutable snapshot
6. System triggers downstream workflows (HMRC, pension, payment intents)

### Journey 3: Handle Corrections
1. **Payroll Manager** identifies need to correct approved payroll
2. System requires reopen authorization with reason capture
3. System creates new calculation version (preserves original)
4. System analyzes downstream impact (resubmission, pension adjustments, payment regeneration)
5. Corrected payroll goes through approval workflow again

---

## Data Models

### Entity: PayPeriod
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| schedule_id | UUID | Yes | FK to PayrollSchedule |
| period_start | Date | Yes | Start of pay period |
| period_end | Date | Yes | End of pay period |
| pay_date | Date | Yes | Date employees are paid |
| tax_period | Int | Yes | HMRC tax week/month number |
| tax_year | String | Yes | "2026-27" format |
| status | Enum | Yes | draft/calculated/reviewed/approved/finalised/reopened |
| created_by | UUID | Yes | FK to User |
| created_at | DateTime | Yes | Timestamp |
| approved_by | UUID | No | FK to User |
| approved_at | DateTime | No | Timestamp |
| snapshot_hash | String | No | Calculation snapshot hash |

### Entity: PayRun
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| pay_period_id | UUID | Yes | FK to PayPeriod |
| version | Int | Yes | Calculation version (1=original) |
| status | Enum | Yes | draft/final/superseded |
| calculated_at | DateTime | Yes | Calculation timestamp |
| calculation_engine_version | String | Yes | Semantic version |
| tax_year_config_version | String | Yes | Tax year rules version |
| total_gross | Money | Yes | Sum of all gross pay |
| total_tax | Money | Yes | Sum of PAYE deductions |
| total_nic_employee | Money | Yes | Sum of employee NIC |
| total_nic_employer | Money | Yes | Sum of employer NIC |
| total_net | Money | Yes | Sum of net pay |

### Entity: Payslip
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| pay_run_id | UUID | Yes | FK to PayRun |
| employee_id | UUID | Yes | FK to Employee |
| employment_id | UUID | Yes | FK to Employment (for multiple employments) |
| gross_pay | Money | Yes | Total gross pay |
| taxable_pay | Money | Yes | Pay subject to income tax |
| tax_deducted | Money | Yes | PAYE deduction |
| nic_employee | Money | Yes | Employee NIC |
| nic_employer | Money | Yes | Employer NIC |
| pension_contribution_employee | Money | No | Employee pension deduction |
| pension_contribution_employer | Money | No | Employer pension contribution |
| other_deductions | Money | No | Sum of other deductions |
| net_pay | Money | Yes | Final net pay |
| ytd_taxable_pay | Money | Yes | Year-to-date taxable pay |
| ytd_tax_deducted | Money | Yes | Year-to-date tax deducted |
| calculation_trace | JSONB | Yes | Full calculation breakdown |

### Entity: PayElement (Earnings/Deductions)
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| payslip_id | UUID | Yes | FK to Payslip |
| type | Enum | Yes | earning/deduction |
| category | String | Yes | salary/hourly/overtime/bonus/pension/tax/nic/etc |
| description | String | Yes | Display description |
| amount | Money | Yes | Amount |
| taxable | Boolean | Yes | Subject to income tax |
| nicable | Boolean | Yes | Subject to NIC |
| pensionable | Boolean | Yes | Counts toward pensionable earnings |

---

## API Contracts

### POST /api/v1/employers/{id}/pay-periods
Create a new pay period for an employer's payroll schedule.

**Input:**
```json
{
  "schedule_id": "uuid",
  "period_start": "2026-04-01",
  "period_end": "2026-04-30",
  "pay_date": "2026-04-30"
}
```

**Output:**
```json
{
  "id": "uuid",
  "status": "draft",
  "period_start": "2026-04-01",
  "period_end": "2026-04-30",
  "pay_date": "2026-04-30",
  "tax_period": 1,
  "tax_year": "2026-27"
}
```

**Errors:**
- 400: Invalid dates, overlapping period
- 403: Insufficient permissions
- 409: Period already exists for this tax period

### POST /api/v1/pay-periods/{id}/calculate
Calculate payroll for the pay period.

**Input:**
```json
{
  "employee_data": [
    {
      "employee_id": "uuid",
      "hours_worked": 160,
      "overtime_hours": 8,
      "bonus": 500.00,
      "ssp_days": 3
    }
  ]
}
```

**Output:**
```json
{
  "pay_run_id": "uuid",
  "status": "calculated",
  "total_employees": 45,
  "total_gross": 125000.00,
  "total_net": 98500.00,
  "warnings": [
    {
      "employee_id": "uuid",
      "type": "nmw_check",
      "message": "Employee hourly rate may fall below NMW"
    }
  ]
}
```

### POST /api/v1/pay-runs/{id}/approve
Approve a calculated pay run.

**Input:**
```json
{
  "notes": "Approved after variance review"
}
```

**Output:**
```json
{
  "id": "uuid",
  "status": "approved",
  "approved_by": "uuid",
  "approved_at": "2026-04-28T10:30:00Z",
  "snapshot_hash": "sha256:abc123..."
}
```

### GET /api/v1/payslips/{id}/calculation-trace
Retrieve full calculation breakdown for a payslip.

**Output:**
```json
{
  "inputs": {
    "tax_code": "1257L",
    "ni_category": "A",
    "gross_salary": 3000.00
  },
  "steps": [
    {
      "step": "taxable_pay",
      "formula": "gross - tax_free",
      "inputs": {"gross": 3000.00, "tax_free": 1047.50},
      "output": 1952.50
    },
    {
      "step": "tax_calculation",
      "formula": "taxable * rate",
      "inputs": {"taxable": 1952.50, "rate": 0.20},
      "output": 390.50
    }
  ],
  "final_result": {
    "tax": 390.50,
    "nic_employee": 245.32,
    "net_pay": 2364.18
  }
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| TaxYearConfig Service | Annual rates/thresholds | Internal service, versioned config |
| HMRC Submissions Module | Trigger FPS generation | Event-driven (pay_run.approved) |
| Pension Module | Trigger AE assessment | Event-driven (pay_run.approved) |
| Payments Module | Create payment intents | Event-driven (pay_run.approved) |
| Audit Service | Log all changes | Async event logging |

---

## Non-Functional Requirements

### Performance
- Payroll calculation for 100 employees: <5 seconds
- Variance report generation: <2 seconds
- Payslip PDF generation: <1 second per payslip

### Accuracy
- 100% match with HMRC reference calculations for standard scenarios
- Deterministic: same inputs always produce same outputs
- All calculations immutable after finalisation

### Security
- All payroll data encrypted at rest
- Field-level encryption for sensitive fields (bank details)
- Comprehensive audit trail for all changes
- Role-based access: processors only see assigned employers

### Compliance
- Support for current tax year plus 2 prior years simultaneously
- Annual uprating via configuration, not code changes
- Week 53 handling per HMRC rules
- Scottish and Welsh tax rate support
