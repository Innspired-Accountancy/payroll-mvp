# Slice a: Personal Details View with Masked Sensitive Data

**Story:** story-24-profile-view
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** story-18-portal-auth (authentication)

---

## Goal

Create the personal details view that displays employee information with sensitive data masked for security. This is a read-only view with change request buttons for editable fields.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, date-fns 3.6.x
- [x] SDK methods/API calls identified: profile.getPersonalDetails()
- [x] External service endpoints: None (internal database queries)
- [x] Data contracts defined: PersonalDetails, MaskedBankDetails
- [x] Configuration variables: MASK_BANK_ACCOUNT=true
- [x] Error scenarios identified: Employee not found, data load failure
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 4: Update Bank Details (view step)
- 02-01-core-payroll-spec.md — Employee data model
- 02-09-identity-access-spec.md — Data visibility rules

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/profile.ts` | create | Profile tRPC procedures |
| `src/app/(employee)/profile/page.tsx` | create | Profile page |
| `src/components/profile/personal-details.tsx` | create | Personal info section |
| `src/components/profile/address-display.tsx` | create | Address formatting |
| `src/lib/masking/sensitive-data.ts` | create | Data masking utilities |

---

## Responsibilities

1. Display employee name, DOB, NI number
2. Show contact information (email, phone)
3. Display address in readable format
4. Mask bank account number (show last 4 digits only)
5. Show sort code in standard format
6. Display "Request Change" buttons for editable fields
7. Show last updated timestamps

---

## Contracts

### profile.getPersonalDetails
- **Method:** tRPC query `profile.getPersonalDetails`
- **Input:** None (uses session)
- **Output:**
  ```typescript
  {
    employee: {
      id: string;
      employeeNumber: string;
      firstName: string;
      lastName: string;
      fullName: string;
      dateOfBirth: string;      // ISO date
      niNumber: string;         // AB123456C
      email: string;
      phone: string | null;
    };
    address: {
      line1: string;
      line2: string | null;
      city: string;
      postcode: string;
      country: string;
    } | null;
    bankDetails: {
      accountName: string;
      sortCode: string;         // 12-34-56
      accountNumberMasked: string;  // ****5678
    } | null;
    emergencyContact: {
      name: string;
      relationship: string | null;
      phone: string;
    } | null;
    metadata: {
      lastUpdated: string;      // ISO timestamp
      updatedBy: string | null;
    };
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — Employee record not found
- **Auth:** Protected procedure with employee session

### Masking Functions
```typescript
export function maskAccountNumber(accountNumber: string): string {
  if (accountNumber.length < 4) return "****";
  return `****${accountNumber.slice(-4)}`;
}

export function formatSortCode(sortCode: string): string {
  // Input: "123456" or "12-34-56"
  // Output: "12-34-56"
  const cleaned = sortCode.replace(/-/g, "");
  if (cleaned.length !== 6) return sortCode;
  return `${cleaned.slice(0, 2)}-${cleaned.slice(2, 4)}-${cleaned.slice(4, 6)}`;
}
```

---

## Business Rules & Invariants

1. Employee can only view own personal details
2. Bank account number always masked (last 4 digits only)
3. Sort code always displayed in XX-XX-XX format
4. NI number displayed in full (required for payslip verification)
5. Last updated timestamp shown for transparency
6. Emergency contact visible to employee

---

## Edge Cases

1. **No address on file** — Show "No address recorded" with add button
2. **No bank details** — Show "No bank details recorded" with add button
3. **No emergency contact** — Show "No emergency contact" with add button
4. **International address** — Display country field
5. **Long phone numbers** — Format with spaces for readability

---

## Tests

### profile.router.test.ts
- getPersonalDetails returns masked data
- Validates employee ownership
- Handles missing optional fields

### personal-details.test.tsx
- Renders all personal info
- Bank account masked correctly
- Sort code formatted
- Change request buttons present

### masking.test.ts
- Masks account numbers correctly
- Formats sort codes correctly
- Handles edge cases

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/profile.test.ts
npx vitest run src/components/profile/personal-details.test.tsx
npx vitest run src/lib/masking/sensitive-data.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Profile View → Personal details
- 02-05-employee-portal-spec.md § Journey 4: Update Bank Details → Profile navigation
- 02-05-employee-portal-spec.md § Security → Strict self-only data visibility
