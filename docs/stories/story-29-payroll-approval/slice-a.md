# Slice a: Payroll Approval API with SoD Checks

**Story:** story-29-payroll-approval
**Epic:** epic-07-employer-portal
**Effort:** M
**Dependencies:** story-28-variable-pay-form

## Goal

Implement the payroll approval workflow API including summary retrieval, variance calculations, approval/rejection actions, and segregation of duties enforcement.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`, `date-fns@3.x`
- [x] SDK methods identified: `db.query`, `db.insert()`, Drizzle transactions
- [x] External service endpoints: N/A
- [x] Data contracts defined: PayrollSummary, VarianceData, ApprovalInput, ApprovalResponse
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ API Contracts → Approval endpoints
- 02-06-employer-portal-spec.md:§ Data Models → PayrollApproval
- 02-09-identity-access-spec.md:§ User Journeys → Journey 4: Segregation Check

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/api/routers/payroll-approval.ts` | create | Approval tRPC router |
| `src/lib/payroll-approval/summary.ts` | create | Summary aggregation |
| `src/lib/payroll-approval/variance.ts` | create | Period comparison logic |
| `src/lib/payroll-approval/types.ts` | create | Type definitions |
| `src/lib/payroll-approval/sod.ts` | create | Segregation of duties checks |

## Responsibilities
1. Aggregate payroll summary data for review
2. Calculate variance vs previous period
3. Identify exceptions and anomalies
4. Enforce segregation of duties (approver ≠ submitter)
5. Process approval/rejection with comments
6. Prevent duplicate approvals
7. Notify bureau of decisions
8. Maintain complete audit trail

## Contracts

### PayrollSummary Type
```typescript
// src/lib/payroll-approval/types.ts
export interface PayrollSummary {
  payRunId: string;
  period: string;
  payDate: Date;
  employer: {
    id: string;
    name: string;
    payeReference: string;
  };
  totals: {
    grossPay: number;
    tax: number;
    employeeNi: number;
    employerNi: number;
    pensionContributions: number;
    netPay: number;
    totalCost: number;
  };
  employeeCount: number;
  variance: VarianceData | null;
  exceptions: PayrollException[];
  submissionDetails: {
    submittedBy: string;
    submittedAt: Date;
    submittedByCurrentUser: boolean;
  } | null;
  currentStatus: 'draft' | 'awaiting_approval' | 'approved' | 'rejected' | 'paid';
  canApprove: boolean;
  canReject: boolean;
  sodViolation: string | null;
}

export interface VarianceData {
  grossPayChange: number;
  grossPayChangePercent: number;
  employeeCountChange: number;
  netPayChange: number;
  taxChange: number;
}

export interface PayrollException {
  type: 'new_employee' | 'significant_increase' | 'significant_decrease' | 'missing_data';
  employeeId?: string;
  employeeName?: string;
  description: string;
  severity: 'warning' | 'info';
}
```

### ApprovalInput Schema
```typescript
// src/lib/payroll-approval/types.ts
import { z } from 'zod';

export const approvalDecisionSchema = z.object({
  payRunId: z.string().uuid(),
  decision: z.enum(['approved', 'rejected']),
  comments: z.string().max(1000).optional(),
});

export type ApprovalInput = z.infer<typeof approvalDecisionSchema>;

export interface ApprovalResponse {
  approvalId: string;
  decision: 'approved' | 'rejected';
  approvedAt: Date;
  status: string;
  bureauNotified: boolean;
}
```

### Approval Router
```typescript
// src/server/api/routers/payroll-approval.ts
export const payrollApprovalRouter = router({
  // Get payroll summary for approval review
  getPayrollSummary: protectedProcedure
    .input(z.object({ payRunId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      const userId = ctx.session.user.id;
      const canApprovePayroll = ctx.session.user.canApprovePayroll;
      
      const payRun = await getPayRunWithAccessCheck(input.payRunId, employerId);
      const summary = await buildPayrollSummary(payRun);
      
      // Check segregation of duties
      const sodViolation = checkSegregationOfDuties(summary, userId);
      
      return {
        ...summary,
        canApprove: canApprovePayroll && !sodViolation && summary.currentStatus === 'awaiting_approval',
        canReject: canApprovePayroll && summary.currentStatus === 'awaiting_approval',
        sodViolation,
      };
    }),

  // Submit approval/rejection decision
  submitDecision: protectedProcedure
    .input(approvalDecisionSchema)
    .mutation(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      const userId = ctx.session.user.id;
      
      // Verify user has approval permission
      if (!ctx.session.user.canApprovePayroll) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'User does not have payroll approval permission',
        });
      }
      
      // Fetch pay run and verify access
      const payRun = await getPayRunWithAccessCheck(input.payRunId, employerId);
      
      // Verify status allows approval
      if (payRun.status !== 'awaiting_approval') {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: `Payroll cannot be approved in ${payRun.status} status`,
        });
      }
      
      // Check segregation of duties
      const sodCheck = await checkSubmitterVsApprover(input.payRunId, userId);
      if (sodCheck.isViolation) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Cannot approve payroll you submitted (segregation of duties)',
        });
      }
      
      // Create approval record and update pay run status
      return await processApprovalDecision(input, userId);
    }),
});
```

### Segregation of Duties Check
```typescript
// src/lib/payroll-approval/sod.ts
export async function checkSubmitterVsApprover(
  payRunId: string,
  approverUserId: string
): Promise<{ isViolation: boolean; reason?: string }> {
  // Get variable pay submission for this pay run
  const submission = await db.query.variablePaySubmissions.findFirst({
    where: eq(variablePaySubmissions.payRunId, payRunId),
    with: {
      submitter: true,
    },
  });
  
  if (!submission) {
    return { isViolation: false }; // No submission, no conflict
  }
  
  if (submission.submittedBy === approverUserId) {
    return {
      isViolation: true,
      reason: 'Approver cannot approve their own submission',
    };
  }
  
  return { isViolation: false };
}
```

## Business Rules & Invariants
1. Approver must have `canApprovePayroll` permission
2. Approver cannot approve their own submission (SoD)
3. Comments required for rejection; optional for approval
4. Payroll must be in "awaiting_approval" status
5. Only one approval/rejection per pay run
6. Approval transitions status to "approved"
7. Rejection transitions status to "rejected" (requires resubmission)
8. All decisions logged to audit trail

## Edge Cases
1. **No variable pay submitted** — Approve based on salary data only
2. **Previous period missing** — No variance data available
3. **Approver lacks permission** — Return 403 with specific message
4. **Concurrent approval attempts** — First wins; second gets conflict error
5. **Payroll already approved** — Return error with current status
6. **Rejection without comments** — Validation error requiring comments

## Tests

### src/lib/payroll-approval/sod.test.ts
- `should allow approval when different user submitted`: Valid case
- `should block approval when same user submitted`: SoD violation
- `should allow approval when no submission exists`: No conflict
- `should handle missing submission gracefully`: Edge case

### src/lib/payroll-approval/variance.test.ts
- `should calculate variance vs previous period`: Math accuracy
- `should handle missing previous period`: Null variance
- `should identify new employees`: Exception detection
- `should flag significant pay changes`: Threshold detection

### src/server/api/routers/payroll-approval.test.ts
- `should return payroll summary scoped to employer`: Security
- `should enforce approval permission`: RBAC
- `should enforce SoD on approval`: SoD
- `should require comments for rejection`: Validation
- `should prevent duplicate approvals`: Idempotency
- `should notify bureau on decision`: Events

## Verification
```bash
npm run test:unit src/lib/payroll-approval/sod.test.ts
npm run test:unit src/lib/payroll-approval/variance.test.ts
npm run test:unit src/server/api/routers/payroll-approval.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-06-employer-portal-spec.md § User Journeys → Journey 2: Approve Payroll
- 02-06-employer-portal-spec.md § API Contracts → Approval endpoints
- 02-09-identity-access-spec.md § User Journeys → Journey 4: Segregation Check
