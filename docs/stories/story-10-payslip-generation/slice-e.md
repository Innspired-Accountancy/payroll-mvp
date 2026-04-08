# Slice e: Employee Portal Payslip View

**Story:** story-10-payslip-generation
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Create employee portal view for payslips with list, detail, and PDF download.

---

## Decision Checklist

- [x] Portal: Employee self-service view
- [x] List: All payslips with filters
- [x] Detail: Full payslip display
- [x] Download: PDF download button
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-05-employee-portal-spec.md — Employee portal
- 02-01-core-payroll-spec.md — Payslip access

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(employee)/payslips/page.tsx` | create | Payslip list |
| `src/app/(employee)/payslips/[id]/page.tsx` | create | Payslip detail |
| `src/components/payslip/payslip-list.tsx` | create | List component |
| `src/components/payslip/payslip-download.tsx` | create | Download button |

---

## Responsibilities

1. List employee's payslips
2. Display payslip detail
3. Provide PDF download
4. Filter by tax year

---

## Contracts

### employeePayslips.list
- **Method:** tRPC query `employeePayslips.list`
- **Input:** `{ tax_year?: string }`
- **Output:** PayslipSummary[]

### PayslipSummary
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Payslip ID |
| pay_date | Date | Pay date |
| period | string | Period description |
| gross_pay | Decimal | Gross amount |
| net_pay | Decimal | Net amount |
| has_pdf | boolean | PDF available |

---

## Business Rules & Invariants

1. Employee sees only own payslips
2. Only finalised pay run payslips visible
3. PDF download logged to audit
4. Mobile responsive

---

## Edge Cases

1. **No payslips** — Empty state
2. **PDF not ready** — Show generating status
3. **Historical data** — Show all available years

---

## Tests

### employee-payslips.test.ts
- List payslips
- View payslip detail
- Download PDF
- Filter by year

---

## Verification

```bash
npm run test:unit
npm run typecheck
npm run build
```

---

## Source Sections

- 02-05-employee-portal-spec.md § Payslip Access
