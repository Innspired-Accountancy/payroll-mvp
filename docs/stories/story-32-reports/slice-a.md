# Slice a: Report API with Data Aggregation

**Story:** story-32-reports
**Epic:** epic-07-employer-portal
**Effort:** M
**Dependencies:** story-26-client-auth

## Goal

Implement the backend API for employer self-service reports including payroll summaries, cost breakdowns, variance analysis, and data export preparation with optimized query performance.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`, `date-fns@3.x`
- [x] SDK methods identified: `db.query`, `db.select`, Drizzle aggregations
- [x] External service endpoints: N/A
- [x] Data contracts defined: ReportType, ReportParameters, ReportData
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ User Journeys → Journey 4: View Reports
- 02-11-reporting-documents-spec.md:§ Report types and requirements

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/api/routers/reports.ts` | create | Reports tRPC router |
| `src/lib/reports/aggregations.ts` | create | Data aggregation queries |
| `src/lib/reports/types.ts` | create | Report type definitions |
| `src/lib/reports/filters.ts` | create | Date range and filter logic |

## Responsibilities
1. Generate payroll summary reports by period
2. Calculate departmental cost breakdowns
3. Provide employee-level cost summaries
4. Compute period-over-period variance
5. Aggregate tax and NI summaries
6. Calculate pension contribution totals
7. Support date range filtering
8. Optimize queries for <10 second generation
9. Enforce strict employer scoping

## Contracts

### ReportType Enum
```typescript
// src/lib/reports/types.ts
export const reportTypeSchema = z.enum([
  'payroll_summary',
  'department_costs',
  'employee_costs',
  'variance_analysis',
  'tax_summary',
  'pension_summary',
]);

export type ReportType = z.infer<typeof reportTypeSchema>;
```

### ReportParameters Schema
```typescript
// src/lib/reports/types.ts
export const reportParametersSchema = z.object({
  reportType: reportTypeSchema,
  startDate: z.date(),
  endDate: z.date(),
  department: z.string().optional(),
  employeeId: z.string().uuid().optional(),
  includeZeroAmounts: z.boolean().default(true),
});

export type ReportParameters = z.infer<typeof reportParametersSchema>;
```

### ReportData Type
```typescript
// src/lib/reports/types.ts
export interface ReportData {
  reportType: ReportType;
  generatedAt: Date;
  period: {
    startDate: Date;
    endDate: Date;
  };
  summary: Record<string, number>;
  rows: ReportRow[];
  totals: Record<string, number>;
  currency: 'GBP';
}

export interface ReportRow {
  id: string;
  label: string;
  period?: string;
  values: Record<string, number | string>;
  subRows?: ReportRow[];
}

// Type-specific report data
export interface PayrollSummaryReport extends ReportData {
  reportType: 'payroll_summary';
  rows: Array<{
    id: string;
    label: string; // Period name
    payDate: Date;
    employeeCount: number;
    grossPay: number;
    tax: number;
    employeeNi: number;
    employerNi: number;
    pension: number;
    netPay: number;
    totalCost: number;
  }>;
}

export interface DepartmentCostReport extends ReportData {
  reportType: 'department_costs';
  rows: Array<{
    id: string;
    label: string; // Department name
    employeeCount: number;
    grossPay: number;
    employerNi: number;
    pension: number;
    totalCost: number;
    percentageOfTotal: number;
  }>;
}

export interface VarianceReport extends ReportData {
  reportType: 'variance_analysis';
  rows: Array<{
    id: string;
    label: string; // Period name
    currentAmount: number;
    previousAmount: number;
    variance: number;
    variancePercent: number;
    trend: 'up' | 'down' | 'stable';
  }>;
}
```

### Reports Router
```typescript
// src/server/api/routers/reports.ts
export const reportsRouter = router({
  // Generate report
  generateReport: protectedProcedure
    .input(reportParametersSchema)
    .query(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      
      // Validate date range (max 24 months)
      const monthsDiff = differenceInMonths(input.endDate, input.startDate);
      if (monthsDiff > 24) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: 'Date range cannot exceed 24 months',
        });
      }
      
      // Generate appropriate report type
      switch (input.reportType) {
        case 'payroll_summary':
          return generatePayrollSummaryReport(employerId, input);
        case 'department_costs':
          return generateDepartmentCostReport(employerId, input);
        case 'employee_costs':
          return generateEmployeeCostReport(employerId, input);
        case 'variance_analysis':
          return generateVarianceReport(employerId, input);
        case 'tax_summary':
          return generateTaxSummaryReport(employerId, input);
        case 'pension_summary':
          return generatePensionSummaryReport(employerId, input);
        default:
          throw new TRPCError({
            code: 'BAD_REQUEST',
            message: 'Unknown report type',
          });
      }
    }),

  // Get available report types with descriptions
  getReportTypes: protectedProcedure
    .query(async () => {
      return [
        {
          id: 'payroll_summary',
          name: 'Payroll Summary',
          description: 'Summary of payroll runs with totals and employee counts',
          requiresDateRange: true,
        },
        {
          id: 'department_costs',
          name: 'Department Costs',
          description: 'Cost breakdown by department or location',
          requiresDateRange: true,
        },
        {
          id: 'employee_costs',
          name: 'Employee Costs',
          description: 'Individual employee cost summary',
          requiresDateRange: true,
        },
        {
          id: 'variance_analysis',
          name: 'Variance Analysis',
          description: 'Period-over-period variance comparison',
          requiresDateRange: true,
        },
        {
          id: 'tax_summary',
          name: 'Tax & NI Summary',
          description: 'Summary of tax and National Insurance deductions',
          requiresDateRange: true,
        },
        {
          id: 'pension_summary',
          name: 'Pension Summary',
          description: 'Pension contribution breakdown',
          requiresDateRange: true,
        },
      ];
    }),
});
```

### Payroll Summary Report Generation
```typescript
// src/lib/reports/aggregations.ts
export async function generatePayrollSummaryReport(
  employerId: string,
  params: ReportParameters
): Promise<PayrollSummaryReport> {
  // Query pay runs within date range
  const payRuns = await db.query.payRuns.findMany({
    where: and(
      eq(payRuns.employerId, employerId),
      gte(payRuns.payDate, params.startDate),
      lte(payRuns.payDate, params.endDate),
      eq(payRuns.status, 'paid')
    ),
    with: {
      payPeriod: true,
      payslips: true,
    },
    orderBy: [desc(payRuns.payDate)],
  });
  
  const rows = payRuns.map(pr => {
    const stats = calculatePayRunStats(pr.payslips);
    
    return {
      id: pr.id,
      label: format(pr.payPeriod.periodStart, 'MMMM yyyy'),
      payDate: pr.payDate,
      employeeCount: pr.payslips.length,
      grossPay: stats.grossPay,
      tax: stats.tax,
      employeeNi: stats.employeeNi,
      employerNi: stats.employerNi,
      pension: stats.pension,
      netPay: stats.netPay,
      totalCost: stats.grossPay + stats.employerNi + stats.pension,
    };
  });
  
  // Calculate totals
  const totals = rows.reduce((acc, row) => ({
    grossPay: acc.grossPay + row.grossPay,
    tax: acc.tax + row.tax,
    employeeNi: acc.employeeNi + row.employeeNi,
    employerNi: acc.employerNi + row.employerNi,
    pension: acc.pension + row.pension,
    netPay: acc.netPay + row.netPay,
    totalCost: acc.totalCost + row.totalCost,
  }), {
    grossPay: 0, tax: 0, employeeNi: 0, employerNi: 0,
    pension: 0, netPay: 0, totalCost: 0,
  });
  
  return {
    reportType: 'payroll_summary',
    generatedAt: new Date(),
    period: {
      startDate: params.startDate,
      endDate: params.endDate,
    },
    summary: {
      totalPayRuns: rows.length,
      averageEmployeeCount: rows.length > 0 
        ? Math.round(rows.reduce((a, r) => a + r.employeeCount, 0) / rows.length)
        : 0,
    },
    rows,
    totals,
    currency: 'GBP',
  };
}
```

## Business Rules & Invariants
1. All reports strictly scoped to user's employer
2. Date range limited to 24 months maximum
3. Only "paid" pay runs included in reports
4. Currency always GBP for UK payroll
5. Totals calculated from individual rows (not separate query)
6. Zero-amount rows included by default (configurable)
7. Reports generated in real-time (no caching in slice-a)

## Edge Cases
1. **No data in date range** — Return empty rows with zero totals
2. **Single period selected** — Variance report shows N/A
3. **Very large date range** — Enforce 24-month limit
4. **Department filter with no matches** — Return empty
5. **Employee left during period** — Include prorated amounts
6. **Report timeout** — Return partial data with warning

## Tests

### src/lib/reports/aggregations.test.ts
- `should calculate payroll summary correctly`: Aggregation accuracy
- `should scope data to employer`: Security
- `should filter by date range`: Date filtering
- `should calculate variance correctly`: Variance math
- `should handle empty result sets`: Empty state
- `should complete within 10 seconds`: Performance

### src/server/api/routers/reports.test.ts
- `should require authentication`: Auth
- `should enforce employer scoping`: Security
- `should reject date range > 24 months`: Validation
- `should return all report types`: Metadata
- `should generate each report type`: Coverage

## Verification
```bash
npm run test:unit src/lib/reports/aggregations.test.ts
npm run test:unit src/server/api/routers/reports.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-06-employer-portal-spec.md § User Journeys → Journey 4: View Reports
- 02-11-reporting-documents-spec.md § Report types
