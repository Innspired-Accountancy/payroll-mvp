# Slice a: NIC Category Configuration

**Story:** story-04-nic-calculator
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** None

---

## Goal

Create the NIC category configuration with thresholds and rates for all employee categories (A, B, C, H, J, M, Z, X) per tax year.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x
- [x] Data contracts defined: NicCategoryConfig, NicThresholds Zod schemas
- [x] Configuration: TypeScript config files per tax year
- [x] Error scenarios: Invalid category, missing config
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — NIC calculation requirements
- HMRC CWG2 — NIC tables and thresholds

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/config/nic/2026-27.ts` | create | 2026-27 NIC config |
| `src/lib/types/nic.ts` | create | NIC type definitions |
| `src/lib/db/schema/nic-configs.ts` | create | Config storage schema |
| `src/tests/nic-config.test.ts` | create | Config validation tests |

---

## Responsibilities

1. Define NIC categories and their letter codes
2. Configure LEL, PT, ST, UEL thresholds
3. Configure employee and employer rates per category
4. Support pro-rata thresholds for different frequencies
5. Version config per tax year

---

## Contracts

### NicCategoryConfig
| Field | Type | Description |
|-------|------|-------------|
| category | enum | A, B, C, H, J, M, Z, X |
| description | string | Human-readable description |
| employee_rates | RateBand[] | Employee contribution rates |
| employer_rates | RateBand[] | Employer contribution rates |

### NicThresholds (2026-27)
| Threshold | Annual | Weekly | Monthly |
|-----------|--------|--------|---------|
| LEL | £6,500 | £125 | £542 |
| PT | £9,100 | £175 | £758 |
| ST | £9,100 | £175 | £758 |
| UEL | £50,270 | £967 | £4,189 |

### RateBand
| Field | Type | Description |
|-------|------|-------------|
| threshold_from | string | Lower threshold code |
| threshold_to | string | Upper threshold code |
| rate | decimal | Percentage rate |

---

## Business Rules & Invariants

1. Category A: Standard rate, most employees
2. Category B: Married women/widows reduced rate (legacy)
3. Category C: Employees over State Pension age
4. Category H: Apprentices under 25
5. Category J: Employees deferring NIC
6. Category M: Employees under 21
7. Category Z: Employees under 21 deferring
8. Category X: No NIC liability

---

## Edge Cases

1. **Legacy categories** — Support B, J for historical calculations
2. **Category transitions** — Handle mid-year changes
3. **Director flag** — Separate calculation path

---

## Tests

### nic-config.test.ts
- All categories have valid rates
- Thresholds are positive and ordered
- Pro-rata calculations are correct
- Config loads for tax year

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § NIC Tables
- 02-01-core-payroll-spec.md § NIC Calculation
