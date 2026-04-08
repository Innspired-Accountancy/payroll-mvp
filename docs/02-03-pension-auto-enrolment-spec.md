# Module: Pension & Auto-Enrolment

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend Engineering Team
**Priority:** P0 (Critical Path)

---

## Purpose

Assess workers for auto-enrolment eligibility, manage enrolment workflows, calculate contributions, generate statutory communications, submit to pension providers (NEST first), and retain compliance evidence per The Pensions Regulator requirements.

---

## User Journeys

### Journey 1: Worker Assessment
1. **Payroll Processor** runs payroll calculation
2. System assesses each worker against age/earnings criteria
3. System categorizes: Eligible Jobholder / Non-Eligible Jobholder / Entitled Worker
4. System flags Eligible Jobholders requiring enrolment
5. System records assessment result with timestamp (evidence)
6. System triggers enrolment workflow for new eligibles

### Journey 2: Enrolment and Communications
1. **System** identifies newly eligible jobholder
2. System generates statutory enrolment letter
3. System sends letter via email/post and records dispatch
4. System creates enrolment record with 1-month opt-out window
5. System includes employee in pension contribution calculation
6. System prepares contribution data for NEST submission

### Journey 3: Opt-Out Processing
1. **Employee** submits opt-out request within 1-month window
2. **Employer Admin** records opt-out in system
3. System calculates refund amount (employee contributions)
4. System processes refund via next payroll
5. System records opt-out with evidence retention flag (4 years)
6. System updates worker status to "opted-out"

### Journey 4: Pension Submission to NEST
1. **Payroll Manager** approves pay run with pension contributions
2. System generates NEST contribution schedule
3. System submits via NEST web services API
4. System polls for acceptance/rejection
5. System reconciles submitted vs accepted records
6. System flags discrepancies for manual review

### Journey 5: Cyclical Re-Enrolment
1. **System** tracks 3-year anniversary of employer duties start date
2. System generates re-enrolment assessment report
3. System identifies previously opted-out employees to re-assess
4. System re-enrols eligible employees
5. System generates re-declaration of compliance reminder (5-month window)

---

## Data Models

### Entity: PensionAssessment
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employee_id | UUID | Yes | FK to Employee |
| pay_period_id | UUID | Yes | FK to PayPeriod |
| assessment_date | Date | Yes | Date of assessment |
| age_at_assessment | Int | Yes | Employee age |
| earnings_in_period | Money | Yes | Pensionable earnings |
| worker_category | Enum | Yes | eligible/non_eligible/entitled |
| must_enrol | Boolean | Yes | Auto-enrolment required |
| can_opt_in | Boolean | Yes | Can request opt-in |
| can_join | Boolean | Yes | Can request to join |
| assessment_reason | Enum | Yes | new_employee/postponement_end/periodic |

### Entity: PensionEnrolment
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employee_id | UUID | Yes | FK to Employee |
| scheme_id | UUID | Yes | FK to PensionScheme |
| enrolment_date | Date | Yes | Date enrolled |
| enrolment_reason | Enum | Yes | automatic/opt_in/join_request |
| opt_out_deadline | Date | Yes | 1 month from enrolment |
| status | Enum | Yes | enrolled/opted_out/opted_in/ceased |
| communication_sent | Boolean | Yes | Enrolment letter sent |
| communication_date | DateTime | No | When letter sent |
| opt_out_received | Boolean | No | Opt-out received |
| opt_out_date | Date | No | Date opted out |
| refund_processed | Boolean | No | Refund processed |

### Entity: PensionContribution
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| payslip_id | UUID | Yes | FK to Payslip |
| enrolment_id | UUID | Yes | FK to PensionEnrolment |
| earnings_basis | Enum | Yes | qualifying_earnings/certified |
| qualifying_earnings | Money | Yes | Earnings used for calculation |
| employee_contribution | Money | Yes | Employee deduction |
| employer_contribution | Money | Yes | Employer contribution |
| total_contribution | Money | Yes | Sum of both |
| relief_method | Enum | Yes | relief_at_source/net_pay |

### Entity: PensionScheme
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| provider | Enum | Yes | nest/other |
| scheme_name | String | Yes | Display name |
| employer_reference | String | Yes | Provider-specific reference |
| employee_contribution_rate | Decimal | Yes | % or fixed amount |
| employer_contribution_rate | Decimal | Yes | % or fixed amount |
| earnings_basis | Enum | Yes | qualifying/banded/total |
| relief_method | Enum | Yes | relief_at_source/net_pay |
| api_config | JSONB | No | Provider API credentials/settings |

### Entity: PensionProviderSubmission
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| scheme_id | UUID | Yes | FK to PensionScheme |
| pay_period_id | UUID | Yes | FK to PayPeriod |
| provider | Enum | Yes | nest |
| submission_type | Enum | Yes | contributions/enrolment/opt_out |
| status | Enum | Yes | pending/submitted/accepted/partial/rejected |
| submission_payload | JSONB | Yes | Provider-specific payload |
| provider_reference | String | No | Provider's reference ID |
| submitted_at | DateTime | No | Submission timestamp |
| response_data | JSONB | No | Provider response |

---

## API Contracts

### GET /api/v1/employees/{id}/pension-assessments
Get pension assessment history for employee.

**Output:**
```json
{
  "current_category": "eligible_jobholder",
  "current_enrolment": {
    "id": "uuid",
    "scheme_name": "NEST Workplace Pension",
    "status": "enrolled",
    "enrolment_date": "2026-01-15",
    "opt_out_deadline": "2026-02-15"
  },
  "assessments": [
    {
      "pay_period": "April 2026",
      "category": "eligible_jobholder",
      "earnings": 2500.00,
      "must_enrol": true
    }
  ]
}
```

### POST /api/v1/pension-enrolments/{id}/opt-out
Record employee opt-out.

**Input:**
```json
{
  "opt_out_date": "2026-01-20",
  "evidence_reference": "OPT-001",
  "notes": "Employee submitted signed opt-out form"
}
```

**Output:**
```json
{
  "id": "uuid",
  "status": "opted_out",
  "opt_out_date": "2026-01-20",
  "refund_due": 45.00,
  "refund_pay_period": "May 2026"
}
```

### POST /api/v1/pay-periods/{id}/submit-pension-contributions
Submit pension contributions to provider.

**Output:**
```json
{
  "submission_id": "uuid",
  "provider": "nest",
  "status": "submitted",
  "total_contributions": 12500.00,
  "employee_count": 42,
  "provider_reference": "NEST-2026-12345"
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| NEST Web Services | Pension submissions | REST API |
| Core Payroll | Earnings data for assessment | Events |
| Document Service | Communication letters | Internal API |
| Notification Service | Enrolment alerts | Events |

---

## Non-Functional Requirements

### Compliance
- Assess every worker every pay reference period
- Record all assessments immutably (6-year retention)
- Opt-out records retained for 4 years
- Communications sent within 6-week statutory window
- Support cyclical re-enrolment every 3 years

### Accuracy
- Correct qualifying earnings calculation
- Proper relief method application (RAS vs net pay)
- Accurate contribution rates per scheme

### Provider Abstraction
- Clean adapter pattern for multiple providers
- NEST as first implementation
- File-based fallback for all providers
- Canonical contribution schedule model
