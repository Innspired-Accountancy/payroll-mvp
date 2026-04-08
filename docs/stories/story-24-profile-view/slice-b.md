# Slice b: Employment Details and Bank Details Display

**Story:** story-24-profile-view
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** slice-a (personal details view)

---

## Goal

Create the employment details and bank details sections of the profile view. This displays job information, employment terms, tax details, and pension scheme information.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: profile.getEmploymentDetails()
- [x] External service endpoints: None (internal database queries)
- [x] Data contracts defined: EmploymentDetails, TaxDetails, PensionDetails
- [x] Configuration variables: None
- [x] Error scenarios identified: Employment not found, multiple employments
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Profile viewing
- 02-01-core-payroll-spec.md — Employment and tax data models
- 02-03-pension-auto-enrolment-spec.md — Pension scheme details

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/profile.ts` | update | Add employment procedures |
| `src/components/profile/employment-details.tsx` | create | Employment info section |
| `src/components/profile/bank-details-section.tsx` | create | Bank details display |
| `src/components/profile/tax-details.tsx` | create | Tax code and NI info |
| `src/components/profile/pension-details.tsx` | create | Pension scheme display |

---

## Responsibilities

1. Display employment dates (start, leaving if applicable)
2. Show job title, department, employment type
3. Display tax code and NI category letter
4. Show pension scheme and contribution rates
5. Display employer contribution rates
6. Show salary/wage information (if visible to employee)
7. Handle multiple employments (if applicable)

---

## Contracts

### profile.getEmploymentDetails
- **Method:** tRPC query `profile.getEmploymentDetails`
- **Input:** None
- **Output:**
  ```typescript
  {
    employments: [
      {
        id: string;
        employerName: string;
        jobTitle: string;
        department: string | null;
        employmentType: "full_time" | "part_time" | "casual";
        startDate: string;
        leavingDate: string | null;
        isCurrent: boolean;
        payFrequency: "weekly" | "monthly" | "four_weekly";
        taxCode: string;
        niCategory: string;       // "A", "B", "C", etc.
        pensionScheme: {
          name: string;
          employeeContribution: number;  // Percentage
          employerContribution: number;  // Percentage
          autoEnrolment: boolean;
        } | null;
        metadata: {
          lastUpdated: string;
          updatedBy: string | null;
        };
      }
    ];
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — No employment records
- **Auth:** Protected procedure

### EmploymentDetails Schema
```typescript
export const pensionSchemeSchema = z.object({
  name: z.string(),
  employeeContribution: z.number().min(0).max(100),
  employerContribution: z.number().min(0).max(100),
  autoEnrolment: z.boolean(),
});

export const employmentDetailSchema = z.object({
  id: z.string().uuid(),
  employerName: z.string(),
  jobTitle: z.string(),
  department: z.string().nullable(),
  employmentType: z.enum(["full_time", "part_time", "casual"]),
  startDate: z.string().datetime(),
  leavingDate: z.string().datetime().nullable(),
  isCurrent: z.boolean(),
  payFrequency: z.enum(["weekly", "monthly", "four_weekly"]),
  taxCode: z.string(),
  niCategory: z.string(),
  pensionScheme: pensionSchemeSchema.nullable(),
  metadata: z.object({
    lastUpdated: z.string().datetime(),
    updatedBy: z.string().nullable(),
  }),
});
```

---

## Business Rules & Invariants

1. Employee sees all active and past employments
2. Tax code displayed in HMRC format (e.g., 1257L, K475, BR)
3. Pension contributions shown as percentages
4. Auto-enrolment status clearly indicated
5. Leaving date only shown if employment ended
6. Primary employment marked as "Current"

---

## Edge Cases

1. **Multiple current employments** — Show all, mark primary
2. **No pension scheme** — Show "Not enrolled" with AE info
3. **Casual worker** — Show irregular pay indicator
4. **Tax code pending** — Show "Pending HMRC notification"
5. **Start date in future** — Show "Starts [date]"

---

## Tests

### profile.router.test.ts
- getEmploymentDetails returns all employments
- Handles multiple employments
- Shows pension details correctly

### employment-details.test.tsx
- Renders employment info
- Shows current/past status
- Pension section displays rates
- Tax code formatted

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/profile.test.ts
npx vitest run src/components/profile/employment-details.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Profile View → Employment details
- 02-05-employee-portal-spec.md § Data Models → Employee fields
- 02-01-core-payroll-spec.md § Employment data
