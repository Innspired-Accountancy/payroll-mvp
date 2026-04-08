# Module: HMRC Submissions

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend Engineering Team
**Priority:** P0 (Critical Path)
**Compliance Risk:** HIGH

---

## Purpose

Generate, validate, submit, poll, and evidence HMRC-facing payroll submissions including RTI FPS/EPS, year-end final submissions, and corrections. Ensure compliance with HMRC technical specifications and retention requirements.

---

## User Journeys

### Journey 1: Submit FPS for Pay Run
1. **Payroll Manager** views approved pay run
2. System auto-generates FPS XML from payroll data
3. System validates against HMRC schema and business rules
4. Manager reviews validation results (warnings/blockers)
5. Manager submits FPS to HMRC
6. System queues submission, assigns correlation ID
7. System polls for acknowledgement
8. System updates submission status (accepted/rejected)
9. System stores submission payload and response as evidence

### Journey 2: Handle FPS Rejection
1. System receives rejection from HMRC
2. System parses error codes and maps to employee/field
3. System notifies Payroll Manager of rejection
4. Manager views error details linked to payroll data
5. Manager corrects data and resubmits
6. System maintains submission chain (original + correction)

### Journey 3: Submit EPS for Adjustments
1. **Payroll Manager** navigates to EPS section for PAYE scheme
2. System shows recoverable amounts (SSP, SMP, NIC compensation)
3. System shows Employment Allowance status
4. Manager reviews and confirms EPS data
5. System generates EPS XML
6. System submits to HMRC by 19th deadline
7. System tracks acknowledgement

### Journey 4: Year-End Final Submission
1. **Payroll Manager** initiates year-end process
2. System checks all pay periods have been submitted
3. System identifies final pay period for each schedule
4. System flags final FPS with "Final submission for year" indicator
5. System submits final FPS or EPS declaration as appropriate
6. System generates P60s for all employees employed on 5 April
7. System makes P60s available via employee portal by 31 May

---

## Data Models

### Entity: HmrcSubmission
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| paye_scheme_id | UUID | Yes | FK to PAYEScheme |
| type | Enum | Yes | fps/eps |
| tax_year | String | Yes | "2026-27" |
| tax_period | Int | Yes | Tax week/month |
| status | Enum | Yes | draft/validated/submitted/acknowledged/accepted/rejected/resubmitted |
| correlation_id | String | Yes | Unique platform submission ID |
| hmrc_correlation_id | String | No | HMRC's correlation ID |
| payload_xml | Text | Yes | Generated XML payload |
| payload_hash | String | Yes | SHA256 of payload |
| response_xml | Text | No | HMRC response XML |
| submitted_at | DateTime | No | Submission timestamp |
| acknowledged_at | DateTime | No | Acknowledgement timestamp |
| submission_version | Int | Yes | 1=original, 2+=correction |
| parent_submission_id | UUID | No | FK to original if correction |
| late_reason | String | No | Late reporting reason code |
| is_final | Boolean | Yes | Final submission for year |

### Entity: HmrcSubmissionError
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| submission_id | UUID | Yes | FK to HmrcSubmission |
| error_code | String | Yes | HMRC error code |
| error_message | String | Yes | Human-readable message |
| severity | Enum | Yes | warning/blocker |
| employee_id | UUID | No | FK to Employee if employee-specific |
| field_path | String | No | XPath to field in XML |
| resolved | Boolean | Yes | Whether error has been fixed |

---

## API Contracts

### POST /api/v1/pay-runs/{id}/generate-fps
Generate FPS XML for an approved pay run.

**Output:**
```json
{
  "submission_id": "uuid",
  "status": "draft",
  "validation_results": {
    "errors": [],
    "warnings": [
      {
        "code": "W01",
        "message": "Employee has multiple employments - verify tax code allocation"
      }
    ]
  },
  "employee_count": 45,
  "total_taxable_pay": 125000.00,
  "total_tax_deducted": 25000.00
}
```

### POST /api/v1/hmrc-submissions/{id}/submit
Submit FPS/EPS to HMRC.

**Input:**
```json
{
  "submission_type": "fps",
  "use_live_gateway": true
}
```

**Output:**
```json
{
  "submission_id": "uuid",
  "status": "submitted",
  "correlation_id": "CORR-2026-000001",
  "submitted_at": "2026-04-28T14:30:00Z",
  "polling_status": "pending"
}
```

### GET /api/v1/hmrc-submissions/{id}/status
Check submission status.

**Output:**
```json
{
  "submission_id": "uuid",
  "status": "accepted",
  "hmrc_response": {
    "acknowledgement_id": "ACK-12345",
    "accepted_at": "2026-04-28T14:31:15Z",
    "warnings": []
  }
}
```

### POST /api/v1/paye-schemes/{id}/generate-eps
Generate EPS for a tax period.

**Input:**
```json
{
  "tax_year": "2026-27",
  "tax_month": 1,
  "include_recoveries": true,
  "include_allowances": true
}
```

**Output:**
```json
{
  "submission_id": "uuid",
  "status": "draft",
  "recoveries": {
    "ssp_recovered": 500.00,
    "smp_recovered": 2000.00,
    "nic_compensation": 300.00
  },
  "employment_allowance": {
    "claimed": true,
    "amount": 1047.50
  }
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| HMRC Gateway | RTI submissions | XML over HTTPS (PAYE Online) |
| Core Payroll | Source data for submissions | Internal API |
| Document Store | Evidence retention | Internal API |
| Notification Service | Submission alerts | Events |

---

## Non-Functional Requirements

### Reliability
- Idempotent submissions (prevent duplicates)
- Automatic retry for transient failures (max 3 attempts)
- Circuit breaker for HMRC gateway outages
- All submissions logged with full audit trail

### Performance
- FPS generation for 100 employees: <3 seconds
- Submission queuing: <1 second
- Status polling interval: Every 30 seconds for 24 hours

### Compliance
- Retain submission evidence for 6 years minimum
- Support prior tax year corrections
- Schema version tracking per tax year
- Fraud prevention headers per HMRC requirements

### Error Handling
- Categorize errors as retryable vs non-retryable
- Link HMRC errors to specific employee records
- Provide actionable correction guidance
- Maintain submission chain for corrections
