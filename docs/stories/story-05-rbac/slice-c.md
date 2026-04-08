# Slice c: User-Role Assignment

**Story:** story-05-rbac
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-b

## Goal

Implement user-role assignment with multi-tenant scoping, allowing users to have different roles for different employers within their bureau access.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`
- [x] SDK methods identified: `db.insert()`, `db.delete()`, `db.query()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Assignment create/delete schemas
- [x] Configuration variables: `MAX_ROLES_PER_USER=10`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Data Models → UserRoleAssignment entity
- 02-09-identity-access-spec.md:§ User Journeys → Assign Role to User

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/roles/assignment.ts` | create | Role assignment service |
| `src/server/trpc/routers/user-roles.ts` | create | Assignment CRUD procedures |

## Responsibilities
1. Assign roles to users with specific scope (bureau/employer/employee)
2. Validate user can receive role (user type compatibility)
3. Prevent duplicate assignments
4. Support bulk assignment operations
5. Track who made each assignment

## Contracts

### Assignment Service
```typescript
// src/lib/roles/assignment.ts
import { eq, and } from 'drizzle-orm';
import { db } from '@/lib/db';
import { userRoleAssignments, users, roles } from '@/lib/db/schema';
import type { ScopeType } from '@/lib/db/schema';

const MAX_ROLES_PER_USER = 10;

export interface AssignRoleInput {
  userId: string;
  roleId: string;
  scopeType: ScopeType;
  scopeId: string;
}

export interface AssignmentValidation {
  valid: boolean;
  error?: string;
}

export class RoleAssignmentService {
  /**
   * Assign a role to a user
   */
  static async assign(
    input: AssignRoleInput,
    assignedBy: string
  ): Promise<typeof userRoleAssignments.$inferSelect> {
    // Validate assignment
    const validation = await this.validateAssignment(input);
    if (!validation.valid) {
      throw new Error(validation.error);
    }
    
    // Check for duplicate
    const existing = await db.query.userRoleAssignments.findFirst({
      where: and(
        eq(userRoleAssignments.userId, input.userId),
        eq(userRoleAssignments.roleId, input.roleId),
        eq(userRoleAssignments.scopeType, input.scopeType),
        eq(userRoleAssignments.scopeId, input.scopeId)
      ),
    });
    
    if (existing) {
      throw new Error('User already has this role for the specified scope');
    }
    
    // Check role limit
    const currentRoles = await db.query.userRoleAssignments.findMany({
      where: eq(userRoleAssignments.userId, input.userId),
    });
    
    if (currentRoles.length >= MAX_ROLES_PER_USER) {
      throw new Error(`User cannot have more than ${MAX_ROLES_PER_USER} role assignments`);
    }
    
    // Create assignment
    const [assignment] = await db.insert(userRoleAssignments).values({
      userId: input.userId,
      roleId: input.roleId,
      scopeType: input.scopeType,
      scopeId: input.scopeId,
      assignedBy,
      assignedAt: new Date(),
    }).returning();
    
    // Log assignment
    await logAuditEvent({
      type: 'ROLE_ASSIGNED',
      actorId: assignedBy,
      targetId: input.userId,
      details: {
        roleId: input.roleId,
        scopeType: input.scopeType,
        scopeId: input.scopeId,
      },
    });
    
    // Invalidate permission cache for user
    await invalidatePermissionCache(input.userId);
    
    return assignment;
  }
  
  /**
   * Remove a role assignment
   */
  static async remove(
    assignmentId: string,
    removedBy: string
  ): Promise<void> {
    const assignment = await db.query.userRoleAssignments.findFirst({
      where: eq(userRoleAssignments.id, assignmentId),
    });
    
    if (!assignment) {
      throw new Error('Assignment not found');
    }
    
    await db.delete(userRoleAssignments)
      .where(eq(userRoleAssignments.id, assignmentId));
    
    // Log removal
    await logAuditEvent({
      type: 'ROLE_REMOVED',
      actorId: removedBy,
      targetId: assignment.userId,
      details: {
        roleId: assignment.roleId,
        scopeType: assignment.scopeType,
        scopeId: assignment.scopeId,
      },
    });
    
    // Invalidate permission cache
    await invalidatePermissionCache(assignment.userId);
  }
  
  /**
   * Get all roles for a user
   */
  static async getUserRoles(userId: string) {
    return db.query.userRoleAssignments.findMany({
      where: eq(userRoleAssignments.userId, userId),
      with: {
        role: true,
      },
    });
  }
  
  /**
   * Validate that an assignment is allowed
   */
  static async validateAssignment(input: AssignRoleInput): Promise<AssignmentValidation> {
    // Get user
    const user = await db.query.users.findFirst({
      where: eq(users.id, input.userId),
      columns: { userType: true, bureauId: true, employerId: true },
    });
    
    if (!user) {
      return { valid: false, error: 'User not found' };
    }
    
    // Get role
    const role = await db.query.roles.findFirst({
      where: eq(roles.id, input.roleId),
    });
    
    if (!role) {
      return { valid: false, error: 'Role not found' };
    }
    
    // Validate scope type matches user type
    const validScopes: Record<string, ScopeType[]> = {
      bureau: ['bureau', 'employer'],
      client: ['employer'],
      employee: ['employee'],
    };
    
    if (!validScopes[user.userType]?.includes(input.scopeType)) {
      return { 
        valid: false, 
        error: `Users of type '${user.userType}' cannot have '${input.scopeType}' scope assignments` 
      };
    }
    
    // Validate scope ID matches user's tenant
    if (input.scopeType === 'bureau' && input.scopeId !== user.bureauId) {
      return { valid: false, error: 'Bureau scope ID does not match user\'s bureau' };
    }
    
    if (input.scopeType === 'employer' && input.scopeId !== user.employerId && user.userType !== 'bureau') {
      return { valid: false, error: 'Employer scope not valid for this user' };
    }
    
    if (input.scopeType === 'employee' && input.scopeId !== user.employerId) {
      return { valid: false, error: 'Employee scope not valid for this user' };
    }
    
    // Validate custom role belongs to user's bureau
    if (!role.isSystem && role.bureauId !== user.bureauId) {
      return { valid: false, error: 'Role does not belong to user\'s bureau' };
    }
    
    return { valid: true };
  }
}

async function invalidatePermissionCache(userId: string): Promise<void> {
  // Implemented in slice-d or using Redis
  const redis = new Redis(process.env.REDIS_URL!);
  await redis.del(`permissions:${userId}`);
}
```

### tRPC User Roles Router
```typescript
// src/server/trpc/routers/user-roles.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { authenticatedProcedure, router } from '../trpc';
import { requirePermission } from '../middleware/auth';
import { RoleAssignmentService } from '@/lib/roles/assignment';
import { scopeTypeSchema } from '@/lib/db/schema';

export const userRolesRouter = router({
  // Get user's roles
  list: authenticatedProcedure
    .input(z.object({ userId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Users can view their own roles, or need user:manage permission
      if (input.userId !== ctx.user!.sub) {
        requirePermission('user:manage');
      }
      
      const assignments = await RoleAssignmentService.getUserRoles(input.userId);
      return assignments;
    }),
  
  // Assign role to user
  assign: authenticatedProcedure
    .use(requirePermission('user:manage'))
    .input(z.object({
      userId: z.string().uuid(),
      roleId: z.string().uuid(),
      scopeType: scopeTypeSchema,
      scopeId: z.string().uuid(),
    }))
    .mutation(async ({ input, ctx }) => {
      try {
        const assignment = await RoleAssignmentService.assign(
          input,
          ctx.user!.sub
        );
        
        return assignment;
      } catch (error: any) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: error.message,
        });
      }
    }),
  
  // Remove role assignment
  remove: authenticatedProcedure
    .use(requirePermission('user:manage'))
    .input(z.object({
      assignmentId: z.string().uuid(),
    }))
    .mutation(async ({ input, ctx }) => {
      try {
        await RoleAssignmentService.remove(input.assignmentId, ctx.user!.sub);
        
        return { success: true };
      } catch (error: any) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: error.message,
        });
      }
    }),
  
  // Validate potential assignment (for UI)
  validate: authenticatedProcedure
    .use(requirePermission('user:manage'))
    .input(z.object({
      userId: z.string().uuid(),
      roleId: z.string().uuid(),
      scopeType: scopeTypeSchema,
      scopeId: z.string().uuid(),
    }))
    .query(async ({ input }) => {
      const result = await RoleAssignmentService.validateAssignment(input);
      return result;
    }),
});

export type UserRolesRouter = typeof userRolesRouter;
```

## Business Rules & Invariants
1. Maximum 10 role assignments per user
2. No duplicate role assignments for same scope
3. Scope type must match user's user_type
4. Custom roles only assignable within same bureau
5. System roles can be assigned across bureaus

## Edge Cases
1. **Duplicate assignment attempt** — Return error with existing info
2. **Role limit reached** — Return error with current count
3. **Scope mismatch** — Return validation error
4. **Remove last role** — Allow (user may have no roles)
5. **Assignment to inactive user** — Warn but allow

## Tests

### src/lib/roles/assignment.test.ts
- `should assign role to user`: Verifies assignment
- `should reject duplicate assignment`: Verifies uniqueness
- `should enforce role limit`: Verifies limit
- `should validate scope compatibility`: Verifies scope rules
- `should invalidate cache on change`: Verifies cache invalidation

## Verification
```bash
npm run test:unit src/lib/roles/assignment.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Data Models → UserRoleAssignment
