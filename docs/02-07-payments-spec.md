# Module: Payments

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend Engineering Team
**Priority:** P1 (Strongly Recommended V1)

---

## Purpose

Convert approved payroll outputs into controlled payment instructions, execute payments through integrated providers (Modulr first), manage approval chains, reconcile outcomes, and maintain full audit evidence with segregation from payroll processing.

---

## User Journeys

### Journey 1: Generate Payment Batch
1. **Payroll Manager** views approved payroll
2. System shows "Create Payment Batch" option
3. Manager selects payment provider (Modulr)
4. System generates payment intents for:
   - Employee net pays
   - HMRC liabilities
   - Pension contributions
   - Third-party deductions
5. System validates bank details
6. System displays batch summary with totals
7. Manager reviews and submits for approval

### Journey 2: Approve Payment Batch
1. **Payment Approver** receives notification
2. Approver views batch details and payroll linkage
3. System requires MFA verification
4. Approver confirms funding account has sufficient balance
5. Approver approves batch
6. System submits to Modulr
7. System tracks submission status

### Journey 3: Monitor Payment Status
1. **Payroll Manager** views payment dashboard
2. System shows payment statuses: pending/submitted/executed/failed
3. System polls Modulr for status updates
4. Manager clicks failed payment to view error
5. Manager initiates retry or manual process

### Journey 4: Reconcile Payments
1. **Finance/Bureau Owner** runs reconciliation report
2. System matches executed payments to payroll
3. System identifies discrepancies
4. System generates accounting journal entries
5. System exports reconciliation data

---

## Data Models

### Entity: PaymentBatch
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| pay_run_id | UUID | Yes | FK to PayRun |
| employer_id | UUID | Yes | FK to Employer |
| provider | Enum | Yes | modulr/bacs_manual |
| status | Enum | Yes | draft/pending_approval/approved/submitted/executing/executed/failed |
| total_amount | Money | Yes | Total batch value |
| payment_count | Int | Yes | Number of payments |
| currency | String | Yes | GBP |
| created_by | UUID | Yes | FK to User |
| approved_by | UUID | No | FK to User |
| approved_at | DateTime | No | Approval timestamp |
| submitted_at | DateTime | No | Submission to provider |
| provider_batch_id | String | No | Provider's reference |

### Entity: PaymentInstruction
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| batch_id | UUID | Yes | FK to PaymentBatch |
| beneficiary_type | Enum | Yes | employee/hmrc/pension/third_party |
| beneficiary_id | UUID | Yes | Reference to beneficiary |
| beneficiary_name | String | Yes | Payee name |
| sort_code | String | Yes | Bank sort code |
| account_number | String | Yes | Bank account number |
| amount | Money | Yes | Payment amount |
| reference | String | Yes | Payment reference |
| status | Enum | Yes | pending/submitted/executed/failed/returned |
| provider_payment_id | String | No | Provider's payment reference |
| executed_at | DateTime | No | Execution timestamp |
| failure_reason | String | No | If failed |

### Entity: BankAccountChange
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employee_id | UUID | Yes | FK to Employee |
| old_sort_code | String | Yes | Previous sort code |
| old_account_number | String | Yes | Previous account |
| new_sort_code | String | Yes | New sort code |
| new_account_number | String | Yes | New account |
| requested_by | UUID | Yes | Who requested (employee/bureau) |
| requested_at | DateTime | Yes | Request timestamp |
| approved_by | UUID | No | Who approved |
| approved_at | DateTime | No | Approval timestamp |
| effective_date | Date | Yes | When change takes effect |
| status | Enum | Yes | pending/approved/rejected |

---

## API Contracts

### POST /api/v1/pay-runs/{id}/create-payment-batch
Create payment batch from approved payroll.

**Input:**
```json
{
  "provider": "modulr",
  "payment_types": ["employee_net", "hmrc", "pension"]
}
```

**Output:**
```json
{
  "batch_id": "uuid",
  "status": "draft",
  "summary": {
    "total_payments": 45,
    "total_amount": 98500.00,
    "employee_payments": 42,
    "hmrc_payment": 1,
    "pension_payments": 2
  },
  "validation_results": {
    "valid": 44,
    "invalid": 1,
    "errors": [
      {
        "employee_id": "uuid",
        "error": "Invalid sort code format"
      }
    ]
  }
}
```

### POST /api/v1/payment-batches/{id}/approve
Approve payment batch for execution.

**Input:**
```json
{
  "mfa_code": "123456",
  "notes": "Funded and approved"
}
```

**Output:**
```json
{
  "batch_id": "uuid",
  "status": "approved",
  "approved_by": "uuid",
  "approved_at": "2026-04-29T09:00:00Z"
}
```

### POST /api/v1/payment-batches/{id}/submit
Submit batch to payment provider.

**Output:**
```json
{
  "batch_id": "uuid",
  "status": "submitted",
  "provider_reference": "MOD-2026-12345",
  "estimated_execution": "2026-04-29T14:00:00Z"
}
```

### GET /api/v1/payment-batches/{id}/status
Get batch execution status.

**Output:**
```json
{
  "batch_id": "uuid",
  "status": "executed",
  "provider_status": "completed",
  "payments": {
    "total": 45,
    "executed": 44,
    "failed": 1
  },
  "executed_at": "2026-04-29T14:05:00Z"
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| Modulr API | Payment execution | REST API |
| Core Payroll | Net pay amounts | Events |
| Pension Module | Contribution amounts | Events |
| HMRC Module | Liability amounts | Events |
| Notification Service | Status alerts | Events |

---

## Non-Functional Requirements

### Security
- Dual approval required for payments > threshold
- MFA required for all payment approvals
- Bank detail changes require separate approval
- Encryption of all bank account data
- Full audit trail from payroll to payment

### Reliability
- Idempotent payment submissions
- Duplicate detection via idempotency keys
- Automatic retry for transient failures
- Manual fallback process documented

### Compliance
- Segregation of duties (processor ≠ approver)
- Evidence retention for 6 years
- Reconciliation reports
- Anomaly detection (new payees, large changes)
