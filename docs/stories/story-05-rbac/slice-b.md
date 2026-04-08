# Slice b: Role Management CRUD

**Story:** story-05-rbac
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-a

## Goal

Implement CRUD operations for custom bureau roles, including creation, update, deletion, and listing with permission validation.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`, `uuid@latest`
- [x] SDK methods identified: `db.insert()`, `db.update()`, `db.delete()`, `db.query()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Role create/update schemas, response types
- [x] Configuration variables: `MAX_CUSTOM_ROLES_PER_BUREAU=50`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Data Models → Role entity
- 02-09-identity-access-spec.md:§ User Journeys → Assign Role to User

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/roles/validation.ts` | create | Role input validation |
| `src/server/trpc/routers/roles.ts` | create | Role CRUD procedures |
| `src/lib/roles/queries.ts` | create | Role query helpers |

## Responsibilities
1. Create custom roles with permission sets
2. Update role names, descriptions, and permissions
3. Delete custom roles (with usage checks)
4. List roles with filtering and pagination
5. Prevent modification of system roles

## Contracts

### Role Validation Schemas
```typescript
// src/lib/roles/validation.ts
import { z } from 'zod';
import { isValidPermissionId } from '@/lib/permissions/registry';

export const createRoleSchema = z.object({
  name: z.string()
    .min(3, 'Role name must be at least 3 characters')
    .max(100, 'Role name must be less than 100 characters')
    .regex(/^[a-zA-Z0-9_\-\s]+$/, 'Role name contains invalid characters'),
  description: z.string()
    .min(10, 'Description must be at least 10 characters')
    .max(500, 'Description must be less than 500 characters'),
  permissions: z.array(z.string())
    .min(1, 'Role must have at least one permission')
    .max(100, 'Role cannot have more than 100 permissions')
    .refine(
      perms => perms.every(isValidPermissionId),
      'Invalid permission IDs in permissions array'
    ),
});

export const updateRoleSchema = createRoleSchema.partial().extend({
  id: z.string().uuid(),
});

export const roleFilterSchema = z.object({
  bureauId: z.string().uuid().optional(),
  isSystem: z.boolean().optional(),
  search: z.string().optional(),
  limit: z.number().int().min(1).max(100).default(20),
  offset: z.number().int().min(0).default(0),
});

export type CreateRoleInput = z.infer<typeof createRoleSchema>;
export type UpdateRoleInput = z.infer<typeof updateRoleSchema>;
export type RoleFilter = z.infer<typeof roleFilterSchema>;
```

### Role CRUD Service
```typescript
// src/lib/roles/service.ts
import { eq, and, like, sql } from 'drizzle-orm';
import { db } from '@/lib/db';
import { roles, users, userRoleAssignments } from '@/lib/db/schema';
import type { CreateRoleInput, UpdateRoleInput, RoleFilter } from './validation';
import type { Role } from '@/lib/db/schema';

const MAX_CUSTOM_ROLES_PER_BUREAU = 50;

export class RoleService {
  /**
   * Create a new custom role
   */
  static async create(
    bureauId: string,
    input: CreateRoleInput,
    createdBy: string
  ): Promise<Role> {
    // Check role limit for bureau
    const existingCount = await db.select({ count: sql<number>`count(*)` })
      .from(roles)
      .where(and(
        eq(roles.bureauId, bureauId),
        eq(roles.isSystem, false)
      ));
    
    if (existingCount[0].count >= MAX_CUSTOM_ROLES_PER_BUREAU) {
      throw new Error(`Maximum ${MAX_CUSTOM_ROLES_PER_BUREAU} custom roles allowed per bureau`);
    }
    
    // Check for duplicate name within bureau
    const existing = await db.query.roles.findFirst({
      where: and(
        eq(roles.bureauId, bureauId),
        sql`lower(${roles.name}) = lower(${input.name})`
      ),
    });
    
    if (existing) {
      throw new Error('A role with this name already exists in your bureau');
    }
    
    // Create role
    const [role] = await db.insert(roles).values({
      bureauId,
      name: input.name,
      description: input.description,
      permissions: input.permissions,
      isSystem: false,
    }).returning();
    
    // Log creation
    await logAuditEvent({
      type: 'ROLE_CREATED',
      actorId: createdBy,
      targetId: role.id,
      details: { name: input.name, permissions: input.permissions },
    });
    
    return role;
  }
  
  /**
   * Update a custom role
   */
  static async update(
    id: string,
    input: UpdateRoleInput,
    updatedBy: string
  ): Promise<Role> {
    // Get existing role
    const existing = await db.query.roles.findFirst({
      where: eq(roles.id, id),
    });
    
    if (!existing) {
      throw new Error('Role not found');
    }
    
    // Cannot modify system roles
    if (existing.isSystem) {
      throw new Error('System roles cannot be modified');
    }
    
    // Check name uniqueness if changing name
    if (input.name && input.name !== existing.name) {
      const duplicate = await db.query.roles.findFirst({
        where: and(
          eq(roles.bureauId, existing.bureauId),
          sql`lower(${roles.name}) = lower(${input.name})`,
          sql`${roles.id} != ${id}`
        ),
      });
      
      if (duplicate) {
        throw new Error('A role with this name already exists');
      }
    }
    
    // Update role
    const [updated] = await db.update(roles)
      .set({
        name: input.name ?? existing.name,
        description: input.description ?? existing.description,
        permissions: input.permissions ?? existing.permissions,
        updatedAt: new Date(),
      })
      .where(eq(roles.id, id))
      .returning();
    
    // Log update
    await logAuditEvent({
      type: 'ROLE_UPDATED',
      actorId: updatedBy,
      targetId: id,
      details: input,
    });
    
    return updated;
  }
  
  /**
   * Delete a custom role
   */
  static async delete(
    id: string,
    deletedBy: string
  ): Promise<void> {
    // Get existing role
    const existing = await db.query.roles.findFirst({
      where: eq(roles.id, id),
      with: {
        userAssignments: true,
      },
    });
    
    if (!existing) {
      throw new Error('Role not found');
    }
    
    // Cannot delete system roles
    if (existing.isSystem) {
      throw new Error('System roles cannot be deleted');
    }
    
    // Check if role is in use
    const assignmentCount = await db.select({ count: sql<number>`count(*)` })
      .from(userRoleAssignments)
      .where(eq(userRoleAssignments.roleId, id));
    
    if (assignmentCount[0].count > 0) {
      throw new Error(`Cannot delete role: assigned to ${assignmentCount[0].count} user(s)`);
    }
    
    // Delete role
    await db.delete(roles).where(eq(roles.id, id));
    
    // Log deletion
    await logAuditEvent({
      type: 'ROLE_DELETED',
      actorId: deletedBy,
      targetId: id,
      details: { name: existing.name },
    });
  }
  
  /**
   * List roles with filtering
   */
  static async list(filter: RoleFilter): Promise<{ roles: Role[]; total: number }> {
    const conditions = [];
    
    if (filter.bureauId !== undefined) {
      conditions.push(eq(roles.bureauId, filter.bureauId));
    }
    
    if (filter.isSystem !== undefined) {
      conditions.push(eq(roles.isSystem, filter.isSystem));
    }
    
    if (filter.search) {
      conditions.push(
        sql`(${roles.name} ilike ${`%${filter.search}%`} or ${roles.description} ilike ${`%${filter.search}%`})`
      );
    }
    
    const whereClause = conditions.length > 0 ? and(...conditions) : undefined;
    
    // Get total count
    const countResult = await db.select({ count: sql<number>`count(*)` })
      .from(roles)
      .where(whereClause);
    
    // Get paginated results
    const roleList = await db.query.roles.findMany({
      where: whereClause,
      limit: filter.limit,
      offset: filter.offset,
      orderBy: [roles.name],
    });
    
    return {
      roles: roleList,
      total: countResult[0].count,
    };
  }
  
  /**
   * Get a single role by ID
   */
  static async getById(id: string): Promise<Role | null> {
    return db.query.roles.findFirst({
      where: eq(roles.id, id),
    });
  }
  
  /**
   * Get role usage count
   */
  static async getUsageCount(roleId: string): Promise<number> {
    const result = await db.select({ count: sql<number>`count(*)` })
      .from(userRoleAssignments)
      .where(eq(userRoleAssignments.roleId, roleId));
    
    return result[0].count;
  }
}
```

### tRPC Roles Router
```typescript
// src/server/trpc/routers/roles.ts
import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { authenticatedProcedure, router } from '../trpc';
import { requirePermission } from '../middleware/auth';
import { RoleService } from '@/lib/roles/service';
import { createRoleSchema, updateRoleSchema, roleFilterSchema } from '@/lib/roles/validation';

export const rolesRouter = router({
  // List roles
  list: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .input(roleFilterSchema)
    .query(async ({ input, ctx }) => {
      // Bureau users can only see their bureau's roles + system roles
      const bureauId = ctx.user!.bureauId;
      
      const result = await RoleService.list({
        ...input,
        bureauId: bureauId || undefined,
      });
      
      return result;
    }),
  
  // Get single role
  get: authenticatedProcedure
    .input(z.object({ id: z.string().uuid() }))
    .query(async ({ input }) => {
      const role = await RoleService.getById(input.id);
      
      if (!role) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: 'Role not found',
        });
      }
      
      return role;
    }),
  
  // Create role
  create: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .input(createRoleSchema)
    .mutation(async ({ input, ctx }) => {
      const bureauId = ctx.user!.bureauId;
      
      if (!bureauId) {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Only bureau users can create custom roles',
        });
      }
      
      try {
        const role = await RoleService.create(
          bureauId,
          input,
          ctx.user!.sub
        );
        
        return role;
      } catch (error: any) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: error.message,
        });
      }
    }),
  
  // Update role
  update: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .input(updateRoleSchema)
    .mutation(async ({ input, ctx }) => {
      try {
        const role = await RoleService.update(
          input.id,
          input,
          ctx.user!.sub
        );
        
        return role;
      } catch (error: any) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: error.message,
        });
      }
    }),
  
  // Delete role
  delete: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .input(z.object({ id: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      try {
        await RoleService.delete(input.id, ctx.user!.sub);
        
        return { success: true };
      } catch (error: any) {
        throw new TRPCError({
          code: 'BAD_REQUEST',
          message: error.message,
        });
      }
    }),
  
  // Get role usage
  usage: authenticatedProcedure
    .input(z.object({ id: z.string().uuid() }))
    .query(async ({ input }) => {
      const count = await RoleService.getUsageCount(input.id);
      return { count };
    }),
});

export type RolesRouter = typeof rolesRouter;
```

## Business Rules & Invariants
1. Maximum 50 custom roles per bureau
2. Role names must be unique within a bureau (case-insensitive)
3. System roles cannot be modified or deleted
4. Roles with user assignments cannot be deleted
5. All permission IDs must be valid system permissions

## Edge Cases
1. **Duplicate role name** — Return validation error
2. **Delete role in use** — Return error with usage count
3. **Modify system role** — Return forbidden error
4. **Invalid permission IDs** — Zod validation rejects
5. **Role limit reached** — Return error with limit info

## Tests

### src/lib/roles/service.test.ts
- `should create custom role`: Verifies creation
- `should reject duplicate role name`: Verifies uniqueness
- `should reject system role modification`: Verifies protection
- `should reject delete if role in use`: Verifies usage check
- `should update role permissions`: Verifies update
- `should enforce role limit`: Verifies limit

## Verification
```bash
npm run test:unit src/lib/roles/service.test.ts
npm run test:unit src/server/trpc/routers/roles.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Data Models → Role entity
- 02-09-identity-access-spec.md § Standard Roles → System vs custom
