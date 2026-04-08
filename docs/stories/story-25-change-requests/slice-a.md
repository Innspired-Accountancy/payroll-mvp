# Slice a: Change Request Form with Validation

**Story:** story-25-change-requests
**Epic:** epic-06-employee-portal
**Effort:** M
**Dependencies:** story-24-profile-view (profile viewing)

---

## Goal

Create the change request form for updating personal details. This covers address changes, phone number updates, and bank detail changes with appropriate validation for each change type.

---

## Decision Checklist

- [x] All libraries/packages named: React Hook Form 7.51.x, Zod 3.22.x
- [x] SDK methods/API calls identified: changeRequests.create(), validation.validatePostcode()
- [x] External service endpoints: Postcode lookup API (optional)
- [x] Data contracts defined: ChangeRequestInput, AddressChange, BankChange
- [x] Configuration variables: POSTCODE_API_KEY (optional)
- [x] Error scenarios identified: Invalid bank details, invalid postcode, duplicate request
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 4: Update Bank Details
- 02-05-employee-portal-spec.md — Data Models: PersonalDetailChangeRequest
- 02-09-identity-access-spec.md — Approval workflow requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/change-requests.ts` | create | Change request procedures |
| `src/app/(employee)/profile/change/[type]/page.tsx` | create | Change form pages |
| `src/components/change-requests/address-form.tsx` | create | Address change form |
| `src/components/change-requests/bank-form.tsx` | create | Bank details form |
| `src/components/change-requests/phone-form.tsx` | create | Phone change form |
| `src/lib/validation/change-requests.ts` | create | Validation schemas |

---

## Responsibilities

1. Display current value for reference
2. Provide form for new value entry
3. Validate inputs based on change type
4. Validate sort code format (XX-XX-XX)
5. Validate account number length (8 digits)
6. Validate UK postcode format
7. Validate UK mobile phone format
8. Prevent duplicate pending requests

---

## Contracts

### changeRequests.create
- **Method:** tRPC mutation `changeRequests.create`
- **Input:**
  ```typescript
  {
    changeType: "address" | "phone" | "bank_details";
    newValue: {
      // For address:
      line1?: string;
      line2?: string;
      city?: string;
      postcode?: string;
      // For phone:
      phone?: string;
      // For bank:
      accountName?: string;
      sortCode?: string;
      accountNumber?: string;
    };
    reason?: string;            // Optional explanation
  }
  ```
- **Output:**
  ```typescript
  {
    success: boolean;
    request: {
      id: string;
      status: "pending_approval";
      submittedAt: string;
      estimatedReviewDate: string;  // 5 business days
    };
  }
  ```
- **Errors:**
  - `CONFLICT` — Pending request already exists for this field
  - `VALIDATION_ERROR` — Invalid input format
  - `BAD_REQUEST` — No changes detected (same value)
- **Auth:** Protected procedure

### Validation Schemas
```typescript
export const addressChangeSchema = z.object({
  line1: z.string().min(1, "Address line 1 is required").max(100),
  line2: z.string().max(100).optional(),
  city: z.string().min(1, "City is required").max(50),
  postcode: z.string().regex(
    /^[A-Z]{1,2}\d[A-Z\d]? ?\d[A-Z]{2}$/i,
    "Enter a valid UK postcode"
  ),
});

export const bankChangeSchema = z.object({
  accountName: z.string().min(1, "Account name is required").max(100),
  sortCode: z.string().regex(
    /^\d{2}-?\d{2}-?\d{2}$/,
    "Enter a valid sort code (e.g., 12-34-56)"
  ),
  accountNumber: z.string().regex(
    /^\d{8}$/,
    "Account number must be 8 digits"
  ),
});

export const phoneChangeSchema = z.object({
  phone: z.string().regex(
    /^07\d{9}$/,
    "Enter a valid UK mobile number (11 digits starting with 07)"
  ),
});
```

---

## Business Rules & Invariants

1. Only one pending request per field type at a time
2. Address requires line1, city, and valid postcode
3. Bank sort code must be XX-XX-XX format (validated)
4. Account number must be exactly 8 digits
5. Phone must be UK mobile format (07XXXXXXXXX)
6. Estimated review: 5 business days
7. Change request logged with before/after values

---

## Edge Cases

1. **Same value submitted** — Reject with "No changes detected"
2. **International address** — Support via manual entry (postcode optional)
3. **Joint account** — Allow, account name can differ from employee name
4. **Building society** — 9-digit account number (support both 8 and 9)
5. **Landline phone** — Mobile only for MFA, reject landline format

---

## Tests

### change-requests.router.test.ts
- create request with valid address
- create request with valid bank details
- Rejects duplicate pending request
- Validates sort code format
- Validates postcode format

### address-form.test.tsx
- Renders current address
- Validates required fields
- Postcode validation
- Submit calls API

### bank-form.test.tsx
- Renders current bank details
- Sort code formatting
- Account number validation
- Shows security warning

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/change-requests.test.ts
npx vitest run src/components/change-requests/address-form.test.tsx
npx vitest run src/components/change-requests/bank-form.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Change Requests → Update requests
- 02-05-employee-portal-spec.md § Journey 4: Update Bank Details
- 02-05-employee-portal-spec.md § API Contracts → POST /api/v1/portal/bank-change-request
