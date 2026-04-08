# Slice d: Tenant Switch and Validation

**Story:** story-06-tenant-isolation
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-a

## Goal

Implement tenant switching for bureau users who have access to multiple employers, with validation that the user has permission to access the target tenant.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`
- [x] SDK methods identified: `db.query()`, context manipulation
- [x] External service endpoints: N/A
- [x] Data contracts defined: Tenant switch request, validation result
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ User Journeys → Journey 2 (assign employer access scope)

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/tenant/switch.ts` | create | Tenant switching logic |
| `src/server/trpc/routers/tenant.ts` | create | Tenant switch endpoints |

## Responsibilities
1. Validate user has access to target tenant
2. Update active scope in context
3. Support switching between bureau and employer scopes
4. List available tenants for user
5. Audit tenant switches

## Contracts

### Tenant Switch Service
```typescript
// src/lib/tenant/switch.ts
import { eq, and } from 'drizzle-orm';
import { db } from '@/lib/db';
import { userRoleAssignments, employers } from '@/lib/db/schema';
import { getTenantContext, setActiveScope } from './context';

export interface AvailableTenant {
  id: string;
  type: 'bureau' | 'employer';
  name: string;
}

export interface SwitchResult {
  success: boolean;
  error?: string;
  newScope?: {
    type: 'bureau' | 'employer';
    id: string;
  };
}

export class TenantSwitchService {
  /**
   * Get list of tenants the user can access
   */
  static async getAvailableTenants(userId: string): Promise<AvailableTenant[]> {
    const context = getTenantContext();
    if (!context) {
      throw new Error('Tenant context required');
    }
    
    const tenants: AvailableTenant[] = [];
    
    // Bureau users can access their bureau and assigned employers
    if (context.userType === 'bureau' && context.bureauId) {
      // Add bureau as option
      tenants.push({
        id: context.bureauId,
        type: 'bureau',
        name: 'Bureau (All Employers)',
      });
      
      // Get assigned employers from role assignments
      const assignments = await db.query.userRoleAssignments.findMany({
        where: eq(userRoleAssignments.userId, userId),
        columns: { scopeType: true, scopeId: true },
      });
      
      const employerIds = [...new Set(
        assignments
          .filter(a => a.scopeType === 'employer')
          .map(a => a.scopeId)
      )];
      
      if (employerIds.length > 0) {
        const employerList = await db.query.employers.findMany({
          where: inArray(employers.id, employerIds),
          columns: { id: true, name: true },
        });
        
        for (const emp of employerList) {
          tenants.push({
            id: emp.id,
            type: 'employer',
            name: emp.name,
          });
        }
      }
    }
    
    // Client/employee users only have their employer
    if (context.employerId) {
      const employer = await db.query.employers.findFirst({
        where: eq(employers.id, context.employerId),
        columns: { name: true },
      });
      
      if (employer) {
        tenants.push({
          id: context.employerId,
          type: 'employer',
          name: employer.name,
        });
      }
    }
    
    return tenants;
  }
  
  /**
   * Switch to a different tenant
   */
  static async switchToTenant(
    userId: string,
    targetType: 'bureau' | 'employer',
    targetId: string
  ): Promise<SwitchResult> {
    const context = getTenantContext();
    if (!context) {
      return { success: false, error: 'No tenant context' };
    }
    
    // Validate user can access this tenant
    const hasAccess = await this.validateTenantAccess(userId, targetType, targetId);
    
    if (!hasAccess) {
      return { success: false, error: 'Access denied to this tenant' };
    }
    
    // Update active scope
    setActiveScope(targetType, targetId);
    
    // Log switch
    await logAuditEvent({
      type: 'TENANT_SWITCH',
      userId,
      details: { from: context.activeScope, to: { type: targetType, id: targetId } },
    });
    
    return {
      success: true,
      newScope: { type: targetType, id: targetId },
    };
  }
  
  /**
   * Validate user has access to tenant
   */
  static async validateTenantAccess(
    userId: string,
    targetType: 'bureau' | 'employer',
    targetId: string
  ): Promise<boolean> {
    const context = getTenantContext();
    if (!context) return false;
    
    // Bureau users can access their own bureau
    if (targetType === 'bureau' && targetId === context.bureauId) {
      return true;
    }
    
    // Check role assignments for employer access
    if (targetType === 'employer') {
      const assignment = await db.query.userRoleAssignments.findFirst({
        where: and(
          eq(userRoleAssignments.userId, userId),
          eq(userRoleAssignments.scopeType, 'employer'),
          eq(userRoleAssignments.scopeId, targetId)
        ),
      });
      
      return !!assignment;
    }
    
    return false;
  }
}
```

### tRPC Tenant Router
```typescript
// src/server/trpc/routers/tenant.ts
import { z } from 'zod';
import { authenticatedProcedure, router } from '../trpc';
import { TenantSwitchService } from '@/lib/tenant/switch';

export const tenantRouter = router({
  // List available tenants
  list: authenticatedProcedure
    .query(async ({ ctx }) => {
      const tenants = await TenantSwitchService.getAvailableTenants(ctx.user!.sub);
      return tenants;
    }),
  
  // Switch tenant
  switch: authenticatedProcedure
    .input(z.object({
      type: z.enum(['bureau', 'employer']),
      id: z.string().uuid(),
    }))
    .mutation(async ({ input, ctx }) => {
      const result = await TenantSwitchService.switchToTenant(
        ctx.user!.sub,
        input.type,
        input.id
      );
      
      if (!result.success) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: result.error || 'Tenant switch failed',
        });
      }
      
      return result;
    }),
  
  // Validate access to tenant
  validate: authenticatedProcedure
    .input(z.object({
      type: z.enum(['bureau', 'employer']),
      id: z.string().uuid(),
    }))
    .query(async ({ input, ctx }) => {
      const valid = await TenantSwitchService.validateTenantAccess(
        ctx.user!.sub,
        input.type,
        input.id
      );
      
      return { valid };
    }),
});

export type TenantRouter = typeof tenantRouter;
```

## Business Rules & Invariants
1. Users can only switch to tenants they have role assignments for
2. Bureau users can switch between bureau and employer scopes
3. Client/employee users have fixed employer scope
4. Tenant switches are audited
5. Invalid switch attempts are logged

## Edge Cases
1. **Switch to invalid tenant** — Return access denied
2. **No available tenants** — Return empty list
3. **Same tenant switch** — Allow (idempotent)
4. **Tenant removed during session** — Next query fails with 403
5. **Concurrent switches** — Last one wins

## Tests

### src/lib/tenant/switch.test.ts
- `should list available tenants`: Verifies listing
- `should switch to valid tenant`: Verifies switch
- `should reject invalid tenant`: Verifies validation
- `should audit tenant switch`: Verifies logging
- `should handle bureau scope`: Verifies bureau switching

## Verification
```bash
npm run test:unit src/lib/tenant/switch.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § User Journeys → Journey 2
