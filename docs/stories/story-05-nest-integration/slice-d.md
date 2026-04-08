# Slice d: File-Based Fallback

**Story:** story-05-nest-integration
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Implement file-based submission fallback for NEST when API is unavailable or for employers preferring manual upload.

---

## Decision Checklist

- [x] File format: NEST-compatible CSV (contributions) / CSV (enrolments)
- [x] Generation: CSV/CSV files matching NEST templates
- [x] Validation: Pre-generation data validation
- [x] Delivery: Download for manual upload or SFTP
- [x] Audit: File generation logged with checksum
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- NEST File Upload Templates
- 02-03-pension-auto-enrolment-spec.md — File-based fallback

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/nest/fileGenerator.ts` | create | CSV/CSV generation |
| `src/lib/nest/templates/enrolmentTemplate.ts` | create | Enrolment file template |
| `src/lib/nest/templates/contributionTemplate.ts` | create | Contribution file template |
| `src/server/routers/nestFileDownloads.ts` | create | File download endpoints |
| `src/tests/nest/fileGenerator.test.ts` | create | File generation tests |

---

## Responsibilities

1. Generate NEST-compatible CSV files for enrolments
2. Generate NEST-compatible CSV files for contributions
3. Validate data before file generation
4. Provide download endpoints
5. Log file generation with checksums
6. Support optional SFTP upload

---

## Contracts

### generateEnrolmentFile()
- **Method:** `generateEnrolmentFile(employerId: UUID, enrolments: Enrolment[]): Promise<FileResult>`
- **Output:** CSV file buffer with NEST enrolment format
- **Format:** Comma-separated, UTF-8, with headers
- **Columns:** employer_ref, ni_number, title, first_name, last_name, dob, address...

### generateContributionFile()
- **Method:** `generateContributionFile(payPeriodId: UUID): Promise<FileResult>`
- **Output:** CSV file buffer with NEST contribution format
- **Format:** Comma-separated, UTF-8, with headers
- **Columns:** employer_ref, ni_number, pensionable_pay, employee_contribution...

### FileResult
| Field | Type | Description |
|-------|------|-------------|
| filename | string | Generated filename |
| content | Buffer | File content |
| checksum | string | SHA-256 checksum |
| recordCount | number | Number of records |
| generatedAt | DateTime | Generation timestamp |

---

## Business Rules & Invariants

1. File format matches NEST upload template exactly
2. All required fields populated or empty string (not null)
3. Dates formatted as DD/MM/YYYY per NEST spec
4. Currency formatted to 2 decimal places
5. UTF-8 encoding with BOM for Excel compatibility
6. Filename includes employer reference and date

---

## Edge Cases

1. **No records to export** — Empty file with headers only
2. **Special characters in names** — Properly escaped/quoted
3. **Very large file** — Chunked generation for memory efficiency
4. **Invalid characters** — Stripped or replaced per NEST rules

---

## Tests

### fileGenerator.test.ts
- Enrolment file generation
- Contribution file generation
- Date formatting (DD/MM/YYYY)
- Currency formatting (2dp)
- Special character handling
- Checksum calculation

---

## Verification

```bash
npm run test:unit src/tests/nest/fileGenerator.test.ts
npm run typecheck
npm run build
```

---

## Source Sections

- story-05-nest-integration/slice-a.md → Data to format
