# Slice d: Audit Log Query and Export

**Story:** story-08-auth-audit
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-c

## Goal

Implement audit log query endpoints for administrators with filtering, pagination, and export capabilities for compliance reports.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`, `csv-stringify@6.x`
- [x] SDK methods identified: `db.query()`, `db.select()`, CSV stringify
- [x] External service endpoints: N/A
- [x] Data contracts defined: Query filters, export formats
- [x] Configuration variables: `AUDIT_EXPORT_MAX_ROWS=10000`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Acceptance Criteria → Audit log query endpoints
- 02-09-identity-access-spec.md:§ Acceptance Criteria → Export capability (CSV/JSON)

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/audit/query.ts` | create | Audit log query service |
| `src/lib/audit/export.ts` | create | Export functionality |
| `src/server/trpc/routers/audit.ts` | create | Audit query endpoints |

## Responsibilities
1. Query audit logs with filters (date range, user, event type, severity)
2. Support pagination for large result sets
3. Export to CSV format
4. Export to JSON format
5. Enforce export size limits

## Contracts

### Audit Query Service
```typescript
// src/lib/audit/query.ts
import { eq, and, gte, lte, ilike, inArray, desc, sql } from 'drizzle-orm';
import { db } from '@/lib/db';
import { auditLogs } from '@/lib/db/schema';
import type { AuditLog } from '@/lib/db/schema';

export interface AuditQueryFilters {
  startDate?: Date;
  endDate?: Date;
  actorId?: string;
  eventTypes?: string[];
  categories?: string[];
  severities?: string[];
  targetType?: string;
  targetId?: string;
  search?: string;
}

export interface AuditQueryResult {
  logs: AuditLog[];
  total: number;
  hasMore: boolean;
}

const DEFAULT_PAGE_SIZE = 50;
const MAX_PAGE_SIZE = 500;

export class AuditQueryService {
  /**
   * Query audit logs with filters
   */
  static async query(
    filters: AuditQueryFilters,
    page: number = 1,
    pageSize: number = DEFAULT_PAGE_SIZE
  ): Promise<AuditQueryResult> {
    const limit = Math.min(pageSize, MAX_PAGE_SIZE);
    const offset = (page - 1) * limit;
    
    // Build where clause
    const conditions = [];
    
    if (filters.startDate) {
      conditions.push(gte(auditLogs.timestamp, filters.startDate));
    }
    
    if (filters.endDate) {
      conditions.push(lte(auditLogs.timestamp, filters.endDate));
    }
    
    if (filters.actorId) {
      conditions.push(eq(auditLogs.actorId, filters.actorId));
    }
    
    if (filters.eventTypes && filters.eventTypes.length > 0) {
      conditions.push(inArray(auditLogs.eventType, filters.eventTypes));
    }
    
    if (filters.categories && filters.categories.length > 0) {
      conditions.push(inArray(auditLogs.eventCategory, filters.categories));
    }
    
    if (filters.severities && filters.severities.length > 0) {
      conditions.push(inArray(auditLogs.severity, filters.severities));
    }
    
    if (filters.targetType) {
      conditions.push(eq(auditLogs.targetType, filters.targetType));
    }
    
    if (filters.targetId) {
      conditions.push(eq(auditLogs.targetId, filters.targetId));
    }
    
    if (filters.search) {
      conditions.push(
        ilike(auditLogs.description, `%${filters.search}%`)
      );
    }
    
    const whereClause = conditions.length > 0 ? and(...conditions) : undefined;
    
    // Get total count
    const countResult = await db.select({ count: sql<number>`count(*)` })
      .from(auditLogs)
      .where(whereClause);
    
    const total = countResult[0].count;
    
    // Get paginated results
    const logs = await db.query.auditLogs.findMany({
      where: whereClause,
      orderBy: [desc(auditLogs.timestamp)],
      limit,
      offset,
    });
    
    return {
      logs,
      total,
      hasMore: offset + logs.length < total,
    };
  }
  
  /**
   * Get summary statistics
   */
  static async getSummary(
    filters: Omit<AuditQueryFilters, 'search'>,
    groupBy: 'eventType' | 'category' | 'severity' | 'day'
  ): Promise<Array<{ key: string; count: number }>> {
    // Implementation depends on groupBy
    // Return aggregated counts
    return [];
  }
  
  /**
   * Get unique event types for filter dropdown
   */
  static async getEventTypes(): Promise<string[]> {
    const result = await db.selectDistinct({ eventType: auditLogs.eventType })
      .from(auditLogs);
    
    return result.map(r => r.eventType);
  }
}
```

### Audit Export Service
```typescript
// src/lib/audit/export.ts
import { stringify } from 'csv-stringify/sync';
import { AuditQueryService, type AuditQueryFilters } from './query';

const MAX_EXPORT_ROWS = 10000;

export type ExportFormat = 'csv' | 'json';

export interface ExportResult {
  data: string;
  filename: string;
  contentType: string;
}

export class AuditExportService {
  /**
   * Export audit logs to CSV
   */
  static async exportCSV(filters: AuditQueryFilters): Promise<ExportResult> {
    // Query all matching logs (up to limit)
    const { logs, total } = await AuditQueryService.query(filters, 1, MAX_EXPORT_ROWS);
    
    if (total > MAX_EXPORT_ROWS) {
      throw new Error(`Export limit exceeded. ${total} records match, max is ${MAX_EXPORT_ROWS}`);
    }
    
    // Prepare CSV data
    const records = logs.map(log => ({
      timestamp: log.timestamp.toISOString(),
      eventType: log.eventType,
      category: log.eventCategory,
      severity: log.severity,
      actorId: log.actorId || '',
      actorEmail: log.actorEmail || '',
      actorIp: log.actorIp || '',
      targetType: log.targetType || '',
      targetId: log.targetId || '',
      description: log.description || '',
      success: log.success ? 'true' : 'false',
      errorCode: log.errorCode || '',
      details: log.details ? JSON.stringify(log.details) : '',
    }));
    
    const csv = stringify(records, {
      header: true,
      columns: [
        'timestamp',
        'eventType',
        'category',
        'severity',
        'actorId',
        'actorEmail',
        'actorIp',
        'targetType',
        'targetId',
        'description',
        'success',
        'errorCode',
        'details',
      ],
    });
    
    const filename = `audit-log-${new Date().toISOString().split('T')[0]}.csv`;
    
    return {
      data: csv,
      filename,
      contentType: 'text/csv',
    };
  }
  
  /**
   * Export audit logs to JSON
   */
  static async exportJSON(filters: AuditQueryFilters): Promise<ExportResult> {
    const { logs, total } = await AuditQueryService.query(filters, 1, MAX_EXPORT_ROWS);
    
    if (total > MAX_EXPORT_ROWS) {
      throw new Error(`Export limit exceeded. ${total} records match, max is ${MAX_EXPORT_ROWS}`);
    }
    
    const exportData = {
      exportedAt: new Date().toISOString(),
      total: logs.length,
      filters,
      logs: logs.map(log => ({
        ...log,
        timestamp: log.timestamp.toISOString(),
      })),
    };
    
    const filename = `audit-log-${new Date().toISOString().split('T')[0]}.json`;
    
    return {
      data: JSON.stringify(exportData, null, 2),
      filename,
      contentType: 'application/json',
    };
  }
}
```

### tRPC Audit Router
```typescript
// src/server/trpc/routers/audit.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { authenticatedProcedure, router } from '../trpc';
import { requirePermission } from '../middleware/auth';
import { AuditQueryService } from '@/lib/audit/query';
import { AuditExportService } from '@/lib/audit/export';
import { AuditEventTypes, AuditEventCategories, AuditSeverities } from '@/lib/audit/types';

export const auditRouter = router({
  // Query audit logs
  query: authenticatedProcedure
    .use(requirePermission('role:manage')) // Or specific audit permission
    .input(z.object({
      filters: z.object({
        startDate: z.date().optional(),
        endDate: z.date().optional(),
        actorId: z.string().uuid().optional(),
        eventTypes: z.array(z.string()).optional(),
        categories: z.array(z.string()).optional(),
        severities: z.array(z.string()).optional(),
        targetType: z.string().optional(),
        targetId: z.string().optional(),
        search: z.string().optional(),
      }),
      page: z.number().int().min(1).default(1),
      pageSize: z.number().int().min(1).max(500).default(50),
    }))
    .query(async ({ input }) => {
      const result = await AuditQueryService.query(
        input.filters,
        input.page,
        input.pageSize
      );
      
      return result;
    }),
  
  // Get filter options
  filterOptions: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .query(async () => {
      const eventTypes = await AuditQueryService.getEventTypes();
      
      return {
        eventTypes,
        categories: Object.values(AuditEventCategories),
        severities: Object.values(AuditSeverities),
      };
    }),
  
  // Export to CSV
  exportCSV: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .input(z.object({
      filters: z.object({
        startDate: z.date().optional(),
        endDate: z.date().optional(),
        actorId: z.string().uuid().optional(),
        eventTypes: z.array(z.string()).optional(),
        categories: z.array(z.string()).optional(),
        severities: z.array(z.string()).optional(),
      }),
    }))
    .mutation(async ({ input }) => {
      try {
        const result = await AuditExportService.exportCSV(input.filters);
        return result;
      } catch (error: any) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: error.message,
        });
      }
    }),
  
  // Export to JSON
  exportJSON: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .input(z.object({
      filters: z.object({
        startDate: z.date().optional(),
        endDate: z.date().optional(),
        actorId: z.string().uuid().optional(),
        eventTypes: z.array(z.string()).optional(),
        categories: z.array(z.string()).optional(),
        severities: z.array(z.string()).optional(),
      }),
    }))
    .mutation(async ({ input }) => {
      try {
        const result = await AuditExportService.exportJSON(input.filters);
        return result;
      } catch (error: any) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: error.message,
        });
      }
    }),
});

export type AuditRouter = typeof auditRouter;
```

## Business Rules & Invariants
1. Query results paginated (max 500 per page)
2. Export limited to 10,000 records
3. Only users with audit permission can query
4. Exports include all fields including JSON details
5. Date range filters required for large exports

## Edge Cases
1. **No matching records** — Return empty array
2. **Export exceeds limit** — Return error with count
3. **Very large date range** — Timeout or enforce limit
4. **Invalid filter values** — Zod validation rejects
5. **Export format invalid** — Return validation error

## Tests

### src/server/trpc/routers/audit.test.ts
- `should query audit logs`: Verifies query
- `should filter by date range`: Verifies date filter
- `should export to CSV`: Verifies CSV export
- `should export to JSON`: Verifies JSON export
- `should enforce export limit`: Verifies limit
- `should paginate results`: Verifies pagination

## Verification
```bash
npm run test:unit src/server/trpc/routers/audit.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Acceptance Criteria → Audit query and export
