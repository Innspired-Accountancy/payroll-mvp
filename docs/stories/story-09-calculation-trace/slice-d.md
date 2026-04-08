# Slice d: Explain Calculation UI

**Story:** story-09-calculation-trace
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-b, slice-c

---

## Goal

Create the "Explain Calculation" UI component displaying trace in hierarchical, user-friendly format.

---

## Decision Checklist

- [x] UI: Hierarchical tree/accordion view
- [x] Components: Section, Step, Formula display
- [x] Export: PDF export for support
- [x] Mobile: Responsive design
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Explain calculation
- 02-05-employee-portal-spec.md — Portal UI patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/calculation/explain-calculation.tsx` | create | Main component |
| `src/components/calculation/trace-section.tsx` | create | Section display |
| `src/components/calculation/trace-step.tsx` | create | Step display |
| `src/components/calculation/formula-display.tsx` | create | Formula renderer |
| `src/app/(bureau)/pay-run/[id]/explain/page.tsx` | create | Explain page |

---

## Responsibilities

1. Display trace hierarchy
2. Show formulas with values substituted
3. Highlight key calculations
4. Support PDF export
5. Mobile responsive

---

## Contracts

### ExplainCalculation Props
| Prop | Type | Description |
|------|------|-------------|
| payslip_id | uuid | Payslip to explain |
| initial_section | string | Open this section first |

### TraceSection Component Props
| Prop | Type | Description |
|------|------|-------------|
| section | TraceSection | Section data |
| depth | number | Nesting level |
| expanded | boolean | Default expanded |

---

## Business Rules & Invariants

1. Tax section always visible
2. NIC section visible if NIC > 0
3. Formula shows substituted values
4. Final result highlighted

---

## Edge Cases

1. **Empty section** — Collapsed by default
2. **Very deep nesting** — Max depth limit
3. **Missing trace** — Show error message

---

## Tests

### explain-calculation.test.tsx
- Render with trace
- Section expand/collapse
- Formula display
- PDF export

---

## Verification

```bash
npm run test:unit
npm run typecheck
npm run build
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Explain Calculation
