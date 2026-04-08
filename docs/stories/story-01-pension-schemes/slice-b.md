# Slice b: Scheme Configuration UI

**Story:** story-01-pension-schemes
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Create UI components for pension scheme configuration including form validation, provider selection, and secure credential input.

---

## Decision Checklist

- [x] UI library: React 18, Tailwind CSS, shadcn/ui components
- [x] Form handling: React Hook Form 7.51.x with Zod resolver
- [x] All form fields validated with Zod schemas from slice-a
- [x] Error display: Inline field errors + toast notifications
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Pension scheme configuration journeys
- story-01-pension-schemes/slice-a.md — API contracts to consume

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/(bureau)/employers/[id]/pension-schemes/page.tsx` | create | Scheme list page |
| `src/app/(bureau)/employers/[id]/pension-schemes/new/page.tsx` | create | Create scheme form |
| `src/app/(bureau)/employers/[id]/pension-schemes/[schemeId]/page.tsx` | create | Edit scheme form |
| `src/components/pension/SchemeForm.tsx` | create | Reusable scheme form component |
| `src/components/pension/SchemeCard.tsx` | create | Scheme summary card |
| `src/hooks/usePensionSchemes.ts` | create | tRPC query hooks |
| `src/components/pension/ProviderConfig/NestConfig.tsx` | create | NEST-specific config fields |

---

## Responsibilities

1. Render scheme list with default indicator
2. Provide form for creating/editing schemes
3. Show provider-specific configuration panels (NEST first)
4. Mask sensitive API credentials in UI
5. Display validation errors inline
6. Handle form submission with loading states

---

## Contracts

### SchemeForm Component
- **Props:** `scheme?: PensionScheme` (undefined for create mode)
- **Events:** `onSubmit: (data: PensionSchemeCreate) => void`, `onCancel: () => void`
- **Validation:** Zod schema from `src/lib/validation/pensionSchemes.ts`

### usePensionSchemes Hook
- **Queries:** `pensionSchemes.listByEmployer`, `pensionSchemes.getById`
- **Mutations:** `pensionSchemes.create`, `pensionSchemes.update`, `pensionSchemes.setDefault`

---

## Business Rules & Invariants

1. NEST config panel only shows when provider === "nest"
2. Password fields show as masked input (type="password")
3. Contribution rate inputs show % symbol for percentages
4. Default scheme marked with badge on list view
5. Cannot delete default scheme without setting new default first

---

## Edge Cases

1. **No schemes exist** — Show empty state with CTA to create
2. **API error on save** — Show toast with error message, keep form state
3. **Unsaved changes on navigate** — Show browser confirm dialog
4. **Credential reveal toggle** — Allow temporary unmask for verification

---

## Tests

### SchemeForm.test.tsx
- Render create form with all fields
- Submit valid data (calls onSubmit with correct shape)
- Show validation errors for invalid rates
- Mask password fields by default
- Toggle password visibility

---

## Verification

```bash
npm run test:unit src/components/pension/
npm run typecheck
npm run build
```

---

## Source Sections

- story-01-pension-schemes/slice-a.md → API contracts to consume
