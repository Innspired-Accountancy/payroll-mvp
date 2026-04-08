# Slice b: P60 Generation and PDF

**Story:** story-11-year-end
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Generate P60 forms with HMRC-compliant format and PDF output for all qualifying employees.

---

## Decision Checklist

- [x] Format: HMRC P60 template compliance
- [x] Fields: All statutory P60 fields
- [x] PDF: Professional format with employer branding
- [x] Distribution: Employee portal and bulk export
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC P60 specification
- 02-01-core-payroll-spec.md — P60 generation
- 02-11-reporting-documents-spec.md — Document formats

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/year-end/p60-generator.ts` | create | P60 generator |
| `src/components/documents/p60-template.tsx` | create | P60 template |
| `src/server/routers/p60.ts` | create | P60 tRPC |
| `src/tests/p60-generation.test.ts` | create | P60 tests |

---

## Responsibilities

1. Generate P60 for each qualifying employee
2. Include all statutory fields
3. Create PDF in HMRC-compliant format
4. Store for employee access

---

## Contracts

### generateP60
- **Method:** `generateP60(employeeData: YearEndEmployeeData): P60Data`
- **Input:** Employee year-end data
- **Output:** P60 data structure

### P60Data
| Field | Type | Description |
|-------|------|-------------|
| employer_name | string | Employer name |
| employer_address | string | Employer address |
| employer_paye_ref | string | PAYE reference |
| employee_name | string | Employee name |
| employee_address | string | Employee address |
| ni_number | string | NI number |
| tax_code | string | Final tax code |
| total_pay | Decimal | Total pay for year |
| total_tax | Decimal | Total tax deducted |
| total_nic | Decimal | Total NIC |
| tax_year | string | Tax year |

### p60.generate
- **Method:** tRPC mutation `p60.generate`
- **Input:** `{ tax_year: string }`
- **Output:** `{ generated: int }`

---

## Business Rules & Invariants

1. P60 must be provided by May 31
2. Only for employment at April 5
3. Leavers with P45 don't get P60
4. Must match FPS year-end totals

---

## Edge Cases

1. **Multiple employments** — Separate P60 per employment
2. **Name change** — Use current name
3. **Address change** — Use current address

---

## Tests

### p60-generation.test.ts
- Generate P60 with all fields
- PDF creation
- Exclude leavers with P45
- Format compliance

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § P60 Generation
- 02-11-reporting-documents-spec.md § Document Templates
