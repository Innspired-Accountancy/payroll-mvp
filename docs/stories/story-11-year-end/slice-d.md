# Slice d: YTD Carry Forward

**Story:** story-11-year-end
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement YTD carry forward to new tax year including resetting tax codes and updating thresholds.

---

## Decision Checklist

- [x] Reset: YTD totals to zero
- [x] Tax codes: Apply P9X updates
- [x] Thresholds: New tax year config
- [x] Validation: Check carry forward success
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Year-end carry forward
- HMRC P9X — Tax code update notices

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/year-end/carry-forward.ts` | create | Carry forward logic |
| `src/server/routers/carry-forward.ts` | create | tRPC endpoint |
| `src/tests/carry-forward.test.ts` | create | Carry forward tests |

---

## Responsibilities

1. Reset YTD totals to zero for new tax year
2. Apply tax code updates from P9X
3. Load new tax year thresholds
4. Validate carry forward completion

---

## Contracts

### carryForwardToNewYear
- **Method:** `carryForwardToNewYear(employerId: uuid, fromTaxYear: string, toTaxYear: string): CarryForwardResult`
- **Input:** Employer and tax years
- **Output:** Carry forward result

### CarryForwardResult
| Field | Type | Description |
|-------|------|-------------|
| employees_processed | int | Count reset |
| tax_codes_updated | int | Codes from P9X |
| errors | string[] | Any errors |
| success | boolean | Overall success |

### carryForward.execute
- **Method:** tRPC mutation `carryForward.execute`
- **Input:** `{ from_tax_year: string, to_tax_year: string }`
- **Output:** CarryForwardResult

---

## Business Rules & Invariants

1. YTD reset to zero for all continuing employees
2. Tax codes updated per P9X notices
3. New tax year config loaded
4. Must complete before first new year payroll

---

## Edge Cases

1. **P9X not received** — Keep current codes, flag for review
2. **Employee on emergency code** — Keep emergency
3. **Mid-year carry forward** — Block until year complete

---

## Tests

### carry-forward.test.ts
- Reset YTD totals
- Apply tax code updates
- Load new thresholds
- Error handling

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § YTD Carry Forward
