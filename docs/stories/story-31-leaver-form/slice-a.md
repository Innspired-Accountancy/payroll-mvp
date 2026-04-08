# Slice a: Leaver Form API and Processing

**Story:** story-31-leaver-form
**Epic:** epic-07-employer-portal
**Effort:** S
**Dependencies:** story-26-client-auth

## Goal

Implement the employee leaver process API including leaving date capture, final pay calculations, P45 generation request, and status updates with bureau notification.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`, `date-fns@3.x`
- [x] SDK methods identified: `db.update()`, `db.query`, Drizzle transactions
- [x] External service endpoints: N/A
- [x] Data contracts defined: LeaverInput, LeaverResponse, FinalPayDetails
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ User Journeys → Leaver process
- 02-01-core-payroll-spec.md:§ Leaver processing requirements

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/api/routers/leaver-form.ts` | create | Leaver tRPC router |
| `src/lib/leaver-form/validation.ts` | create | Date and status validation |
| `src/lib/leaver-form/types.ts` | create | Type definitions |
| `src/lib/leaver-form/processing.ts` | create | Leaver processing logic |

## Responsibilities
1. Validate leaving date is within reasonable range
2. Determine final pay period for the employee
3. Calculate outstanding holiday/pay entitlements hint
4. Flag P45 requirement
5. Update employee status to "leaving"
6. Prevent edits to leaving employees
7. Notify bureau for P45 generation
8. Handle reversal of leaver status if needed

## Contracts

### LeaverInput Schema
```typescript
// src/lib/leaver-form/types.ts
import { z } from 'zod';

export const leavingReasonSchema = z.enum([
  'resignation',
  'termination',
  'redundancy',
  'retirement',
  'end_of_contract',
  'other',
]);

export const leaverFormSchema = z.object({
  employeeId: z.string().uuid(),
  leavingDate: z.date(),
  reason: leavingReasonSchema,
  reasonOther: z.string().max(500).optional(),
  noticeDate: z.date().optional(),
  finalWorkingDate: z.date().optional(),
  
  // Final pay details
  holidayEntitlementRemaining: z.number().min(0).optional(),
  holidayPayDue: z.number().min(0).optional(),
  outstandingExpenses: z.number().min(0).optional(),
  otherDeductions: z.number().min(0).optional(),
  
  // P45 details
  p45Required: z.boolean().default(true),
  p45Address: z.string().max(500).optional(), // If different from employee address
  
  // Final payment instructions
  paymentMethod: z.enum(['bacs', 'cheque', 'cash']).default('bacs'),
  paymentDate: z.date().optional(),
  
  // Notes
  notes: z.string().max(1000).optional(),
});

export type LeaverInput = z.infer<typeof leaverFormSchema>;
```

### LeaverResponse Type
```typescript
// src/lib/leaver-form/types.ts
export interface LeaverResponse {
  leaverId: string;
  employeeId: string;
  status: 'processed';
  processedAt: Date;
  finalPayPeriod: {
    periodId: string;
    periodName: string;
    payDate: Date;
  };
  p45Scheduled: boolean;
  bureauNotified: boolean;
  warnings: LeaverWarning[];
  nextSteps: string[];
}

export interface LeaverWarning {
  type: 'holiday_remaining' | 'negative_balance' | 'future_date' | 'past_tax_year';
  message: string;
  severity: 'warning' | 'info';
}
```

### Leaver Router
```typescript
// src/server/api/routers/leaver-form.ts
export const leaverFormRouter = router({
  // Get employee details for leaver form
  getEmployeeForLeaver: protectedProcedure
    .input(z.object({ employeeId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      
      const employee = await db.query.employees.findFirst({
        where: and(
          eq(employees.id, input.employeeId),
          eq(employees.employerId, employerId),
          eq(employees.status, 'active')
        ),
      });
      
      if (!employee) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: 'Employee not found or not active',
        });
      }
      
      // Calculate suggested values
      const currentYear = new Date().getFullYear();
      const holidayYearStart = new Date(currentYear, 3, 6); // April 6th
      const holidayEntitlement = calculateHolidayEntitlement(employee);
      const holidayTaken = await calculateHolidayTaken(employee.id, holidayYearStart);
      
      return {
        employee: {
          id: employee.id,
          name: `${employee.firstName} ${employee.lastName}`,
          employeeNumber: employee.employeeNumber,
          startDate: employee.startDate,
          department: employee.department,
        },
        suggestions: {
          holidayEntitlementRemaining: Math.max(0, holidayEntitlement - holidayTaken),
          finalPayPeriod: await getNextPayPeriod(employerId),
        },
      };
    }),

  // Process leaver
  processLeaver: protectedProcedure
    .input(leaverFormSchema)
    .mutation(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      const userId = ctx.session.user.id;
      
      // Verify user has employee edit permission
      if (!ctx.session.user.canEditEmployees) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'User does not have permission to process leavers',
        });
      }
      
      // Verify employee belongs to employer and is active
      const employee = await db.query.employees.findFirst({
        where: and(
          eq(employees.id, input.employeeId),
          eq(employees.employerId, employerId),
          eq(employees.status, 'active')
        ),
      });
      
      if (!employee) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: 'Employee not found or not active',
        });
      }
      
      // Validate leaving date
      const warnings: LeaverWarning[] = [];
      
      if (input.leavingDate < employee.startDate) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: 'Leaving date cannot be before start date',
        });
      }
      
      // Check if leaving date is in different tax year
      const taxYear = getTaxYear(input.leavingDate);
      const currentTaxYear = getTaxYear(new Date());
      if (taxYear !== currentTaxYear) {
        warnings.push({
          type: 'past_tax_year',
          message: `Leaving date is in tax year ${taxYear} - verify correct`,
          severity: 'warning',
        });
      }
      
      // Check if future date
      if (input.leavingDate > new Date()) {
        warnings.push({
          type: 'future_date',
          message: 'Leaving date is in the future - employee will remain active until then',
          severity: 'info',
        });
      }
      
      // Determine final pay period
      const finalPayPeriod = await determineFinalPayPeriod(employerId, input.leavingDate);
      
      // Create leaver record
      const leaverRecord = await db.insert(leavers).values({
        id: crypto.randomUUID(),
        employeeId: input.employeeId,
        employerId,
        leavingDate: input.leavingDate,
        reason: input.reason,
        reasonOther: input.reasonOther,
        noticeDate: input.noticeDate,
        finalWorkingDate: input.finalWorkingDate,
        holidayEntitlementRemaining: input.holidayEntitlementRemaining,
        holidayPayDue: input.holidayPayDue,
        outstandingExpenses: input.outstandingExpenses,
        otherDeductions: input.otherDeductions,
        p45Required: input.p45Required,
        p45Address: input.p45Address,
        paymentMethod: input.paymentMethod,
        paymentDate: input.paymentDate,
        finalPayPeriodId: finalPayPeriod.id,
        processedBy: userId,
        processedAt: new Date(),
        notes: input.notes,
      }).returning();
      
      // Update employee status
      await db.update(employees)
        .set({ 
          status: input.leavingDate > new Date() ? 'leaving' : 'left',
          leavingDate: input.leavingDate,
          updatedAt: new Date(),
        })
        .where(eq(employees.id, input.employeeId));
      
      // Notify bureau
      await notifyBureauOfLeaver(employerId, input.employeeId, userId, input.p45Required);
      
      return {
        leaverId: leaverRecord[0].id,
        employeeId: input.employeeId,
        status: 'processed',
        processedAt: new Date(),
        finalPayPeriod: {
          periodId: finalPayPeriod.id,
          periodName: finalPayPeriod.name,
          payDate: finalPayPeriod.payDate,
        },
        p45Scheduled: input.p45Required,
        bureauNotified: true,
        warnings,
        nextSteps: [
          input.p45Required ? 'Bureau will generate P45 after final pay' : null,
          'Employee included in final pay period',
          'Employee portal access will be deactivated',
        ].filter(Boolean),
      };
    }),
});
```

## Business Rules & Invariants
1. Leaving date cannot be before employee start date
2. Only active employees can be processed as leavers
3. P45 automatically scheduled if requested
4. Employee status changes to "leaving" (future date) or "left" (past date)
5. Bureau notified for all leaver processes
6. Final pay period determined by leaving date
7. Holiday calculations provided as hints only (bureau verifies)

## Edge Cases
1. **Future leaving date** — Status becomes "leaving" until date reached
2. **Same day leave** — Immediate status change to "left"
3. **Already processed in payroll** — Block if payslip generated for period
4. **Reversal needed** — Separate admin function (not in portal)
5. **Missing holiday data** — Allow submission with zero values
6. **Multiple leaver submissions** — Block duplicate with error

## Tests

### src/lib/leaver-form/validation.test.ts
- `should validate leaving date after start date`: Date validation
- `should reject leaving before start`: Error case
- `should detect tax year boundary`: Warning generation
- `should handle future leaving date`: Status logic

### src/server/api/routers/leaver-form.test.ts
- `should process valid leaver`: Success case
- `should require edit permission`: RBAC
- `should set correct final pay period`: Period logic
- `should schedule P45 when requested`: P45 flag
- `should notify bureau`: Events
- `should block duplicate leaver processing`: Idempotency

## Verification
```bash
npm run test:unit src/lib/leaver-form/validation.test.ts
npm run test:unit src/server/api/routers/leaver-form.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-06-employer-portal-spec.md § User Journeys → Leaver process
- 02-01-core-payroll-spec.md § Leaver processing requirements
