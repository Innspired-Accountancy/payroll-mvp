# Module: CIS (Construction Industry Scheme)

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend Engineering Team
**Priority:** P2 (Strongly Recommended V1)

---

## Purpose

Manage contractor-side CIS obligations including subcontractor verification with HMRC, deduction calculations, monthly CIS300 returns, deduction statements, and CIS suffered offset tracking for construction sector clients.

---

## User Journeys

### Journey 1: Verify Subcontractor
1. **CIS Operator** navigates to subcontractor register
2. Operator clicks "Verify New Subcontractor"
3. System displays verification form
4. Operator enters UTR, NI number, name
5. System submits to HMRC CIS Online
6. System displays result: gross/20%/30%/unmatched
7. System records verification with reference number
8. System blocks payments to unverified subcontractors

### Journey 2: Record CIS Payment
1. **CIS Operator** processes payment to subcontractor
2. Operator enters gross amount, materials cost
3. System calculates labour amount (gross - materials)
4. System applies verification rate to labour
5. System calculates deduction amount
6. System records net payment due
7. System accumulates for monthly return

### Journey 3: Submit Monthly Return
1. **CIS Operator** navigates to monthly returns
2. System shows current tax month summary
3. System lists all payments made to subcontractors
4. System validates no negative values
5. Operator reviews and confirms
6. System generates CIS300 XML
7. System submits to HMRC by 19th deadline
8. System tracks acknowledgement

### Journey 4: Generate Deduction Statement
1. **System** identifies need for deduction statements (14-day deadline)
2. System generates statement for each subcontractor with deductions
3. System emails statement to subcontractor
4. System records distribution evidence
5. Subcontractor can also download via portal

---

## Data Models

### Entity: Subcontractor
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer (contractor) |
| utr | String | Yes | Unique Taxpayer Reference |
| ni_number | String | No | National Insurance number |
| company_number | String | No | If limited company |
| trading_name | String | Yes | Business name |
| entity_type | Enum | Yes | sole_trader/partnership/company |
| address | JSONB | Yes | Address |
| verification_status | Enum | Yes | unverified/verified/unmatched |
| verification_rate | Enum | No | gross/net_20/net_30 |
| created_at | DateTime | Yes | Timestamp |

### Entity: SubcontractorVerification
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| subcontractor_id | UUID | Yes | FK to Subcontractor |
| verification_date | Date | Yes | Date of verification |
| verification_number | String | Yes | HMRC verification reference |
| outcome | Enum | Yes | gross/net_20/net_30/unmatched |
| verified_by | UUID | Yes | FK to User |
| valid_until | Date | No | Re-verification required after |

### Entity: CISPayment
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| subcontractor_id | UUID | Yes | FK to Subcontractor |
| tax_month | Int | Yes | CIS tax month |
| tax_year | String | Yes | "2026-27" |
| payment_date | Date | Yes | Date paid |
| gross_amount | Money | Yes | Total payment |
| materials_cost | Money | Yes | Materials (not subject to CIS) |
| labour_amount | Money | Yes | Gross - materials |
| deduction_rate | Decimal | Yes | 0/20/30 |
| deduction_amount | Money | Yes | CIS deduction |
| net_amount | Money | Yes | Amount paid |
| verification_id | UUID | Yes | FK to verification used |

### Entity: CISReturn
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| tax_month | Int | Yes | Month of return |
| tax_year | String | Yes | Tax year |
| status | Enum | Yes | draft/submitted/accepted |
| nil_return | Boolean | Yes | No payments this month |
| total_payments | Int | Yes | Number of subcontractors |
| total_gross | Money | Yes | Sum of gross amounts |
| total_deductions | Money | Yes | Sum of deductions |
| submitted_at | DateTime | No | Submission timestamp |
| hmrc_reference | String | No | HMRC acknowledgement |

---

## API Contracts

### POST /api/v1/subcontractors/verify
Verify subcontractor with HMRC.

**Input:**
```json
{
  "utr": "1234567890",
  "ni_number": "AB123456C",
  "name": "John Smith",
  "entity_type": "sole_trader"
}
```

**Output:**
```json
{
  "verification_id": "uuid",
  "status": "verified",
  "outcome": "net_20",
  "verification_number": "V12345678",
  "valid_until": "2028-04-06"
}
```

### POST /api/v1/cis-payments
Record CIS payment.

**Input:**
```json
{
  "subcontractor_id": "uuid",
  "payment_date": "2026-04-15",
  "gross_amount": 5000.00,
  "materials_cost": 1000.00
}
```

**Output:**
```json
{
  "payment_id": "uuid",
  "labour_amount": 4000.00,
  "deduction_rate": 20,
  "deduction_amount": 800.00,
  "net_amount": 4200.00
}
```

### POST /api/v1/cis-returns/{id}/submit
Submit monthly return.

**Output:**
```json
{
  "return_id": "uuid",
  "status": "submitted",
  "hmrc_reference": "CIS300-12345",
  "submitted_at": "2026-04-19T10:00:00Z"
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| HMRC CIS Online | Verification and returns | XML API |
| Core Payroll | Dual-status employee handling | Internal API |
| Document Service | Deduction statements | Internal API |
| EPS Module | CIS suffered offset | Events |

---

## Non-Functional Requirements

### Compliance
- Verification before first payment (mandatory)
- Re-verification if not on return in 2 tax years
- CIS300 submitted by 19th of following month
- Deduction statements within 14 days
- No negative values in returns
- CIS suffered tracking for EPS

### Accuracy
- Correct labour/materials split
- Correct rate application
- Accurate deduction calculations
