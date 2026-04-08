# Slice a: Tenant Context Middleware

**Story:** story-06-tenant-isolation
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** story-05-rbac

## Goal

Implement tenant context extraction from JWT tokens and storage in request context for use throughout the application.

## Decision Checklist

- [x] All libraries/packages named: `async_hooks` (Node.js built-in), `drizzle-orm@latest`
- [x] SDK methods identified: `AsyncLocalStorage.run()`, `getStore()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: TenantContext type, extraction logic
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Data Models → User entity (bureau_id, employer_id)
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Tenant isolation

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/tenant/context.ts` | create | Tenant context storage |
| `src/lib/tenant/extract.ts` | create | JWT extraction logic |
| `src/server/trpc/middleware/tenant.ts` | create | tRPC tenant middleware |

## Responsibilities
1. Extract tenant IDs (bureau, employer, employee) from JWT
2. Store tenant context in AsyncLocalStorage
3. Provide helper to get current tenant context
4. Validate tenant context matches user's permissions
5. Support tenant switching for multi-tenant users

## Contracts

### Tenant Context
```typescript
// src/lib/tenant/context.ts
import { AsyncLocalStorage } from 'async_hooks';

export interface TenantContext {
  userId: string;
  userType: 'bureau' | 'client' | 'employee';
  bureauId?: string;
  employerId?: string;
  employeeId?: string;
  // Current active scope (for queries)
  activeScope?: {
    type: 'bureau' | 'employer' | 'employee';
    id: string;
  };
}

const tenantStorage = new AsyncLocalStorage<TenantContext>();

export function getTenantContext(): TenantContext | undefined {
  return tenantStorage.getStore();
}

export function runWithTenantContext<T>(
  context: TenantContext,
  callback: () => T
): T {
  return tenantStorage.run(context, callback);
}

export function setActiveScope(
  type: 'bureau' | 'employer' | 'employee',
  id: string
): void {
  const context = tenantStorage.getStore();
  if (context) {
    context.activeScope = { type, id };
  }
}

export function getActiveScope(): { type: string; id: string } | undefined {
  return tenantStorage.getStore()?.activeScope;
}
```

### JWT Extraction
```typescript
// src/lib/tenant/extract.ts
import type { AccessTokenClaims } from '@/lib/jwt/claims';
import type { TenantContext } from './context';

export function extractTenantContext(claims: AccessTokenClaims): TenantContext {
  return {
    userId: claims.sub,
    userType: claims.userType,
    bureauId: claims.bureauId,
    employerId: claims.employerId,
    employeeId: claims.employeeId,
  };
}

export function validateTenantContext(context: TenantContext): boolean {
  // Bureau users must have bureauId
  if (context.userType === 'bureau' && !context.bureauId) {
    return false;
  }
  
  // Client users must have employerId
  if (context.userType === 'client' && !context.employerId) {
    return false;
  }
  
  // Employee users must have employerId
  if (context.userType === 'employee' && !context.employerId) {
    return false;
  }
  
  return true;
}
```

### tRPC Tenant Middleware
```typescript
// src/server/trpc/middleware/tenant.ts
import { middleware } from '../trpc';
import { extractTenantContext, validateTenantContext } from '@/lib/tenant/extract';
import { runWithTenantContext } from '@/lib/tenant/context';
import { TRPCError } from '@trpc/server';

export const withTenantContext = middleware(async ({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({
      code: 'UNAUTHORIZED',
      message: 'Authentication required',
    });
  }
  
  const tenantContext = extractTenantContext(ctx.user);
  
  if (!validateTenantContext(tenantContext)) {
    throw new TRPCError({
      code: 'FORBIDDEN',
      message: 'Invalid tenant context',
    });
  }
  
  // Run resolver with tenant context
  return runWithTenantContext(tenantContext, () => 
    next({
      ctx: {
        ...ctx,
        tenant: tenantContext,
      },
    })
  );
});
```

## Business Rules & Invariants
1. Tenant context extracted from validated JWT claims
2. Bureau users have bureauId, client/employee have employerId
3. Tenant context stored in AsyncLocalStorage for request lifetime
4. Invalid tenant context rejects the request

## Edge Cases
1. **Missing bureauId for bureau user** — Reject request
2. **JWT without tenant claims** — Reject as malformed
3. **Context accessed outside request** — Return undefined
4. **Tenant switching** — Validate user has access to target tenant

## Tests

### src/lib/tenant/context.test.ts
- `should extract tenant from JWT`: Verifies extraction
- `should validate bureau user has bureauId`: Verifies validation
- `should store context in AsyncLocalStorage`: Verifies storage
- `should retrieve active scope`: Verifies scope access

## Verification
```bash
npm run test:unit src/lib/tenant/context.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Data Models → User entity
