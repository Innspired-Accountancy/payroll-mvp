# Slice d: Permission Checking Middleware

**Story:** story-05-rbac
**Epic:** epic-05-identity-access
**Effort:** L
**Dependencies:** slice-a

## Goal

Implement comprehensive permission checking middleware for tRPC procedures with caching, scoped permission evaluation, and clear error responses.

## Decision Checklist

- [x] All libraries/packages named: `ioredis@5.x`, `drizzle-orm@latest`
- [x] SDK methods identified: `redis.get()`, `redis.setex()`, middleware composition
- [x] External service endpoints: Redis for caching
- [x] Data contracts defined: Permission check result, cache structure
- [x] Configuration variables: `PERMISSION_CACHE_TTL=300` (5 minutes)
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ API Contracts → GET /api/v1/users/{id}/permissions
- 02-09-identity-access-spec.md:§ Integration Points → Permission checks

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/permissions/check.ts` | create | Permission checking service |
| `src/lib/permissions/cache.ts` | create | Permission caching |
| `src/server/trpc/middleware/permission.ts` | create | Permission middleware |

## Responsibilities
1. Check if user has specific permission (globally or scoped)
2. Cache user permissions in Redis for performance
3. Provide middleware for tRPC permission enforcement
4. Support checking multiple permissions (any/all)
5. Clear cache on role changes

## Contracts

### Permission Checking Service
```typescript
// src/lib/permissions/check.ts
import { eq, and } from 'drizzle-orm';
import { db } from '@/lib/db';
import { userRoleAssignments, roles } from '@/lib/db/schema';
import type { PermissionId } from './types';

export interface PermissionCheck {
  userId: string;
  permission: PermissionId;
  scopeType?: string;
  scopeId?: string;
}

export interface PermissionCheckResult {
  granted: boolean;
  assignments?: Array<{
    roleId: string;
    roleName: string;
    scopeType: string;
    scopeId: string;
  }>;
}

export class PermissionChecker {
  /**
   * Check if user has a permission (optionally scoped)
   */
  static async check(input: PermissionCheck): Promise<PermissionCheckResult> {
    // Get user's effective permissions
    const userPermissions = await this.getUserPermissions(input.userId);
    
    // Check for global permission
    const globalMatch = userPermissions.find(p => 
      p.permissionId === input.permission && !p.scopeType
    );
    
    if (globalMatch) {
      return { granted: true };
    }
    
    // Check for scoped permission
    if (input.scopeType && input.scopeId) {
      const scopedMatch = userPermissions.find(p =>
        p.permissionId === input.permission &&
        p.scopeType === input.scopeType &&
        p.scopeId === input.scopeId
      );
      
      if (scopedMatch) {
        return { granted: true };
      }
    }
    
    // Check for parent scope permission (bureau scope covers employer)
    if (input.scopeType === 'employer' && input.scopeId) {
      const bureauMatch = userPermissions.find(p =>
        p.permissionId === input.permission &&
        p.scopeType === 'bureau'
      );
      
      if (bureauMatch) {
        return { granted: true };
      }
    }
    
    return { granted: false };
  }
  
  /**
   * Check multiple permissions (returns true if ANY match)
   */
  static async checkAny(
    userId: string,
    permissions: PermissionId[],
    scopeType?: string,
    scopeId?: string
  ): Promise<boolean> {
    for (const permission of permissions) {
      const result = await this.check({ userId, permission, scopeType, scopeId });
      if (result.granted) return true;
    }
    return false;
  }
  
  /**
   * Check multiple permissions (returns true if ALL match)
   */
  static async checkAll(
    userId: string,
    permissions: PermissionId[],
    scopeType?: string,
    scopeId?: string
  ): Promise<boolean> {
    for (const permission of permissions) {
      const result = await this.check({ userId, permission, scopeType, scopeId });
      if (!result.granted) return false;
    }
    return true;
  }
  
  /**
   * Get all permissions for a user (with caching)
   */
  static async getUserPermissions(userId: string): Promise<Array<{
    permissionId: string;
    scopeType?: string;
    scopeId?: string;
  }>>> {
    // Try cache first
    const cached = await getCachedPermissions(userId);
    if (cached) return cached;
    
    // Query database
    const assignments = await db.query.userRoleAssignments.findMany({
      where: eq(userRoleAssignments.userId, userId),
      with: {
        role: true,
      },
    });
    
    const permissions: Array<{ permissionId: string; scopeType?: string; scopeId?: string }> = [];
    
    for (const assignment of assignments) {
      for (const permissionId of assignment.role.permissions) {
        permissions.push({
          permissionId,
          scopeType: assignment.scopeType,
          scopeId: assignment.scopeId,
        });
      }
    }
    
    // Cache result
    await cachePermissions(userId, permissions);
    
    return permissions;
  }
  
  /**
   * Get formatted permission list for API response
     */
  static async getUserPermissionsList(userId: string): Promise<Array<{
    permission: string;
    scope: { type: string; id: string } | null;
  }>> {
    const permissions = await this.getUserPermissions(userId);
    
    // Deduplicate and format
    const seen = new Set<string>();
    const result: Array<{ permission: string; scope: { type: string; id: string } | null }> = [];
    
    for (const p of permissions) {
      const key = `${p.permissionId}:${p.scopeType}:${p.scopeId}`;
      if (seen.has(key)) continue;
      seen.add(key);
      
      result.push({
        permission: p.permissionId,
        scope: p.scopeType && p.scopeId 
          ? { type: p.scopeType, id: p.scopeId }
          : null,
      });
    }
    
    return result;
  }
}
```

### Permission Cache
```typescript
// src/lib/permissions/cache.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);
const CACHE_PREFIX = 'permissions:';
const CACHE_TTL = 300; // 5 minutes

export async function getCachedPermissions(
  userId: string
): Promise<Array<{ permissionId: string; scopeType?: string; scopeId?: string }> | null> {
  const data = await redis.get(`${CACHE_PREFIX}${userId}`);
  if (!data) return null;
  return JSON.parse(data);
}

export async function cachePermissions(
  userId: string,
  permissions: Array<{ permissionId: string; scopeType?: string; scopeId?: string }>
): Promise<void> {
  await redis.setex(
    `${CACHE_PREFIX}${userId}`,
    CACHE_TTL,
    JSON.stringify(permissions)
  );
}

export async function invalidatePermissionCache(userId: string): Promise<void> {
  await redis.del(`${CACHE_PREFIX}${userId}`);
}

export async function invalidateAllPermissionCaches(): Promise<void> {
  const keys = await redis.keys(`${CACHE_PREFIX}*`);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}
```

### Permission Middleware
```typescript
// src/server/trpc/middleware/permission.ts
import { TRPCError } from '@trpc/server';
import { middleware } from '../trpc';
import { PermissionChecker } from '@/lib/permissions/check';
import type { PermissionId } from '@/lib/permissions/types';

interface RequirePermissionOptions {
  permission: PermissionId;
  scopeType?: 'bureau' | 'employer' | 'employee';
  getScopeId?: (ctx: any) => string | undefined;
}

export const requirePermission = (options: PermissionId | RequirePermissionOptions) => {
  const opts: RequirePermissionOptions = typeof options === 'string' 
    ? { permission: options }
    : options;
  
  return middleware(async ({ ctx, next, path }) => {
    if (!ctx.user) {
      throw new TRPCError({
        code: 'UNAUTHORIZED',
        message: 'Authentication required',
      });
    }
    
    const scopeId = opts.getScopeId ? opts.getScopeId(ctx) : undefined;
    
    const check = await PermissionChecker.check({
      userId: ctx.user.sub,
      permission: opts.permission,
      scopeType: opts.scopeType,
      scopeId,
    });
    
    if (!check.granted) {
      throw new TRPCError({
        code: 'FORBIDDEN',
        message: 'Permission denied',
      });
    }
    
    return next({ ctx });
  });
};

export const requireAnyPermission = (...permissions: PermissionId[]) => {
  return middleware(async ({ ctx, next }) => {
    if (!ctx.user) {
      throw new TRPCError({
        code: 'UNAUTHORIZED',
        message: 'Authentication required',
      });
    }
    
    const hasAny = await PermissionChecker.checkAny(ctx.user.sub, permissions);
    
    if (!hasAny) {
      throw new TRPCError({
        code: 'FORBIDDEN',
        message: 'Permission denied',
      });
    }
    
    return next({ ctx });
  });
};

export const requireAllPermissions = (...permissions: PermissionId[]) => {
  return middleware(async ({ ctx, next }) => {
    if (!ctx.user) {
      throw new TRPCError({
        code: 'UNAUTHORIZED',
        message: 'Authentication required',
      });
    }
    
    const hasAll = await PermissionChecker.checkAll(ctx.user.sub, permissions);
    
    if (!hasAll) {
      throw new TRPCError({
        code: 'FORBIDDEN',
        message: 'Permission denied',
      });
    }
    
    return next({ ctx });
  });
};
```

## Business Rules & Invariants
1. Permissions cached for 5 minutes to reduce database load
2. Cache invalidated on role assignment changes
3. Bureau scope permissions grant access to all employers within bureau
4. Permission checks return 403 without revealing which permission was missing
5. Scoped permission checks require both scope type and ID

## Edge Cases
1. **Cache miss** — Query database and populate cache
2. **Redis unavailable** — Fall back to database query
3. **User has no roles** — Return permission denied
4. **Multiple assignments with same permission** — Deduplicate in response
5. **Scope escalation attempt** — Validate scope matches user's tenant

## Tests

### src/lib/permissions/check.test.ts
- `should grant global permission`: Verifies global check
- `should grant scoped permission`: Verifies scoped check
- `should grant bureau scope for employer access`: Verifies parent scope
- `should deny without permission`: Verifies denial
- `should check any permission`: Verifies any check
- `should check all permissions`: Verifies all check
- `should use cache on subsequent checks`: Verifies caching

## Verification
```bash
npm run test:unit src/lib/permissions/check.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § API Contracts → Permissions endpoint
- 02-09-identity-access-spec.md § Integration Points → Permission checks
