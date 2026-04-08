# Slice c: Final FPS with Year-End Indicators

**Story:** story-11-year-end
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Prepare final Full Payment Submission (FPS) with year-end indicators and summary totals for HMRC.

---

## Decision Checklist

- [x] FPS: Final submission with year-end flag
- [x] Indicators: Final submission, no more payments
- [x] Totals: Match P60 and year-end totals
- [x] Submission: Via HMRC submissions module
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md — FPS submission
- 02-01-core-payroll-spec.md — Year-end FPS

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/year-end/final-fps.ts` | create | Final FPS generator |
| `src/server/routers/final-fps.ts` | create | tRPC endpoint |
| `src/tests/final-fps.test.ts` | create | FPS tests |

---

## Responsibilities

1. Generate final FPS with year-end flag
2. Include all employee year-end data
3. Set final submission indicator
4. Submit via HMRC module

---

## Contracts

### generateFinalFps
- **Method:** `generateFinalFps(taxYear: string, employerId: uuid): FpsData`
- **Input:** Tax year and employer
- **Output:** FPS data ready for submission

### FinalFpsData
| Field | Type | Description |
|-------|------|-------------|
| tax_year | string | Tax year |
| is_final_submission | boolean | Year-end flag |
| no_more_payments | boolean | No more payments flag |
| employees | FpsEmployee[] | Per-employee data |
| totals | FpsTotals | Summary totals |

### finalFps.prepare
- **Method:** tRPC mutation `finalFps.prepare`
- **Input:** `{ tax_year: string }`
- **Output:** `{ fps_id: uuid, status: string }`

---

## Business Rules & Invariants

1. Final FPS submitted after last pay run
2. Must match P60 totals exactly
3. No more payments flag if no further payments
4. Submission deadline: April 19 (paper), April 22 (electronic)

---

## Edge Cases

1. **Late payments after final** — Submit additional FPS
2. **Corrections needed** — Submit corrected FPS
3. **No employees** — Empty final FPS

---

## Tests

### final-fps.test.ts
- Generate final FPS
- Year-end indicators set
- Totals match P60s
- Submission preparation

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § FPS Submission
- 02-01-core-payroll-spec.md § Year End FPS
