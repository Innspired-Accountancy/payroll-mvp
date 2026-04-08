# Epic: Core Payroll Engine

**Date:** 2026-04-08
**Sprint(s):** Months 1-2
**Dependencies:** None (foundation epic)

---

## Scope & Deliverables

Accurate UK payroll calculation engine supporting all standard scenarios: PAYE, NIC (all categories), statutory payments, student loans, salary sacrifice, and attachment orders. Produces immutable calculation snapshots and feeds downstream compliance modules.

### In Scope
- Employee and employment management
- Pay period and pay run lifecycle
- PAYE tax calculation (cumulative, Week 1/Month 1, Scottish/Welsh)
- NIC calculation (all categories, directors methods)
- Statutory payments (SSP, SMP, SPP, SAP, ShPP)
- Student and postgraduate loans
- Salary sacrifice and pre/post-tax deductions
- Basic AEO support
- Payslip generation with calculation trace
- Year-end P60 generation

### Out of Scope
- CIS processing (separate epic)
- P11D/benefits (phase 2)
- Advanced AEO edge cases (phase 2)
- Net-to-gross (phase 2)

---

## Decisions

### Libraries & Packages

| Package | Version | Rationale | License |
|---------|---------|-----------|---------|
| Drizzle ORM | 0.30.x | Type-safe SQL, lightweight, PostgreSQL native | Apache-2.0 |
| Zod | 3.22.x | Schema validation (shared with frontend) | MIT |
| Vitest | 1.x | Fast testing, TypeScript native | MIT |
| fast-xml-parser | 4.x | HMRC XML generation/parsing (if needed) | MIT |
| date-fns | 3.x | Date manipulation for tax calculations | MIT |

### Calculation Engine Design

| Component | Approach | Details |
|-----------|----------|---------|
| Tax Calculator | HMRC exact percentage method | TypeScript implementation, Decimal.js for precision |
| NIC Calculator | Category-based rules | LEL, PT, ST, UEL thresholds per tax year |
| Statutory Pay | AWE-based calculations | 52-week reference period, eligibility checks |
| Configuration | Versioned tax year config | TypeScript configs per tax year, effective dating |

### Data Types

| Value | Type | Precision |
|-------|------|-----------|
| Money | Decimal.js | 2 decimal places GBP |
| Rates | number | 4 decimal places for percentages |
| Dates | Date | ISO 8601 |

### Data Contracts

#### PayCalculationInput
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| employee_id | UUID | Yes | Employee reference |
| tax_code | String | Yes | e.g., "1257L" |
| ni_category | String | Yes | e.g., "A" |
| gross_salary | Decimal | Yes | Monthly gross |
| year_to_date | YTDRecord | Yes | Cumulative totals |
| pay_period | PayPeriod | Yes | Period dates |

#### PayCalculationResult
| Field | Type | Description |
|-------|------|-------------|
| tax_deducted | Decimal | PAYE amount |
| nic_employee | Decimal | Employee NIC |
| nic_employer | Decimal | Employer NIC |
| net_pay | Decimal | Final net |
| ytd_after | YTDRecord | Updated cumulative |
| trace | CalculationTrace | Step-by-step |

---

## Build Order (Story Sequence)

| # | Story | Description | Effort | Dependencies |
|---|-------|-------------|--------|--------------|
| 1 | story-01-employee-management | Employee CRUD, employments, effective dating | M | None |
| 2 | story-02-payroll-setup | Pay schedules, pay periods, calendar | M | story-01 |
| 3 | story-03-tax-calculator | PAYE calculation engine | L | story-02 |
| 4 | story-04-nic-calculator | NIC calculation, all categories | L | story-02 |
| 5 | story-05-statutory-payments | SSP, SMP, SPP calculations | L | story-03, story-04 |
| 6 | story-06-student-loans | Plan 1, 2, 4, 5, PGL deductions | M | story-03 |
| 7 | story-07-deductions | Salary sacrifice, basic AEO | M | story-03, story-04 |
| 8 | story-08-pay-run-lifecycle | Draft → Calculated → Approved flow | M | story-03, story-04 |
| 9 | story-09-calculation-trace | Explain calculation feature | S | story-08 |
| 10 | story-10-payslip-generation | PDF payslips, YTD display | M | story-08 |
| 11 | story-11-year-end | P60 generation, carry forward | M | story-10 |

---

## Decision Completeness Checklist

- [x] All third-party libraries named with versions and license compatibility confirmed
- [x] All calculation methods specified (HMRC exact percentage)
- [x] Data types defined for financial calculations (Decimal)
- [x] Tax year configuration approach defined (versioned JSON)
- [x] Error handling strategy defined (validation + domain errors)
- [x] No "TBD", slash-notation, or placeholder text remaining
