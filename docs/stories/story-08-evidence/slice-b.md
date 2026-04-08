# Slice b: Audit Query Interface

**Story:** story-08-evidence
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Create an audit query interface for retrieving evidence records by employee, employer, date range, or record type.

---

## Decision Checklist

- [x] Query filters: employee_id, employer_id, date_range, record_type
- [x] Pagination: Cursor-based for large result sets
- [x] Sorting: By recorded_at (default), reference_id
- [x] Export: JSON/CSV export for evidence packages
- [x] Access control: Role-based (compliance officer, admin)
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Compliance audit requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/evidence/auditQuery.ts` | create | Evidence query builder |
| `src/server/routers/evidence.ts` | create | tRPC evidence API |
| `src/lib/evidence/export.ts` | create | Evidence export utilities |
| `src/tests/evidence/auditQuery.test.ts` | create | Query tests |

---

## Responsibilities

1. Query evidence records with filters
2. Support pagination for large datasets
3. Export evidence packages
4. Enforce access controls
5. Provide evidence verification

---

## Contracts

### queryEvidence()
- **Method:** `queryEvidence(filters: EvidenceFilters, pagination: Pagination): Promise<EvidencePage>`
- **Filters:**
  - employee_id: UUID (optional)
  - employer_id: UUID (optional)
  - record_type: enum[] (optional)
  - date_from: Date (optional)
  - date_to: Date (optional)
- **Pagination:** { cursor, limit }
- **Returns:** { records, next_cursor, total_count }

### exportEvidence()
- **Method:** `exportEvidence(filters: EvidenceFilters, format: enum): Promise<ExportResult>`
- **Formats:** "json", "csv"
- **Returns:** { download_url, expires_at, record_count }

### evidence.getByEmployee
- **Method:** tRPC query `evidence.getByEmployee`
- **Input:** { employeeId, dateFrom?, dateTo? }
- **Output:** Evidence records for employee
- **Auth:** Requires evidence:read permission

### evidence.getByEmployer
- **Method:** tRPC query `evidence.getByEmployer`
- **Input:** { employerId, dateFrom?, dateTo?, recordType? }
- **Output:** Evidence records for employer
- **Auth:** Requires evidence:read permission

---

## Business Rules & Invariants

1. Users can only access evidence for their tenant (employer_id)
2. Compliance officers can access cross-employer evidence
3. Exports limited to 10,000 records per request
4. Export URLs expire after 24 hours
5. All queries logged for audit

---

## Edge Cases

1. **No matching records** — Return empty result, not error
2. **Very large export** — Chunk into multiple files
3. **Unauthorized access** — Return 403
4. **Invalid date range** — Return 400 with error

---

## Tests

### auditQuery.test.ts
- Query by employee
- Query by employer
- Query by date range
- Query by record type
- Pagination handling
- Export generation

---

## Verification

```bash
npm run test:unit src/tests/evidence/auditQuery.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-08-evidence/slice-a.md → Evidence records to query
