# Slice e: Pro-Rata Threshold Calculations

**Story:** story-04-nic-calculator
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-b

---

## Goal

Implement pro-rata threshold calculations for weekly, monthly, and irregular pay periods ensuring accurate NIC calculation across all frequencies.

---

## Decision Checklist

- [x] Algorithm: Annual thresholds ÷ period divisor
- [x] Data: Annual thresholds from NicThresholds config
- [x] Precision: 4 decimal places, round at final step
- [x] Divisors: Weekly=52, Monthly=12, Annual=1
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- HMRC CWG2 — NIC thresholds for different intervals
- 02-01-core-payroll-spec.md — Pay frequency handling

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/nic-thresholds.ts` | create | Pro-rata threshold calculator |
| `src/tests/calculations/nic-thresholds.test.ts` | create | Threshold tests |

---

## Responsibilities

1. Convert annual thresholds to period thresholds
2. Handle weekly, monthly, annual frequencies
3. Support director annual calculation (no pro-rata)
4. Ensure precision in threshold calculations
5. Cache threshold calculations per tax year

---

## Contracts

### getProRataThresholds
- **Method:** `getProRataThresholds(annual: NicThresholds, frequency: Frequency): NicThresholds`
- **Input:** Annual thresholds and frequency
- **Output:** Pro-rata thresholds for period

### Frequency Divisors
| Frequency | Divisor | Example (UEL £50,270) |
|-----------|---------|----------------------|
| weekly | 52 | £967.12 |
| monthly | 12 | £4,189.17 |
| annual | 1 | £50,270.00 |

---

## Business Rules & Invariants

1. Weekly thresholds = annual ÷ 52 (rounded to 4 decimal places)
2. Monthly thresholds = annual ÷ 12 (rounded to 4 decimal places)
3. Annual thresholds used for director calculations
4. Irregular payments: use relevant period rules per HMRC

---

## Edge Cases

1. **Week 53** — Use weekly threshold (not pro-rata adjustment)
2. **Month with 5 weeks** — Use monthly threshold
3. **Irregular period** — Calculate based on period length

---

## Tests

### nic-thresholds.test.ts
- Weekly threshold calculation
- Monthly threshold calculation
- Annual threshold (no change)
- Precision handling (4 decimal places)
- All threshold types (LEL, PT, ST, UEL)

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- HMRC CWG2 § NICs for various pay intervals
- 02-01-core-payroll-spec.md § Pay frequency handling
