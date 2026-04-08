# Slice a: Trace Data Structure

**Story:** story-09-calculation-trace
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** None

---

## Goal

Define the calculation trace data structure with hierarchical sections for all calculation components.

---

## Decision Checklist

- [x] Structure: Hierarchical with sections and steps
- [x] Sections: Earnings, tax, NIC, statutory, deductions
- [x] Step format: Formula, inputs, outputs, description
- [x] Storage: JSONB in payslip table
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Calculation trace
- 02-01-core-payroll-spec.md — Payslip entity

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/types/calculation-trace.ts` | create | Trace type definitions |
| `src/lib/validation/trace.ts` | create | Trace validation |
| `src/tests/trace-structure.test.ts` | create | Structure tests |

---

## Responsibilities

1. Define trace data structure
2. Create validation schema
3. Support hierarchical sections
4. Ensure serializability to JSON

---

## Contracts

### CalculationTrace
| Field | Type | Description |
|-------|------|-------------|
| version | string | Trace format version |
| generated_at | DateTime | Timestamp |
| sections | TraceSection[] | Hierarchical sections |
| final_result | TraceResult | Summary |

### TraceSection
| Field | Type | Description |
|-------|------|-------------|
| name | string | Section name (e.g., "Tax") |
| description | string | Human-readable |
| steps | TraceStep[] | Calculation steps |
| subsections | TraceSection[] | Nested sections |

### TraceStep
| Field | Type | Description |
|-------|------|-------------|
| step_id | string | Unique identifier |
| description | string | What this step does |
| formula | string | Formula used (optional) |
| inputs | Record<string, Decimal> | Input values |
| output | Decimal | Result value |
| notes | string | Additional info |

---

## Business Rules & Invariants

1. Every calculation step must be traceable
2. Trace must be deterministic (same inputs = same trace)
3. Trace stored as JSONB for querying
4. Version field for format evolution

---

## Edge Cases

1. **Large trace** — Size limits, compression
2. **Circular references** — Avoid in structure design
3. **Precision** — Decimal serialized as string

---

## Tests

### trace-structure.test.ts
- Valid trace creation
- Section nesting
- Decimal serialization
- Validation passes/fails

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Calculation Trace
