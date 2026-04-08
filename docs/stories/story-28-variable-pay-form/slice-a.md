# Slice a: Variable Pay Submission API and Validation

**Story:** story-28-variable-pay-form
**Epic:** epic-07-employer-portal
**Effort:** M
**Dependencies:** story-26-client-auth

## Goal

Implement the backend API for variable pay data submission including employee listing, validation rules, draft saving, and final submission with bureau notification.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`, `date-fns@3.x`
- [x] SDK methods identified: `db.insert()`, `db.update()`, Drizzle transactions
- [x] External service endpoints: N/A
- [x] Data contracts defined: VariablePayInput, ValidationResult, SubmissionResponse
- [x] Configuration variables: `NMW_RATES_2026` (hourly rates by age band)
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ API Contracts → Variable pay endpoints
- 02-06-employer-portal-spec.md:§ Data Models → VariablePaySubmission
- 02-01-core-payroll-spec.md:§ National Minimum Wage requirements

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/api/routers/variable-pay.ts` | create | Variable pay tRPC router |
| `src/lib/variable-pay/validation.ts` | create | NMW and limit validation |
| `src/lib/variable-pay/types.ts` | create | Type definitions |
| `src/lib/variable-pay/calculations.ts` | create | Rate calculations |
| `src/lib/variable-pay/submission.ts` | create | Submission logic |

## Responsibilities
1. List employees with current period variable pay data
2. Validate NMW compliance for hourly workers
3. Validate reasonable limits for overtime and bonuses
4. Save draft data without triggering workflows
5. Submit final data with bureau notification
6. Prevent submission after deadline
7. Track submission history and audit trail

## Contracts

### EmployeeVariablePay Type
```typescript
// src/lib/variable-pay/types.ts
export interface EmployeeVariablePay {
  employeeId: string;
  employeeNumber: string;
  firstName: string;
  lastName: string;
  department: string | null;
  existingData: {
    hoursWorked: number | null;
    overtimeHours: number | null;
    bonus: number | null;
  };
  validations: ValidationResult[];
}

export interface ValidationResult {
  field: 'hoursWorked' | 'overtimeHours' | 'bonus';
  type: 'error' | 'warning';
  message: string;
  code: string;
}
```

### VariablePayInput Schema
```typescript
// src/lib/variable-pay/validation.ts
import { z } from 'zod';

export const variablePayEntrySchema = z.object({
  employeeId: z.string().uuid(),
  hoursWorked: z.number().min(0).max(500).nullable(),
  overtimeHours: z.number().min(0).max(200).nullable(),
  bonus: z.number().min(0).max(100000).nullable(),
});

export const variablePaySubmissionSchema = z.object({
  payPeriodId: z.string().uuid(),
  employeeData: z.array(variablePayEntrySchema),
});

export type VariablePayInput = z.infer<typeof variablePaySubmissionSchema>;
export type VariablePayEntry = z.infer<typeof variablePayEntrySchema>;
```

### SubmissionResponse Type
```typescript
// src/lib/variable-pay/types.ts
export interface SubmissionResponse {
  submissionId: string;
  status: 'draft_saved' | 'submitted';
  submittedAt: Date;
  bureauNotified: boolean;
  validationErrors: ValidationResult[];
  summary: {
    totalEmployees: number;
    totalHours: number;
    totalOvertime: number;
    totalBonus: number;
  };
}
```

### Variable Pay Router
```typescript
// src/server/api/routers/variable-pay.ts
export const variablePayRouter = router({
  // Get variable pay form data for pay period
  getFormData: protectedProcedure
    .input(z.object({ payPeriodId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      
      // Verify pay period belongs to employer
      const payPeriod = await verifyPayPeriodAccess(input.payPeriodId, employerId);
      
      // Get all active employees for employer
      const employees = await getEmployeesForVariablePay(employerId, input.payPeriodId);
      
      return {
        payPeriod: {
          id: payPeriod.id,
          period: format(payPeriod.periodStart, 'MMMM yyyy'),
          cutOff: payPeriod.cutOffDate,
        },
        employees,
        canSubmit: new Date() <= payPeriod.cutOffDate,
      };
    }),

  // Save draft (no validation errors required)
  saveDraft: protectedProcedure
    .input(variablePaySubmissionSchema)
    .mutation(async ({ input, ctx }) => {
      return saveVariablePayData(input, ctx.session.user.id, 'draft');
    }),

  // Submit final (requires no validation errors)
  submit: protectedProcedure
    .input(variablePaySubmissionSchema)
    .mutation(async ({ input, ctx }) => {
      // Validate no blocking errors
      const validationResults = await validateSubmission(input);
      const hasErrors = validationResults.some(r => r.type === 'error');
      
      if (hasErrors) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: 'Cannot submit with validation errors',
        });
      }
      
      return saveVariablePayData(input, ctx.session.user.id, 'submitted');
    }),
});
```

### NMW Validation
```typescript
// src/lib/variable-pay/validation.ts
const NMW_RATES_2026 = {
  '23+': 11.44,
  '21-22': 11.44,
  '18-20': 8.60,
  '16-17': 6.40,
  'apprentice': 6.40,
};

export function validateNMW(
  hourlyRate: number,
  age: number,
  isApprentice: boolean
): ValidationResult | null {
  const rateKey = isApprentice ? 'apprentice' : getAgeBand(age);
  const minimumRate = NMW_RATES_2026[rateKey];
  
  if (hourlyRate < minimumRate) {
    return {
      field: 'hoursWorked',
      type: 'error',
      message: `Rate £${hourlyRate.toFixed(2)}/hr below NMW (£${minimumRate}/hr)`,
      code: 'NMW_VIOLATION',
    };
  }
  return null;
}
```

## Business Rules & Invariants
1. NMW validation applies to hourly-paid employees only
2. Hours worked limited to 500 per period (prevents data entry errors)
3. Overtime limited to 200 hours per period
4. Bonus limited to £100,000 per employee per period
5. Submission blocked after cut-off date
6. Draft saves do not trigger bureau notifications
7. Final submission notifies bureau via event
8. Each employee can only have one submission per period

## Edge Cases
1. **Employee left during period** — Show with partial period indicator
2. **New starter with no history** — No variance comparison available
3. **Zero hours for all employees** — Allow submission with warning
4. **Extreme hours entry (typo)** — Caught by validation limits
5. **Concurrent draft saves** — Last write wins; optimistic locking
6. **Pay period closed** — Block submission; show read-only view
7. **Employee salary (not hourly)** — Hide hours fields

## Tests

### src/lib/variable-pay/validation.test.ts
- `should validate NMW compliance for age 23+`: NMW check
- `should validate apprentice rate`: Apprentice check
- `should flag hours exceeding limit`: Upper bound
- `should flag negative hours`: Lower bound
- `should allow zero hours`: Edge case
- `should calculate hourly rate correctly`: Rate math

### src/lib/variable-pay/submission.test.ts
- `should save draft without validation`: Draft flow
- `should reject submission with NMW errors`: Validation gate
- `should allow submission with only warnings`: Warning tolerance
- `should block submission after deadline`: Deadline enforcement
- `should notify bureau on final submission`: Event firing
- `should enforce employer scoping`: Security

## Verification
```bash
npm run test:unit src/lib/variable-pay/validation.test.ts
npm run test:unit src/lib/variable-pay/submission.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-06-employer-portal-spec.md § User Journeys → Journey 1: Submit Variable Pay Data
- 02-06-employer-portal-spec.md § API Contracts → Variable pay endpoints
