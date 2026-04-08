# Module: Audit & Compliance

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend/Security Team
**Priority:** P0 (Critical Path)

---

## Purpose

Provide immutable audit logging of all data changes, system actions, and user activities with comprehensive evidence retention, compliance reporting, and support for regulatory investigations across all modules.

---

## User Journeys

### Journey 1: Review Audit Trail
1. **Compliance User** navigates to Audit section
2. User filters by date range, entity type, user
3. System displays audit log entries
4. User clicks entry to view details
5. System shows before/after values, actor, timestamp, reason
6. User exports filtered log for investigation

### Journey 2: Investigate Payroll Change
1. **Payroll Manager** notices discrepancy
2. Manager navigates to employee audit history
3. System shows all changes to employee record
4. Manager traces back to original entry
5. System identifies who made change and when
6. Manager views calculation snapshots

### Journey 3: Compliance Report
1. **Compliance Officer** generates compliance report
2. System aggregates audit data for period
3. System identifies potential violations
4. System generates report with evidence links
5. Officer reviews and signs off

---

## Data Models

### Entity: AuditLog
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| timestamp | DateTime | Yes | Event timestamp |
| actor_type | Enum | Yes | user/system/api |
| actor_id | UUID | No | ID of actor |
| actor_name | String | Yes | Human-readable name |
| action | Enum | Yes | create/update/delete/view/approve/etc |
| resource_type | String | Yes | Entity type (employee, payroll, etc) |
| resource_id | UUID | Yes | Entity ID |
| resource_description | String | Yes | Human-readable description |
| changes | JSONB | No | Before/after values |
| reason | String | No | User-provided reason |
| ip_address | String | No | Source IP |
| user_agent | String | No | Browser/app identifier |
| session_id | String | No | Session correlation |

### Entity: CalculationSnapshot
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| pay_run_id | UUID | Yes | FK to PayRun |
| snapshot_hash | String | Yes | SHA256 of snapshot |
| tax_year_config | JSONB | Yes | Complete tax year configuration |
| employee_snapshots | JSONB | Yes | Array of employee calc data |
| created_at | DateTime | Yes | Timestamp |
| retained_until | Date | Yes | Retention deadline |

### Entity: ComplianceReport
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| report_type | Enum | Yes | access_review/payroll_accuracy/submission_timeliness |
| period_start | Date | Yes | Report period start |
| period_end | Date | Yes | Report period end |
| generated_by | UUID | Yes | FK to User |
| generated_at | DateTime | Yes | Timestamp |
| findings | JSONB | Yes | Report findings |
| status | Enum | Yes | draft/reviewed/approved |

---

## Audit Events

### Payroll Events
- pay_period.created
- pay_run.calculated
- pay_run.approved
- pay_run.reopened
- payslip.viewed
- payslip.downloaded

### Employee Events
- employee.created
- employee.updated
- employee.terminated
- bank_details.changed
- tax_code.changed

### Compliance Events
- hmrc_submission.submitted
- pension_submission.submitted
- payment_batch.approved
- report.generated

### Access Events
- user.login
- user.logout
- user.password_changed
- permission.granted
- permission.revoked

---

## API Contracts

### GET /api/v1/audit-log
Query audit log with filters.

**Query Parameters:**
- `start_date`: From date
- `end_date`: To date
- `actor_id`: Filter by user
- `resource_type`: Filter by entity type
- `action`: Filter by action

**Output:**
```json
{
  "entries": [
    {
      "id": "uuid",
      "timestamp": "2026-04-28T10:30:00Z",
      "actor": "John Smith",
      "action": "update",
      "resource": "Employee",
      "resource_description": "Jane Doe (EMP001)",
      "changes": {
        "tax_code": {"from": "1257L", "to": "BR"}
      }
    }
  ],
  "total": 150,
  "page": 1
}
```

### GET /api/v1/audit-log/employees/{id}/history
Get complete history for employee.

**Output:**
```json
{
  "employee_id": "uuid",
  "employee_name": "Jane Doe",
  "history": [
    {
      "timestamp": "2026-01-15T09:00:00Z",
      "action": "created",
      "actor": "System"
    },
    {
      "timestamp": "2026-04-01T10:30:00Z",
      "action": "tax_code_changed",
      "actor": "John Smith",
      "from": "1257L",
      "to": "BR"
    }
  ]
}
```

### POST /api/v1/compliance-reports
Generate compliance report.

**Input:**
```json
{
  "report_type": "payroll_accuracy",
  "period_start": "2026-04-01",
  "period_end": "2026-04-30"
}
```

**Output:**
```json
{
  "report_id": "uuid",
  "status": "generated",
  "download_url": "/api/v1/compliance-reports/uuid/download"
}
```

---

## Retention Policies

| Record Type | Retention | Legal Basis |
|-------------|-----------|-------------|
| Payroll records | 6 years | PAYE Regulations |
| Pension records | 6 years | TPR guidance |
| Opt-out notices | 4 years | TPR guidance |
| Audit logs | 7 years | Best practice |
| Submission evidence | 6 years | HMRC compliance |
| CIS records | 3-6 years | CIS Regulations |

---

## Non-Functional Requirements

### Immutability
- Audit logs append-only
- No modification of historical records
- Tamper-evident logging
- Regular integrity verification

### Performance
- Audit log write <50ms
- Query response <2 seconds
- Archive old records after 2 years

### Compliance
- GDPR data subject access support
- DSAR export capability
- Automated retention enforcement
- Legal hold capability
