# Slice c: Trace Storage and Retrieval API

**Story:** story-09-calculation-trace
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Implement trace storage in payslip table and API endpoints for retrieval.

---

## Decision Checklist

- [x] Storage: payslip.calculation_trace JSONB column
- [x] API: tRPC query for trace retrieval
- [x] Size: Compression for large traces
- [x] Query: JSONB indexing for common queries
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Payslip entity

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/payslips.ts` | update | Add trace column |
| `src/server/routers/calculation-trace.ts` | create | Trace tRPC |
| `src/lib/storage/trace-compression.ts` | create | Compression utility |
| `src/tests/trace-storage.test.ts` | create | Storage tests |

---

## Responsibilities

1. Store trace in payslip record
2. Provide trace retrieval API
3. Compress large traces
4. Support trace querying

---

## Contracts

### payslipTrace.get
- **Method:** tRPC query `payslipTrace.get`
- **Input:** `{ payslip_id: uuid }`
- **Output:** CalculationTrace
- **Errors:** NOT_FOUND, FORBIDDEN

### payslipTrace.getSection
- **Method:** tRPC query `payslipTrace.getSection`
- **Input:** `{ payslip_id: uuid, section_path: string }`
- **Output:** Specific trace section

### TraceStorage
| Method | Description |
|--------|-------------|
| `store(payslipId, trace): void` | Store trace |
| `retrieve(payslipId): Trace` | Retrieve trace |
| `compress(trace): Buffer` | Compress large trace |
| `decompress(buffer): Trace` | Decompress |

---

## Business Rules & Invariants

1. Trace stored with every payslip
2. Trace immutable after creation
3. Compression applied if >100KB
4. API returns decompressed trace

---

## Edge Cases

1. **Very large trace** — Compression + size limit
2. **Missing trace** — Legacy payslip before feature
3. **Corrupted trace** — Validation on retrieval

---

## Tests

### trace-storage.test.ts
- Store and retrieve trace
- Compression roundtrip
- Large trace handling
- Missing trace error

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Calculation Trace
