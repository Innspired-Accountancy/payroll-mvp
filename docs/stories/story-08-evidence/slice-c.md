# Slice c: Compliance Reporting

**Story:** story-08-evidence
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Generate compliance reports for The Pensions Regulator including declaration of compliance data and evidence summaries.

---

## Decision Checklist

- [x] Report types: Declaration of compliance, evidence summary, assessment log
- [x] Format: PDF reports with official formatting
- [x] Data source: Evidence records and summary queries
- [x] TPR alignment: Report fields match TPR requirements
- [x] Signing: Digital signature support (optional)
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Journey 5 (Re-declaration)
- The Pensions Regulator: "Declaration of compliance"

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/evidence/complianceReports.ts` | create | Report generation |
| `src/lib/evidence/reports/declarationOfCompliance.ts` | create | DOC report generator |
| `src/lib/evidence/reports/evidenceSummary.ts` | create | Evidence summary generator |
| `src/server/routers/complianceReports.ts` | create | tRPC report API |
| `src/tests/evidence/complianceReports.test.ts` | create | Report tests |

---

## Responsibilities

1. Generate Declaration of Compliance report
2. Generate evidence summary by employer
3. Generate assessment log for period
4. Format reports per TPR requirements
5. Provide PDF output
6. Track report generation for audit

---

## Contracts

### generateDeclarationOfCompliance()
- **Method:** `generateDeclarationOfCompliance(employerId: UUID, taxYear: string): Promise<ReportResult>`
- **Content:**
  - Employer details
  - Staging date
  - Eligible jobholders count
  - Enrolled count
  - Opt-outs count
  - Pension scheme details
  - Compliance confirmation
- **Output:** PDF buffer and metadata

### generateEvidenceSummary()
- **Method:** `generateEvidenceSummary(employerId: UUID, dateFrom: Date, dateTo: Date): Promise<ReportResult>`
- **Content:**
  - Assessment count
  - Enrolment count
  - Opt-out count
  - Communication dispatch count
  - Evidence integrity summary
- **Output:** PDF buffer and metadata

### ReportResult
| Field | Type | Description |
|-------|------|-------------|
| report_id | UUID | Generated report ID |
| pdf_buffer | Buffer | PDF content |
| generated_at | DateTime | Generation timestamp |
| generated_by | UUID | User who generated |
| record_count | number | Evidence records included |

### complianceReports.generate
- **Method:** tRPC mutation `complianceReports.generate`
- **Input:** { employerId, reportType, dateRange? }
- **Output:** ReportResult with download URL
- **Auth:** Requires compliance:report:generate permission

---

## Business Rules & Invariants

1. Declaration of compliance generated annually
2. Evidence summaries generated on request for TPR inspection
3. Reports include tamper-evident footer (hash of content)
4. Report generation logged as evidence event
5. Reports retained for 6 years

---

## Edge Cases

1. **No data for period** — Report shows "No activity"
2. **Employer not yet staged** — Report shows future staging date
3. **Large employer (1000+ employees)** — Paginated report sections

---

## Tests

### complianceReports.test.ts
- Declaration of compliance generation
- Evidence summary generation
- PDF output validation
- Report metadata accuracy

---

## Verification

```bash
npm run test:unit src/tests/evidence/complianceReports.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-08-evidence/slice-a.md → Evidence data source
- story-08-evidence/slice-b.md → Query interface
