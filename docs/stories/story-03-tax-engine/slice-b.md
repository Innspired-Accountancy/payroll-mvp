# Slice b: Core Tax Calculator

**Story:** story-03-tax-engine
**Epic:** epic-01-core-payroll
**Effort:** L
**Dependencies:** slice-a (tax code parser)

---

## Goal

Implement the core PAYE tax calculation using HMRC's exact percentage method with cumulative basis tracking.

---

## Decision Checklist

- [x] Algorithm: HMRC exact percentage method (cumulative)
- [x] Data types: Decimal(10,2) for money, Decimal(5,4) for rates
- [x] Thresholds: From TaxYearConfig (versioned per tax year)
- [x] Output: TaxCalculationResult with full trace
- [x] Error handling: InvalidCodeError, MissingYTDError
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md § Tax Calculation
- epic-01-core-payroll/epic-plan.md § Calculation Engine Design
- HMRC CWG2 Employer Further Guide

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `backend/app/services/tax_calculator.py` | create | Tax calculation service |
| `backend/app/models/tax_calculation.py` | create | Tax calculation result model |
| `backend/app/core/tax_year_config.py` | create | Tax year configuration |
| `backend/tests/test_tax_calculator.py` | create | Unit tests with HMRC test data |

---

## Responsibilities

1. Parse tax code and extract allowances/rates
2. Calculate taxable pay (gross - tax-free)
3. Apply tax bands and rates cumulatively
4. Return tax due with calculation trace
5. Update cumulative YTD tracking

---

## Contracts

### TaxCalculator.calculate()
- **Method:** `TaxCalculator.calculate(input: TaxCalculationInput) -> TaxCalculationResult`
- **Input:** TaxCalculationInput with employee data, earnings, YTD
- **Output:** TaxCalculationResult with tax_due, trace, updated_ytd
- **Errors:** InvalidTaxCodeError, MissingDataError

### TaxCalculationInput
| Field | Type | Description |
|-------|------|-------------|
| tax_code | str | e.g., "1257L" |
| gross_pay | Decimal | Period gross |
| taxable_pay_ytd | Decimal | YTD taxable |
| tax_paid_ytd | Decimal | YTD tax paid |
| tax_year | str | "2026-27" |
| period | int | Period number |

### TaxCalculationResult
| Field | Type | Description |
|-------|------|-------------|
| tax_due | Decimal | Tax for this period |
| taxable_pay | Decimal | Pay subject to tax |
| tax_free_amount | Decimal | Allowance portion |
| updated_ytd_taxable | Decimal | New YTD taxable |
| updated_ytd_tax | Decimal | New YTD tax |
| trace | List[CalculationStep] | Step-by-step |

---

## Business Rules & Invariants

1. Tax-free amount = (allowance / 12) for monthly, (/52) for weekly
2. Taxable pay = gross - tax-free (minimum 0)
3. Tax bands applied cumulatively: 20% on first £37,700, then 40%, etc.
4. Cumulative YTD must be tracked and updated
5. Emergency tax (W1/M1) ignores cumulative, calculates per-period

---

## Edge Cases

1. **Negative taxable** (K-code) — Add to taxable, tax max 50% of gross
2. **Week 53** — Special annual calculation
3. **Tax code change mid-year** — Recalculate using new code
4. **Multiple employments** — Each calculated separately
5. **Zero gross** — Return zero tax, carry YTD forward

---

## Tests

### test_tax_calculator.py
- Standard 1257L monthly calculation
- Week 1 basis calculation
- Scottish S-prefix codes
- Welsh C-prefix codes
- K-code negative tax
- BR code (no allowance)
- Week 53 handling
- HMRC reference data test pack (10 scenarios)

---

## Verification

```bash
cd backend
pytest tests/test_tax_calculator.py -v
pytest tests/test_tax_calculator.py::test_hmrc_reference_pack -v
```

---

## Source Sections

- epic-01-core-payroll/epic-plan.md § Calculation Engine Design
- 02-01-core-payroll-spec.md § PAY-050 to PAY-056
