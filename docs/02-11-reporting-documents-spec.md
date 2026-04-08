# Module: Reporting & Documents

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Full Stack Team
**Priority:** P1 (Required for V1)

---

## Purpose

Generate standard payroll, compliance, and operational reports plus configurable exports, scheduled outputs, and secure document storage with retention management and audit trails.

---

## User Journeys

### Journey 1: Generate Payroll Report
1. **Bureau User** navigates to Reports
2. User selects report type (Payroll Register, Cost Analysis, etc.)
3. User selects date range and employers
4. User clicks Generate
5. System processes report
6. System displays report inline
7. User downloads as PDF or Excel

### Journey 2: Schedule Recurring Report
1. **Bureau Manager** configures report schedule
2. Manager selects report template
3. Manager sets frequency (weekly/monthly)
4. Manager selects recipients
5. System saves schedule
6. System automatically generates and emails report

### Journey 3: Access Employee Documents
1. **Employee** views Documents section
2. System lists available documents by category
3. Employee selects tax year
4. System displays P60 or payslips
5. Employee views or downloads PDF

### Journey 4: Generate Year-End P60s
1. **Payroll Manager** initiates year-end
2. System generates P60 for all employees employed on 5 April
3. System validates data completeness
4. System makes P60s available to employees
5. System tracks download/access for audit

---

## Data Models

### Entity: ReportTemplate
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| name | String | Yes | Report name |
| description | String | Yes | Description |
| category | Enum | Yes | payroll/compliance/operational |
| query_template | String | Yes | SQL or template reference |
| parameters | JSONB | Yes | Required parameters |
| output_formats | JSONB | Yes | pdf/excel/csv |

### Entity: GeneratedReport
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| template_id | UUID | Yes | FK to ReportTemplate |
| generated_by | UUID | Yes | FK to User |
| generated_at | DateTime | Yes | Timestamp |
| parameters | JSONB | Yes | Report parameters used |
| format | Enum | Yes | pdf/excel/csv |
| file_path | String | Yes | Storage location |
| file_size | Int | Yes | Size in bytes |
| retention_until | Date | Yes | Deletion date |

### Entity: Document
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| document_type | Enum | Yes | payslip/p60/p45/p11d/cis_statement |
| entity_type | String | Yes | Employee/Employer |
| entity_id | UUID | Yes | ID of related entity |
| tax_year | String | Yes | Tax year |
| period | String | No | Pay period if applicable |
| file_path | String | Yes | Storage path |
| file_hash | String | Yes | SHA256 for integrity |
| generated_at | DateTime | Yes | Generation timestamp |
| generated_from | UUID | No | FK to PayRun if applicable |

---

## Standard Reports

### Payroll Reports
| Report | Description | Audience |
|--------|-------------|----------|
| Payroll Register | All pay runs for period | Bureau staff |
| Employee Summary | Individual employee history | Bureau staff |
| Variance Report | Period-over-period changes | Reviewers |
| Department Costing | Costs by department/cost centre | Employers |

### Compliance Reports
| Report | Description | Audience |
|--------|-------------|----------|
| RTI Submission Log | FPS/EPS submission history | Compliance |
| Pension Compliance | AE assessment and enrolment | Compliance |
| CIS Returns | Monthly returns summary | Compliance |
| Year-End Summary | P60s and final submissions | Compliance |

### Operational Reports
| Report | Description | Audience |
|--------|-------------|----------|
| Client Activity | Client portfolio status | Bureau owners |
| Exception Summary | Errors and failures | Managers |
| User Activity | System usage audit | Admins |

---

## API Contracts

### GET /api/v1/reports/templates
List available report templates.

**Output:**
```json
{
  "templates": [
    {
      "id": "uuid",
      "name": "Payroll Register",
      "category": "payroll",
      "formats": ["pdf", "excel"],
      "parameters": ["start_date", "end_date", "employer_id"]
    }
  ]
}
```

### POST /api/v1/reports/generate
Generate a report.

**Input:**
```json
{
  "template_id": "uuid",
  "parameters": {
    "start_date": "2026-04-01",
    "end_date": "2026-04-30",
    "employer_id": "uuid"
  },
  "format": "pdf"
}
```

**Output:**
```json
{
  "report_id": "uuid",
  "status": "generating",
  "estimated_completion": "2026-04-28T10:05:00Z"
}
```

### GET /api/v1/documents
List documents for entity.

**Query:**
- `entity_type`: employee/employer
- `entity_id`: UUID
- `document_type`: payslip/p60/p45

**Output:**
```json
{
  "documents": [
    {
      "id": "uuid",
      "type": "payslip",
      "period": "April 2026",
      "date": "2026-04-30",
      "download_url": "/api/v1/documents/uuid/download"
    }
  ]
}
```

---

## Non-Functional Requirements

### Performance
- Standard reports generate in <10 seconds
- Large reports (1000+ employees) in <60 seconds
- PDF generation <5 seconds per document
- Async generation for complex reports

### Security
- Role-based report access
- Audit logging of all report access
- Secure document storage with encryption
- Time-limited download URLs

### Retention
- Reports retained per policy (configurable)
- Automatic archival after 2 years
- GDPR-compliant deletion
