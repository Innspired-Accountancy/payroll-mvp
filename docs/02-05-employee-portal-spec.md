# Module: Employee Portal

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Frontend Team
**Priority:** P1 (Required for V1)

---

## Purpose

Provide secure self-service access for employees to view payslips and P60s, request leave, and manage personal details with strict self-only data visibility and approval workflows for sensitive changes.

---

## User Journeys

### Journey 1: View Payslip
1. **Employee** logs in to portal (email/password + MFA)
2. System displays dashboard with latest payslip
3. Employee clicks to view payslip details
4. System shows itemized payslip with earnings, deductions, net pay
5. Employee downloads PDF or views online
6. Employee views year-to-date totals

### Journey 2: Access P60
1. **Employee** navigates to Documents section
2. System lists available documents (payslips, P60s, P45s)
3. Employee selects tax year
4. System displays P60 for selected year
5. Employee downloads PDF

### Journey 3: Request Leave
1. **Employee** navigates to Leave section
2. System displays current leave balance
3. Employee clicks "Request Leave"
4. Employee selects dates and leave type
5. System checks for conflicts
6. Employee submits request
7. System notifies employer approver
8. Employee receives notification when approved/rejected

### Journey 4: Update Bank Details
1. **Employee** navigates to Profile
2. Employee clicks "Update Bank Details"
3. Employee enters new sort code and account number
4. System validates format
5. System submits for employer/bureau approval
6. Employee sees "pending approval" status
7. System notifies when approved/rejected

---

## Data Models

### Entity: EmployeePortalSession
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employee_id | UUID | Yes | FK to Employee |
| login_time | DateTime | Yes | Session start |
| logout_time | DateTime | No | Session end |
| ip_address | String | Yes | Source IP |
| mfa_verified | Boolean | Yes | MFA completed |

### Entity: EmployeeDocumentAccess
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employee_id | UUID | Yes | FK to Employee |
| document_type | Enum | Yes | payslip/p60/p45 |
| document_id | UUID | Yes | Reference to document |
| accessed_at | DateTime | Yes | Access timestamp |
| ip_address | String | Yes | Source IP |

### Entity: PersonalDetailChangeRequest
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employee_id | UUID | Yes | FK to Employee |
| change_type | Enum | Yes | address/phone/bank_details |
| old_value | JSONB | Yes | Previous value |
| new_value | JSONB | Yes | Requested value |
| status | Enum | Yes | pending/approved/rejected |
| requested_at | DateTime | Yes | Timestamp |
| approved_by | UUID | No | FK to User who approved |
| approved_at | DateTime | No | Approval timestamp |

---

## API Contracts

### GET /api/v1/portal/payslips
Get employee's payslip history.

**Output:**
```json
{
  "employee": {
    "name": "John Smith",
    "employee_number": "EMP001"
  },
  "payslips": [
    {
      "id": "uuid",
      "pay_date": "2026-04-30",
      "period": "April 2026",
      "gross_pay": 3500.00,
      "net_pay": 2750.00,
      "download_url": "/api/v1/payslips/uuid/download"
    }
  ],
  "ytd_summary": {
    "taxable_pay": 14000.00,
    "tax_paid": 2800.00,
    "nic_paid": 840.00
  }
}
```

### GET /api/v1/portal/payslips/{id}
Get detailed payslip.

**Output:**
```json
{
  "id": "uuid",
  "employer_name": "ABC Ltd",
  "pay_date": "2026-04-30",
  "period": "April 2026",
  "tax_code": "1257L",
  "ni_number": "AB123456C",
  "earnings": [
    {"description": "Basic Salary", "amount": 3000.00},
    {"description": "Bonus", "amount": 500.00}
  ],
  "deductions": [
    {"description": "Income Tax", "amount": 390.50},
    {"description": "National Insurance", "amount": 245.32},
    {"description": "Pension", "amount": 150.00}
  ],
  "totals": {
    "gross": 3500.00,
    "deductions": 785.82,
    "net": 2714.18
  }
}
```

### POST /api/v1/portal/leave-requests
Submit leave request.

**Input:**
```json
{
  "leave_type": "annual_leave",
  "start_date": "2026-05-10",
  "end_date": "2026-05-14",
  "days": 5,
  "reason": "Holiday"
}
```

**Output:**
```json
{
  "id": "uuid",
  "status": "pending",
  "submitted_at": "2026-04-28T10:00:00Z",
  "balance_after": 15
}
```

### POST /api/v1/portal/bank-change-request
Request bank detail change.

**Input:**
```json
{
  "account_name": "John Smith",
  "sort_code": "12-34-56",
  "account_number": "12345678"
}
```

**Output:**
```json
{
  "request_id": "uuid",
  "status": "pending_approval",
  "message": "Your bank detail change is pending approval from your employer"
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| Core Payroll | Payslip data | Internal API |
| Leave Module | Leave balances | Internal API |
| Notification Service | Email alerts | Events |
| Document Store | PDF generation | Internal API |

---

## Non-Functional Requirements

### Security
- MFA required for all employees (TOTP or SMS)
- Session timeout after 30 minutes inactivity
- Strict self-only data visibility (no cross-employee access)
- Bank detail changes require employer approval
- All access logged with IP and timestamp

### Accessibility
- WCAG 2.1 AA compliance
- Mobile-responsive design
- Screen reader compatible
- Keyboard navigation support

### Performance
- Page load <2 seconds
- PDF download <5 seconds
- Available 24/7 (99.9% uptime)
