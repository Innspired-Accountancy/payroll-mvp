# Slice c: Bank Validation Modulus Check and Notifications

**Story:** story-25-change-requests
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** slice-b (approval workflow)

---

## Goal

Implement bank detail modulus check validation and notification system for change requests. This ensures bank details are valid before submission and keeps employees informed of request status changes.

---

## Decision Checklist

- [x] All libraries/packages named: valideerbank 1.0.x (or similar modulus check library)
- [x] SDK methods/API calls identified: bankValidation.modulusCheck(), notifications.sendChangeRequestUpdate()
- [x] External service endpoints: None (offline modulus check)
- [x] Data contracts defined: ModulusCheckResult, BankValidationError
- [x] Configuration variables: None
- [x] Error scenarios identified: Invalid sort code, invalid account number, modulus check failure
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 4: Update Bank Details (validation)
- 02-07-payments-spec.md — Bank detail validation requirements
- 02-10-audit-compliance-spec.md — Notification audit

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/validation/bank-details.ts` | create | Bank modulus check |
| `src/server/routers/change-requests.ts` | update | Add validation |
| `src/lib/notifications/change-requests.ts` | create | Notification templates |
| `src/components/change-requests/bank-validation.tsx` | create | Validation feedback |

---

## Responsibilities

1. Validate sort code exists in UK clearing
2. Perform modulus check on sort code + account number
3. Show validation errors before submission
4. Send email confirmation on request submission
5. Send status update on approval/rejection
6. Log all validation attempts

---

## Contracts

### bankValidation.modulusCheck
- **Method:** Utility function `validateBankDetails(sortCode, accountNumber)`
- **Input:**
  ```typescript
  {
    sortCode: string;         // "12-34-56"
    accountNumber: string;    // "12345678"
  }
  ```
- **Output:**
  ```typescript
  {
    valid: boolean;
    errors: string[];         // Empty if valid
    sortCodeValid: boolean;
    accountNumberValid: boolean;
    modulusCheckPassed: boolean | null;  // null if sort code not supported
    bankName: string | null;  // e.g., "Barclays Bank"
  }
  ```

### notifications.sendChangeRequestSubmitted
- **Method:** Internal service call
- **Input:**
  ```typescript
  {
    to: string;               // Employee email
    requestId: string;
    changeType: string;
    submittedAt: string;
    estimatedReviewDate: string;
  }
  ```

### notifications.sendChangeRequestStatusChanged
- **Method:** Internal service call
- **Input:**
  ```typescript
  {
    to: string;               // Employee email
    requestId: string;
    changeType: string;
    newStatus: "approved" | "rejected";
    approverNotes: string | null;
    reviewedAt: string;
  }
  ```

---

## Business Rules & Invariants

1. Sort code must exist in UK bank clearing directory
2. Account number must pass modulus check for that sort code
3. Some building societies don't support modulus check (warning only)
4. Notifications sent within 60 seconds of status change
5. Rejection must include reason for employee
6. Bank name displayed if sort code recognized

---

## Edge Cases

1. **Building society account** — May not support modulus check (warning only)
2. **New bank sort code** — Not in directory yet (manual verification)
3. **International bank** — Reject, UK only for payroll
4. **Savings account** — Valid but may reject BACS (warning)
5. **Joint account name mismatch** — Allow, payroll can use any name

---

## Tests

### bank-details.test.ts
- Validates correct bank details
- Rejects invalid sort code
- Rejects failed modulus check
- Handles building societies

### change-requests.router.test.ts
- Validates before submission
- Rejects invalid bank details
- Sends notification on submit

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/lib/validation/bank-details.test.ts
npx vitest run src/server/routers/change-requests.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Change Requests → Bank validation and notifications
- 02-05-employee-portal-spec.md § Journey 4: Update Bank Details → System validates format
- 02-07-payments-spec.md § Bank detail validation
