# Slice b: Database Query Tenant Filtering

**Story:** story-06-tenant-isolation
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-a

## Goal

Implement automatic tenant filtering for all database queries using Drizzle ORM, ensuring users can only access data within their tenant scope.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`
- [x] SDK methods identified: SQL `WHERE` clause building, query composition
- [x] External service endpoints: N/A
- [x] Data contracts defined: Tenant filter options, query builder extensions
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Multi-tenant data isolation
- 02-09-identity-access-spec.md:§ Data Models → All entities with tenant fields

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/filters/tenant.ts` | create | Tenant filter utilities |
| `src/lib/db/extensions/with-tenant.ts` | create | Extended query builder |

## Responsibilities
1. Add tenant WHERE clauses to all queries
2. Support tenant-scoped queries automatically
3. Provide helpers for cross-tenant queries (admin only)
4. Enforce tenant filtering at the query level
5. Block queries without tenant context

## Contracts

### Tenant Filters
```typescript
// src/lib/db/filters/tenant.ts
import { eq, and, SQL } from 'drizzle-orm';
import type { PgTable } from 'drizzle-orm/pg-core';
import { getTenantContext } from '@/lib/tenant/context';

export interface TenantColumnConfig {
  bureauIdColumn?: string;
  employerIdColumn?: string;
}

export function withTenantFilter<T extends PgTable>(
  table: T,
  baseWhere?: SQL
): SQL | undefined {
  const context = getTenantContext();
  
  if (!context) {
    throw new Error('Tenant context required for query');
  }
  
  const tenantFilter = buildTenantFilter(table, context);
  
  if (!tenantFilter) {
    return baseWhere;
  }
  
  return baseWhere ? and(baseWhere, tenantFilter) : tenantFilter;
}

function buildTenantFilter<T extends PgTable>(
  table: T,
  context: ReturnType<typeof getTenantContext>
): SQL | undefined {
  if (!context) return undefined;
  
  const tableConfig = getTableTenantConfig(table);
  
  // Determine which tenant column to filter on
  if (context.userType === 'bureau' && context.bureauId && tableConfig.bureauIdColumn) {
    return eq(table[tableConfig.bureauIdColumn as keyof T] as any, context.bureauId);
  }
  
  if (context.employerId && tableConfig.employerIdColumn) {
    return eq(table[tableConfig.employerIdColumn as keyof T] as any, context.employerId);
  }
  
  return undefined;
}

// Table configuration mapping
const tableTenantConfig: Map<string, TenantColumnConfig> = new Map([
  ['users', { bureauIdColumn: 'bureau_id', employerIdColumn: 'employer_id' }],
  ['employers', { bureauIdColumn: 'bureau_id' }],
  ['employees', { employerIdColumn: 'employer_id' }],
  ['payroll_runs', { employerIdColumn: 'employer_id' }],
  ['payslips', { employerIdColumn: 'employer_id' }],
]);

function getTableTenantConfig<T extends PgTable>(table: T): TenantColumnConfig {
  const tableName = table[Symbol.for('drizzle:name') as any] as string;
  return tableTenantConfig.get(tableName) || {};
}
```

### Extended Query Builder
```typescript
// src/lib/db/extensions/with-tenant.ts
import { db } from '@/lib/db';
import { withTenantFilter } from '../filters/tenant';

/**
 * Wrapper for database queries that applies tenant filtering
 */
export function tenantQuery() {
  const context = getTenantContext();
  
  if (!context) {
    throw new Error('Tenant context required');
  }
  
  return {
    // Wrap findMany to add tenant filter
    findMany: async <T extends PgTable>(
      table: T,
      config?: any
    ) => {
      const tenantFilter = withTenantFilter(table);
      
      if (tenantFilter) {
        config = {
          ...config,
          where: config?.where 
            ? and(config.where, tenantFilter)
            : tenantFilter,
        };
      }
      
      return db.query[table._.name].findMany(config);
    },
    
    // Wrap findFirst
    findFirst: async <T extends PgTable>(
      table: T,
      config?: any
    ) => {
      const tenantFilter = withTenantFilter(table);
      
      if (tenantFilter) {
        config = {
          ...config,
          where: config?.where 
            ? and(config.where, tenantFilter)
            : tenantFilter,
        };
      }
      
      return db.query[table._.name].findFirst(config);
    },
  };
}
```

## Business Rules & Invariants
1. All queries must have tenant context
2. Bureau users filter by bureau_id
3. Client/employee users filter by employer_id
4. Queries without tenant context throw error
5. Cross-tenant queries require explicit admin override

## Edge Cases
1. **No tenant context** — Throw error before query
2. **Table without tenant column** — Allow query (system tables)
3. **Admin needs cross-tenant data** — Use separate unfiltered query path
4. **Tenant ID doesn't exist** — Query returns empty (no data leakage)

## Tests

### src/lib/db/filters/tenant.test.ts
- `should add bureau filter for bureau user`: Verifies bureau filter
- `should add employer filter for client user`: Verifies employer filter
- `should throw without tenant context`: Verifies context requirement
- `should combine with existing where clause`: Verifies composition

## Verification
```bash
npm run test:unit src/lib/db/filters/tenant.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → Tenant isolation
