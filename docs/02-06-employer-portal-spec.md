# Module: Employer/Client Portal

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Frontend Team
**Priority:** P1 (Required for V1)

---

## Purpose

Provide client-facing interface for employer admins and managers to submit payroll variable data, approve payrolls, view reports, manage starters/leavers, and collaborate with the bureau while maintaining strict employer-scoped access.

---

## User Journeys

### Journey 1: Submit Variable Pay Data
1. **Employer Admin** receives notification that payroll is open
2. Admin logs in to portal
3. System shows "Payroll Input Required" with deadline
4. Admin enters hours, overtime, bonuses for each employee
5. System validates entries (NMW checks, limits)
6. Admin submits data
7. System notifies bureau processor

### Journey 2: Approve Payroll
1. **Employer Manager** receives notification payroll ready for review
2. Manager logs in and views payroll summary
3. System shows variance vs previous period
4. System displays employee-level summary
5. Manager reviews totals and exceptions
6. Manager approves or rejects with comments
7. System notifies bureau of approval/rejection

### Journey 3: Process Starter
1. **Employer Admin** navigates to Employees
2. Admin clicks "Add New Starter"
3. System displays starter form (HMRC starter checklist)
4. Admin enters personal details, P45 data if available
5. System validates data completeness
6. Admin submits
7. System notifies bureau and adds to next payroll

### Journey 4: View Reports
1. **Employer Manager** navigates to Reports
2. System lists available reports (payroll summary, costing, variance)
3. Manager selects report and date range
4. System generates report
5. Manager views online or downloads PDF/Excel

---

## Data Models

### Entity: EmployerPortalUser
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| email | String | Yes | Login email |
| first_name | String | Yes | First name |
| last_name | String | Yes | Last name |
| role | Enum | Yes | admin/manager/viewer |
| can_approve_payroll | Boolean | Yes | Payroll approval permission |
| can_edit_employees | Boolean | Yes | Employee edit permission |
| mfa_enabled | Boolean | Yes | MFA status |
| last_login | DateTime | No | Last login timestamp |

### Entity: VariablePaySubmission
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| pay_period_id | UUID | Yes | FK to PayPeriod |
| submitted_by | UUID | Yes | FK to EmployerPortalUser |
| submitted_at | DateTime | Yes | Timestamp |
| data | JSONB | Yes | Variable pay data by employee |
| status | Enum | Yes | draft/submitted |

### Entity: PayrollApproval
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| pay_run_id | UUID | Yes | FK to PayRun |
| approver_id | UUID | Yes | FK to EmployerPortalUser |
| approved_at | DateTime | Yes | Timestamp |
| decision | Enum | Yes | approved/rejected |
| comments | Text | No | Approval comments |

---

## API Contracts

### GET /api/v1/employer-portal/dashboard
Get employer dashboard.

**Output:**
```json
{
  "employer": {
    "name": "ABC Ltd",
    "paye_scheme": "123/AB45678"
  },
  "current_payroll": {
    "period": "April 2026",
    "status": "awaiting_approval",
    "pay_date": "2026-04-30",
    "deadline": "2026-04-28T17:00:00Z",
    "days_remaining": 2
  },
  "pending_actions": [
    {
      "type": "payroll_approval",
      "description": "April 2026 payroll awaiting approval",
      "due_date": "2026-04-28"
    }
  ],
  "recent_activity": [
    {
      "type": "payroll_paid",
      "description": "March 2026 payroll paid",
      "date": "2026-03-31"
    }
  ]
}
```

### GET /api/v1/employer-portal/employees/{id}/variable-pay
Get variable pay form for employee.

**Output:**
```json
{
  "employee": {
    "id": "uuid",
    "name": "John Smith",
    "employee_number": "EMP001"
  },
  "pay_period": {
    "id": "uuid",
    "period": "April 2026",
    "cut_off": "2026-04-28"
  },
  "fields": [
    {
      "name": "hours_worked",
      "label": "Hours Worked",
      "type": "number",
      "value": null,
      "required": true
    },
    {
      "name": "overtime_hours",
      "label": "Overtime Hours",
      "type": "number",
      "value": null,
      "required": false
    },
    {
      "name": "bonus",
      "label": "Bonus",
      "type": "currency",
      "value": null,
      "required": false
    }
  ]
}
```

### POST /api/v1/employer-portal/pay-periods/{id}/submit-variable-pay
Submit variable pay data.

**Input:**
```json
{
  "employee_data": [
    {
      "employee_id": "uuid",
      "hours_worked": 160,
      "overtime_hours": 8,
      "bonus": 200.00
    }
  ]
}
```

**Output:**
```json
{
  "submission_id": "uuid",
  "status": "submitted",
  "submitted_at": "2026-04-27T14:30:00Z",
  "bureau_notified": true
}
```

### POST /api/v1/employer-portal/pay-runs/{id}/approve
Approve payroll.

**Input:**
```json
{
  "decision": "approved",
  "comments": "All figures correct"
}
```

**Output:**
```json
{
  "approval_id": "uuid",
  "approved_at": "2026-04-27T15:00:00Z",
  "status": "approved"
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| Core Payroll | Payroll data | Internal API |
| Bureau Operations | Notifications | Events |
| Document Store | Reports | Internal API |
| Notification Service | Emails | Events |

---

## Non-Functional Requirements

### Security
- Employer-scoped access (cannot see other clients)
- MFA required for admins and managers
- Audit logging of all approvals and data submissions
- Bank detail changes require bureau approval

### UX
- Mobile-responsive for on-the-go approvals
- Clear deadline visibility with countdown
- Simple data entry with validation
- PDF report viewing inline

### Performance
- Dashboard load <2 seconds
- Data submission <3 seconds
- Report generation <10 seconds
