# Slice b: Trace Generation in Calculators

**Story:** story-09-calculation-trace
**Epic:** epic-01-core-payroll
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Integrate trace generation into all calculators (tax, NIC, statutory, deductions) capturing every calculation step.

---

## Decision Checklist

- [x] Integration: Trace captured in each calculator
- [x] Pattern: Builder pattern for trace construction
- [x] Performance: Optional tracing for production
- [x] Detail: All formulas, inputs, intermediate values
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Calculation trace

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/calculations/trace-builder.ts` | create | Trace builder utility |
| `src/lib/calculations/tax.ts` | update | Add tracing |
| `src/lib/calculations/nic.ts` | update | Add tracing |
| `src/lib/calculations/ssp.ts` | update | Add tracing |

---

## Responsibilities

1. Add trace builder to calculators
2. Capture each calculation step
3. Record formulas and inputs
4. Return trace with results

---

## Contracts

### TraceBuilder
| Method | Description |
|--------|-------------|
| `addStep(step: TraceStep): void` | Add calculation step |
| `addSection(section: TraceSection): void` | Add subsection |
| `setResult(result: TraceResult): void` | Set final result |
| `build(): CalculationTrace` | Generate trace |

### CalculatorWithTrace
- **Method:** `calculateTaxWithTrace(input): { result, trace }`
- **Input:** Same as regular calculator
- **Output:** Result + trace

---

## Business Rules & Invariants

1. Trace captures every arithmetic operation
2. Threshold values recorded at time of use
3. Formula strings human-readable
4. Trace optional in production (performance)

---

## Edge Cases

1. **Conditional logic** — Record which branch taken
2. **Early returns** — Trace shows why
3. **Zero amounts** — Explicitly recorded

---

## Tests

### trace-generation.test.ts
- Tax calculation trace
- NIC calculation trace
- SSP calculation trace
- Trace completeness

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Calculation Trace
