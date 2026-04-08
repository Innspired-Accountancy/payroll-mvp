# Slice c: PDF Generation Service

**Story:** story-10-payslip-generation
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-b

---

## Goal

Implement PDF generation from HTML payslip using headless browser or PDF library.

---

## Decision Checklist

- [x] Library: Playwright or Puppeteer for PDF
- [x] Rendering: Server-side HTML to PDF
- [x] Styling: Consistent with HTML template
- [x] Performance: Caching and optimization
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — PDF payslips
- 02-11-reporting-documents-spec.md — Document generation

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pdf/payslip-pdf.ts` | create | PDF generator |
| `src/server/routers/payslip-pdf.ts` | create | PDF tRPC |
| `src/tests/pdf-generation.test.ts` | create | PDF tests |

---

## Responsibilities

1. Convert HTML payslip to PDF
2. Apply consistent styling
3. Handle page breaks
4. Return PDF buffer/stream

---

## Contracts

### generatePayslipPdf
- **Method:** `generatePayslipPdf(payslipId: uuid): Promise<Buffer>`
- **Input:** Payslip ID
- **Output:** PDF buffer

### payslipPdf.generate
- **Method:** tRPC mutation `payslipPdf.generate`
- **Input:** `{ payslip_id: uuid }`
- **Output:** `{ download_url: string }`

---

## Business Rules & Invariants

1. PDF matches HTML display exactly
2. Single page per payslip (standard layout)
3. PDF/A-1b compliance for archiving
4. Fonts embedded

---

## Edge Cases

1. **Large payslip** — Multi-page if needed
2. **Font loading** — Embed fonts for consistency
3. **Generation failure** — Retry with fallback

---

## Tests

### pdf-generation.test.ts
- Generate PDF from payslip
- Verify content in PDF
- Page count check
- File size reasonable

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § PDF Payslips
