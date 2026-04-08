# Module: Bureau Operations Centre

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Full Stack Team
**Priority:** P0 (Key Differentiator)

---

## Purpose

Provide the central orchestration layer for payroll bureaux and accountancy firms to manage multiple clients, track deadlines, handle exceptions, automate client chasing, and gain real-time visibility across their entire portfolio. This is the primary differentiator from incumbent solutions.

---

## User Journeys

### Journey 1: Multi-Client Dashboard Review
1. **Bureau Owner/Payroll Manager** logs in and sees dashboard
2. System displays all clients with current payroll status
3. System highlights urgent items (deadlines today, failed submissions, missing data)
4. System shows capacity view (payrolls due this week, team workload)
5. Manager drills into specific client to see detailed status
6. Manager reassigns overdue payroll to available processor

### Journey 2: Exception Queue Management
1. **Payroll Manager** views exception queue
2. System categorizes exceptions (HMRC failure, pension rejection, missing data, approval pending)
3. Manager filters by severity and assignee
4. Manager clicks exception to view details and context
5. Manager assigns to team member or resolves directly
6. System logs resolution with timestamp and notes

### Journey 3: Automated Client Chasing
1. **System** detects client payroll data incomplete 2 days before cut-off
2. System sends automated reminder email to client contact
3. System updates dashboard with "chase sent" status
4. System escalates to bureau team if no response after 24 hours
5. System logs all communications for audit

### Journey 4: Deadline Calendar Management
1. **Payroll Manager** views bureau calendar
2. System shows all client pay dates, cut-offs, filing deadlines
3. System color-codes by status (green=on track, amber=attention needed, red=overdue)
4. Manager filters by team member or client group
5. Manager clicks deadline to navigate to relevant payroll

---

## Data Models

### Entity: BureauDashboardView
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| bureau_id | UUID | Yes | FK to Bureau |
| view_name | String | Yes | Display name |
| view_type | Enum | Yes | all_clients/my_clients/team_view |
| filters | JSONB | No | Saved filter configuration |
| columns | JSONB | Yes | Column configuration |
| created_by | UUID | Yes | FK to User |

### Entity: ClientPayrollStatus
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| current_period_id | UUID | No | FK to current PayPeriod |
| status | Enum | Yes | not_started/data_collection/calculated/in_review/approved/submitted/paid/complete |
| cut_off_date | Date | No | Client data deadline |
| pay_date | Date | No | Employee pay date |
| filing_deadline | Date | No | HMRC filing deadline |
| assigned_processor | UUID | No | FK to User |
| assigned_reviewer | UUID | No | FK to User |
| last_activity | DateTime | Yes | Timestamp of last update |
| days_overdue | Int | No | Days past cut-off if applicable |

### Entity: ExceptionQueueItem
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| bureau_id | UUID | Yes | FK to Bureau |
| employer_id | UUID | Yes | FK to Employer |
| exception_type | Enum | Yes | hmrc_failure/pension_failure/payment_failure/missing_data/approval_pending/validation_error |
| severity | Enum | Yes | low/medium/high/critical |
| status | Enum | Yes | open/assigned/resolved/escalated |
| title | String | Yes | Short description |
| description | Text | Yes | Full details |
| source_module | String | Yes | Which module raised it |
| source_reference | UUID | No | ID of related record |
| assigned_to | UUID | No | FK to User |
| created_at | DateTime | Yes | Timestamp |
| resolved_at | DateTime | No | Resolution timestamp |
| resolution_notes | Text | No | How it was resolved |

### Entity: ClientCommunication
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| communication_type | Enum | Yes | reminder/chase/approval_request/notification |
| channel | Enum | Yes | email/portal/sms |
| subject | String | Yes | Message subject |
| content | Text | Yes | Message body |
| sent_by | Enum | Yes | system/user |
| sent_by_user_id | UUID | No | FK to User if manual |
| sent_at | DateTime | Yes | Timestamp |
| opened_at | DateTime | No | When recipient opened |
| response_received | Boolean | Yes | Whether response received |

---

## API Contracts

### GET /api/v1/bureau/dashboard
Get bureau dashboard summary.

**Output:**
```json
{
  "summary": {
    "total_clients": 150,
    "payrolls_this_week": 45,
    "overdue_items": 3,
    "critical_exceptions": 2
  },
  "status_breakdown": {
    "not_started": 12,
    "data_collection": 28,
    "calculated": 15,
    "in_review": 8,
    "approved": 5,
    "submitted": 42,
    "complete": 40
  },
  "upcoming_deadlines": [
    {
      "employer_id": "uuid",
      "employer_name": "ABC Ltd",
      "deadline_type": "cut_off",
      "deadline_date": "2026-04-29",
      "days_remaining": 1,
      "status": "attention_needed"
    }
  ],
  "exceptions": [
    {
      "id": "uuid",
      "type": "hmrc_failure",
      "severity": "critical",
      "employer_name": "XYZ Ltd",
      "description": "FPS rejected - invalid tax code",
      "assigned_to": "John Smith"
    }
  ]
}
```

### GET /api/v1/bureau/clients
List all clients with current status.

**Query Parameters:**
- `status`: Filter by status
- `assigned_to`: Filter by processor
- `overdue`: Boolean for overdue items only

**Output:**
```json
{
  "clients": [
    {
      "employer_id": "uuid",
      "name": "ABC Ltd",
      "paye_scheme": "123/AB45678",
      "current_period": "April 2026",
      "status": "in_review",
      "cut_off_date": "2026-04-28",
      "pay_date": "2026-04-30",
      "assigned_processor": "Jane Doe",
      "days_overdue": 0,
      "has_exceptions": false
    }
  ],
  "total": 150,
  "page": 1
}
```

### POST /api/v1/exceptions/{id}/assign
Assign exception to team member.

**Input:**
```json
{
  "assigned_to": "uuid",
  "notes": "Please investigate HMRC rejection"
}
```

### POST /api/v1/employers/{id}/send-reminder
Send manual reminder to client.

**Input:**
```json
{
  "reminder_type": "missing_data",
  "message": "Please submit timesheets by 5pm today",
  "channel": "email"
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| Core Payroll | Payroll status updates | Events |
| HMRC Submissions | Submission failures | Events |
| Pension Module | Pension submission failures | Events |
| Email Service | Client communications | Internal API |
| Notification Service | Internal alerts | Events |

---

## Non-Functional Requirements

### Real-Time Updates
- Dashboard updates within 5 seconds of status change
- Exception queue notifications immediate
- Push notifications for critical items

### Scalability
- Support 1000+ clients per bureau
- 100 concurrent users per bureau
- Filter and search <2 seconds

### UX
- Responsive design for desktop/tablet
- Keyboard shortcuts for common actions
- Customizable views per user role
- Bulk actions (reassign, chase, approve)
