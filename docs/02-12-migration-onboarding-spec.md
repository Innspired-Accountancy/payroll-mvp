# Module: Migration & Onboarding

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend/Implementation Team
**Priority:** P1 (Required for V1)

---

## Purpose

Enable structured migration from incumbent systems (BrightPay, FreshPay, Xero) with employee/YTD import, validation, reconciliation reporting, parallel-run support, and readiness scoring to de-risk cutover.

---

## User Journeys

### Journey 1: Import Employees
1. **Implementation User** navigates to Migration
2. User selects source system (BrightPay/FreshPay/Xero)
3. User uploads export file
4. System parses and validates data
5. System shows import preview with errors/warnings
6. User maps source fields to platform fields
7. User confirms import
8. System creates employee records

### Journey 2: Validate YTD Data
1. **Implementation User** imports YTD values
2. System validates against employee records
3. System calculates validation checksums
4. System flags discrepancies
5. User reviews and corrects
6. System confirms YTD totals match

### Journey 3: Parallel Run
1. **Implementation User** runs parallel payroll
2. System processes same inputs as incumbent
3. System generates comparison report
4. User reviews variances
5. System identifies root causes
6. User iterates until variance <0.1%

### Journey 4: Cutover
1. **Implementation User** views readiness score
2. System checks: employees imported, YTD validated, parallel run passed
3. User confirms go-live date
4. System locks migration data
5. System enables live processing
6. System archives parallel run data

---

## Data Models

### Entity: MigrationProject
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| employer_id | UUID | Yes | FK to Employer |
| source_system | Enum | Yes | brightpay/freshpay/xero/other |
| status | Enum | Yes | in_progress/validated/live/aborted |
| created_by | UUID | Yes | FK to User |
| created_at | DateTime | Yes | Timestamp |
| target_go_live | Date | No | Planned go-live |
| actual_go_live | Date | No | Actual go-live |
| readiness_score | Int | No | 0-100 validation score |

### Entity: ImportedEmployee
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| migration_id | UUID | Yes | FK to MigrationProject |
| source_id | String | Yes | ID in source system |
| import_status | Enum | Yes | pending/valid/imported/error |
| raw_data | JSONB | Yes | Original import data |
| mapped_data | JSONB | No | Mapped to platform schema |
| validation_errors | JSONB | No | List of errors |
| platform_employee_id | UUID | No | FK to Employee once created |

### Entity: YTDImport
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| migration_id | UUID | Yes | FK to MigrationProject |
| employee_id | UUID | Yes | FK to Employee |
| tax_year | String | Yes | Tax year |
| taxable_pay_ytd | Money | Yes | YTD taxable pay |
| tax_paid_ytd | Money | Yes | YTD tax paid |
| nic_employee_ytd | Money | Yes | YTD employee NIC |
| nic_employer_ytd | Money | Yes | YTD employer NIC |
| validation_status | Enum | Yes | pending/validated/failed |
| validation_notes | String | No | Notes |

### Entity: ParallelRun
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| migration_id | UUID | Yes | FK to MigrationProject |
| pay_period | String | Yes | Period tested |
| incumbent_result | JSONB | Yes | Incumbent system totals |
| platform_result | JSONB | Yes | Platform totals |
| variance | Money | Yes | Difference amount |
| variance_percent | Decimal | Yes | Difference % |
| status | Enum | Yes | pass/fail/investigating |
| notes | Text | No | Investigation notes |

---

## Import Sources

### BrightPay
| Data | Format | Method |
|------|--------|--------|
| Employees | CSV export | File upload |
| YTD Values | CSV export | File upload |
| FPS History | XML files | File upload |

### FreshPay
| Data | Format | Method |
|------|--------|--------|
| Employees | CSV/API | Upload or API |
| YTD Values | CSV export | File upload |

### Xero Payroll
| Data | Format | Method |
|------|--------|--------|
| Employees | API | OAuth + API |
| YTD Values | API | OAuth + API |

---

## API Contracts

### POST /api/v1/migrations
Create new migration project.

**Input:**
```json
{
  "employer_id": "uuid",
  "source_system": "brightpay",
  "target_go_live": "2026-06-01"
}
```

**Output:**
```json
{
  "migration_id": "uuid",
  "status": "in_progress",
  "upload_url": "/api/v1/migrations/uuid/upload"
}
```

### POST /api/v1/migrations/{id}/upload-employees
Upload employee import file.

**Input:**
```multipart/form-data```
- file: CSV/Excel file

**Output:**
```json
{
  "upload_id": "uuid",
  "records_found": 45,
  "valid_records": 43,
  "errors": [
    {
      "row": 12,
      "error": "Missing NI number"
    }
  ]
}
```

### GET /api/v1/migrations/{id}/readiness
Get readiness score.

**Output:**
```json
{
  "migration_id": "uuid",
  "readiness_score": 95,
  "status": "ready_for_go_live",
  "checks": {
    "employees_imported": {"status": "pass", "count": 45},
    "ytd_validated": {"status": "pass", "variance": 0.00},
    "parallel_run": {"status": "pass", "variance_percent": 0.02},
    "pension_setup": {"status": "pass"},
    "bank_accounts": {"status": "warning", "missing": 2}
  }
}
```

### POST /api/v1/migrations/{id}/go-live
Complete migration and go live.

**Input:**
```json
{
  "confirmed": true,
  "notes": "All checks passed, ready for production"
}
```

**Output:**
```json
{
  "migration_id": "uuid",
  "status": "live",
  "go_live_date": "2026-05-01",
  "employer_status": "active"
}
```

---

## Non-Functional Requirements

### Accuracy
- 100% employee data accuracy post-import
- YTD variance tolerance: £0.01
- Parallel run variance tolerance: 0.1%

### Security
- Import files encrypted at rest
- Access limited to implementation users
- Audit trail of all imports
- PII masking in logs

### Performance
- Employee import: 100 records/second
- Validation report: <30 seconds
- Parallel run comparison: <5 seconds
