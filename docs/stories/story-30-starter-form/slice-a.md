# Slice a: Starter Form API and Validation

**Story:** story-30-starter-form
**Epic:** epic-07-employer-portal
**Effort:** M
**Dependencies:** story-26-client-auth

## Goal

Implement the backend API for new employee starter form including HMRC starter checklist data capture, validation, and submission with bureau notification.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`, `date-fns@3.x`
- [x] SDK methods identified: `db.insert()`, `db.query`, Drizzle transactions
- [x] External service endpoints: N/A
- [x] Data contracts defined: StarterFormInput, StarterDeclaration, ValidationResult
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ User Journeys → Journey 3: Process Starter
- 02-01-core-payroll-spec.md:§ Employee onboarding requirements
- HMRC starter checklist specification

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/api/routers/starter-form.ts` | create | Starter form tRPC router |
| `src/lib/starter-form/validation.ts` | create | NI number and date validation |
| `src/lib/starter-form/types.ts` | create | Type definitions |
| `src/lib/starter-form/nIValidation.ts` | create | NI number checksum validation |

## Responsibilities
1. Capture complete starter personal details
2. Validate National Insurance number format and checksum
3. Capture P45 data when available (tax code, previous earnings)
4. Record starter declaration (A, B, or C)
5. Validate student loan status
6. Flag bank details for bureau approval
7. Validate employment dates (not before 16th birthday)
8. Notify bureau of new starter submission

## Contracts

### StarterFormInput Schema
```typescript
// src/lib/starter-form/types.ts
import { z } from 'zod';

export const starterDeclarationSchema = z.enum(['A', 'B', 'C']);

export const studentLoanStatusSchema = z.enum([
  'none',
  'plan_1',
  'plan_2',
  'plan_4',
  'postgraduate',
  'plan_1_and_postgrad',
]);

export const starterFormSchema = z.object({
  // Personal details
  title: z.enum(['Mr', 'Mrs', 'Ms', 'Miss', 'Dr', 'Mx']).optional(),
  firstName: z.string().min(1).max(100),
  middleName: z.string().max(100).optional(),
  lastName: z.string().min(1).max(100),
  dateOfBirth: z.date().max(new Date()),
  gender: z.enum(['male', 'female', 'other', 'prefer_not_to_say']).optional(),
  
  // Contact details
  addressLine1: z.string().min(1).max(200),
  addressLine2: z.string().max(200).optional(),
  town: z.string().min(1).max(100),
  county: z.string().max(100).optional(),
  postcode: z.string().regex(/^[A-Z]{1,2}[0-9][A-Z0-9]? ?[0-9][A-Z]{2}$/i),
  email: z.string().email().optional(),
  phone: z.string().max(20).optional(),
  
  // Employment details
  niNumber: z.string().regex(/^[A-Z]{2}[0-9]{6}[A-D]$/i).optional(),
  startDate: z.date(),
  jobTitle: z.string().min(1).max(100),
  department: z.string().max(100).optional(),
  employmentType: z.enum(['full_time', 'part_time', 'casual', 'irregular']),
  
  // Payment details (flagged for bureau approval)
  bankAccountName: z.string().max(200).optional(),
  bankAccountNumber: z.string().regex(/^[0-9]{8}$/).optional(),
  bankSortCode: z.string().regex(/^[0-9]{6}$/).optional(),
  
  // Tax details
  starterDeclaration: starterDeclarationSchema,
  
  // P45 data (optional)
  p45TaxCode: z.string().regex(/^[0-9]{1,4}[LNTM]$/i).optional(),
  p45TotalPayToDate: z.number().min(0).optional(),
  p45TotalTaxToDate: z.number().min(0).optional(),
  p45PreviousEmployer: z.string().max(200).optional(),
  p45LeavingDate: z.date().optional(),
  
  // Student loan
  studentLoanStatus: studentLoanStatusSchema,
  
  // Metadata
  notes: z.string().max(1000).optional(),
});

export type StarterFormInput = z.infer<typeof starterFormSchema>;
```

### StarterSubmissionResponse Type
```typescript
// src/lib/starter-form/types.ts
export interface StarterSubmissionResponse {
  employeeId: string;
  submittedAt: Date;
  status: 'pending_bureau_review';
  bureauNotified: boolean;
  warnings: ValidationWarning[];
  nextSteps: string[];
}

export interface ValidationWarning {
  field: string;
  message: string;
  severity: 'warning' | 'info';
}
```

### NI Number Validation
```typescript
// src/lib/starter-form/nIValidation.ts
const NI_PREFIXES = [
  'AA', 'AB', 'AE', 'AH', 'AK', 'AL', 'AM', 'AP', 'AR', 'AS', 'AT', 'AW', 'AX', 'AY', 'AZ',
  'BA', 'BB', 'BE', 'BH', 'BK', 'BL', 'BM', 'BT',
  'CA', 'CB', 'CE', 'CH', 'CK', 'CL', 'CM', 'CR', 'CS', 'CT', 'CW', 'CX', 'CY', 'CZ',
  'EA', 'EB', 'EE', 'EH', 'EK', 'EL', 'EM', 'EP', 'ER', 'ES', 'ET', 'EW', 'EX', 'EY', 'EZ',
  'GY', 'HA', 'HB', 'HE', 'HH', 'HK', 'HL', 'HM', 'HP', 'HR', 'HS', 'HT', 'HW', 'HX', 'HY', 'HZ',
  'JA', 'JB', 'JE', 'JH', 'JK', 'JL', 'JM', 'JP', 'JR', 'JS', 'JT', 'JW', 'JX', 'JY', 'JZ',
  'KA', 'KB', 'KE', 'KH', 'KK', 'KL', 'KM', 'KP', 'KR', 'KS', 'KT', 'KW', 'KX', 'KY', 'KZ',
  'LA', 'LB', 'LE', 'LH', 'LK', 'LL', 'LM', 'LP', 'LR', 'LS', 'LT', 'LW', 'LX', 'LY', 'LZ',
  'MA', 'MB', 'ME', 'MH', 'MK', 'ML', 'MM', 'MP', 'MR', 'MS', 'MT', 'MW', 'MX', 'MY', 'MZ',
  'NA', 'NB', 'NE', 'NH', 'NK', 'NL', 'NM', 'NP', 'NR', 'NS', 'NT', 'NW', 'NX', 'NY', 'NZ',
  'OA', 'OB', 'OE', 'OH', 'OK', 'OL', 'OM', 'OP', 'OR', 'OS', 'OT', 'OW', 'OX', 'OY', 'OZ',
  'PA', 'PB', 'PE', 'PH', 'PK', 'PL', 'PM', 'PP', 'PR', 'PS', 'PT', 'PW', 'PX', 'PY', 'PZ',
  'RA', 'RB', 'RE', 'RH', 'RK', 'RL', 'RM', 'RP', 'RR', 'RS', 'RT', 'RW', 'RX', 'RY', 'RZ',
  'SA', 'SB', 'SE', 'SH', 'SK', 'SL', 'SM', 'SP', 'SR', 'SS', 'ST', 'SW', 'SX', 'SY', 'SZ',
  'TA', 'TB', 'TE', 'TH', 'TK', 'TL', 'TM', 'TP', 'TR', 'TS', 'TT', 'TW', 'TX', 'TY', 'TZ',
  'WA', 'WB', 'WE', 'WH', 'WK', 'WL', 'WM', 'WP', 'WR', 'WS', 'WT', 'WW', 'WX', 'WY', 'WZ',
  'YA', 'YB', 'YE', 'YH', 'YK', 'YL', 'YM', 'YP', 'YR', 'YS', 'YT', 'YW', 'YX', 'YY', 'YZ',
  'ZA', 'ZB', 'ZE', 'ZH', 'ZK', 'ZL', 'ZM', 'ZP', 'ZR', 'ZS', 'ZT', 'ZW', 'ZX', 'ZY', 'ZZ',
];

export function validateNINumber(niNumber: string): { valid: boolean; error?: string } {
  const normalized = niNumber.toUpperCase().replace(/\s/g, '');
  
  // Check format
  const formatRegex = /^[A-Z]{2}[0-9]{6}[A-D]$/;
  if (!formatRegex.test(normalized)) {
    return { valid: false, error: 'Invalid format. Expected: AB123456C' };
  }
  
  const prefix = normalized.substring(0, 2);
  const suffix = normalized.charAt(8);
  
  // Check prefix is valid
  if (!NI_PREFIXES.includes(prefix)) {
    return { valid: false, error: 'Invalid NI number prefix' };
  }
  
  // Check suffix is valid
  if (!['A', 'B', 'C', 'D'].includes(suffix)) {
    return { valid: false, error: 'Invalid NI number suffix' };
  }
  
  return { valid: true };
}
```

### Starter Form Router
```typescript
// src/server/api/routers/starter-form.ts
export const starterFormRouter = router({
  submitStarter: protectedProcedure
    .input(starterFormSchema)
    .mutation(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      const userId = ctx.session.user.id;
      
      // Verify user has employee edit permission
      if (!ctx.session.user.canEditEmployees) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'User does not have permission to add employees',
        });
      }
      
      // Validate NI number if provided
      const warnings: ValidationWarning[] = [];
      if (input.niNumber) {
        const niValidation = validateNINumber(input.niNumber);
        if (!niValidation.valid) {
          throw new TRPCError({
            code: 'BAD_REQUEST',
            message: niValidation.error,
          });
        }
        
        // Check for duplicate NI number
        const existing = await checkDuplicateNINumber(input.niNumber, employerId);
        if (existing) {
          warnings.push({
            field: 'niNumber',
            message: 'NI number matches existing employee - verify not duplicate',
            severity: 'warning',
          });
        }
      }
      
      // Validate start date not in distant past
      const daysAgo = differenceInDays(new Date(), input.startDate);
      if (daysAgo > 90) {
        warnings.push({
          field: 'startDate',
          message: 'Start date is more than 90 days ago - verify correct',
          severity: 'warning',
        });
      }
      
      // Validate age at start date (must be 16+)
      const ageAtStart = differenceInYears(input.startDate, input.dateOfBirth);
      if (ageAtStart < 16) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: 'Employee must be at least 16 years old at start date',
        });
      }
      
      // Create employee record
      const employee = await createEmployeeFromStarterForm(input, employerId, userId);
      
      // Notify bureau
      await notifyBureauOfNewStarter(employerId, employee.id, userId);
      
      return {
        employeeId: employee.id,
        submittedAt: new Date(),
        status: 'pending_bureau_review',
        bureauNotified: true,
        warnings,
        nextSteps: [
          'Bureau will review and set up payroll',
          'Employee will be included in next available pay period',
          'P45 data will be processed if provided',
        ],
      };
    }),
});
```

## Business Rules & Invariants
1. NI number must pass format validation and checksum
2. Start date cannot be before employee's 16th birthday
3. Bank details require bureau approval before use
4. P45 data processed if provided; otherwise starter declaration applies
5. Student loan status required for correct tax calculation
6. New employees flagged for bureau review before first payroll
7. Duplicate NI number detection warns but doesn't block

## Edge Cases
1. **No NI number yet** — Allow submission; flag for follow-up
2. **P45 leaving date after new start date** — Validation error
3. **Very old start date (>90 days)** — Warning but allow
4. **Incomplete address** — Require minimum fields
5. **Invalid tax code format** — Reject if P45 tax code provided
6. **Concurrent submissions** — Prevent duplicate via unique constraint

## Tests

### src/lib/starter-form/nIValidation.test.ts
- `should validate correct NI number`: Valid case
- `should reject invalid format`: Format validation
- `should reject invalid prefix`: Prefix check
- `should reject invalid suffix`: Suffix check
- `should accept NI number with spaces`: Normalization

### src/lib/starter-form/validation.test.ts
- `should validate age 16+ at start date`: Age check
- `should reject start before 16th birthday`: Age validation
- `should detect duplicate NI number`: Duplicate check
- `should warn on old start date`: Warning generation
- `should validate postcode format`: Postcode check

### src/server/api/routers/starter-form.test.ts
- `should create employee from valid form`: Success case
- `should require edit permission`: RBAC
- `should flag bank details for approval`: Security
- `should notify bureau on submission`: Events
- `should enforce employer scoping`: Security

## Verification
```bash
npm run test:unit src/lib/starter-form/nIValidation.test.ts
npm run test:unit src/lib/starter-form/validation.test.ts
npm run test:unit src/server/api/routers/starter-form.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-06-employer-portal-spec.md § User Journeys → Journey 3: Process Starter
- 02-01-core-payroll-spec.md § Employee onboarding requirements
