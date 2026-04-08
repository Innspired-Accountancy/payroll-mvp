# Slice b: Payslip HTML Template

**Story:** story-10-payslip-generation
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Create payslip HTML template with all statutory required fields and professional formatting.

---

## Decision Checklist

- [x] Fields: All statutory payslip requirements
- [x] Layout: Professional, readable design
- [x] Responsive: Print-friendly CSS
- [x] Accessibility: Screen reader support
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- Employment Rights Act — Statutory payslip requirements
- 02-01-core-payroll-spec.md — Payslip display

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/payslip/payslip-template.tsx` | create | Payslip component |
| `src/styles/payslip.css` | create | Print styles |
| `src/tests/payslip-template.test.tsx` | create | Template tests |

---

## Responsibilities

1. Display all required payslip fields
2. Show earnings breakdown
3. Show deductions breakdown
4. Display YTD totals
5. Print-friendly formatting

---

## Contracts

### PayslipTemplate Props
| Prop | Type | Description |
|------|------|-------------|
| payslip | PayslipData | Payslip data |
| employer | EmployerInfo | Employer details |
| employee | EmployeeInfo | Employee details |
| show_ytd | boolean | Show YTD column |

### Statutory Required Fields
| Field | Required |
|-------|----------|
| Gross pay | Yes |
| Net pay | Yes |
| Deductions (type and amount) | Yes |
| Pay period | Yes |
| Pay date | Yes |
| Employer name | Yes |
| Employee name | Yes |

---

## Business Rules & Invariants

1. All statutory fields must be present
2. Deductions shown individually
3. YTD visible for current tax year
4. Print hides navigation

---

## Edge Cases

1. **Many deductions** — Scrollable or multi-page
2. **Zero amounts** — Show as £0.00
3. **Long names** — Truncate with tooltip

---

## Tests

### payslip-template.test.tsx
- All statutory fields present
- Earnings breakdown correct
- Deductions breakdown correct
- YTD display
- Print styles applied

---

## Verification

```bash
npm run test:unit
npm run typecheck
npm run build
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Payslip Display
