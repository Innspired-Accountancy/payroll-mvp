# Slice b: Role and Permission Entities

**Story:** story-01-user-model
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-a

## Goal

Create Role and Permission tables to support the RBAC system with system-defined and bureau-custom roles.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`
- [x] SDK methods identified: `pgTable()`, `uuid()`, `varchar()`, `jsonb()`, `boolean()`, `index()`, `unique()`
- [x] External service endpoints: N/A (database only)
- [x] Data contracts defined: Role and Permission schemas
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Data Models → Role entity
- 02-09-identity-access-spec.md:§ Data Models → Permission entity
- 02-09-identity-access-spec.md:§ Standard Roles → Bureau/Client/Employee roles

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/roles.ts` | create | Role table schema |
| `src/lib/db/schema/permissions.ts` | create | Permission table schema |
| `src/lib/db/types/role.ts` | create | TypeScript types |
| `src/lib/db/types/permission.ts` | create | TypeScript types |
| `src/lib/permissions/registry.ts` | create | Permission definitions registry |

## Responsibilities
1. Define Role table with JSONB permissions array
2. Define Permission table with static permission codes
3. Support system roles (is_system=true) and custom bureau roles
4. Create permission registry with all system permissions
5. Generate migrations

## Contracts

### Permission Table Schema
```typescript
export const permissions = pgTable('permissions', {
  id: varchar('id', { length: 100 }).primaryKey(), // e.g., 'payroll:view'
  name: varchar('name', { length: 100 }).notNull(),
  description: varchar('description', { length: 255 }).notNull(),
  resource: varchar('resource', { length: 50 }).notNull(), // 'payroll', 'employee', etc.
  action: varchar('action', { length: 20 }).notNull(), // 'view', 'create', 'edit', 'delete', 'approve'
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

export type Permission = typeof permissions.$inferSelect;
```

### Role Table Schema
```typescript
export const roles = pgTable('roles', {
  id: uuid('id').defaultRandom().primaryKey(),
  bureauId: uuid('bureau_id').references(() => bureaus.id), // null for system roles
  name: varchar('name', { length: 100 }).notNull(),
  description: varchar('description', { length: 255 }).notNull(),
  permissions: jsonb('permissions').notNull().$type<string[]>(), // Array of permission IDs
  isSystem: boolean('is_system').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  bureauIdx: index('roles_bureau_idx').on(table.bureauId),
  isSystemIdx: index('roles_is_system_idx').on(table.isSystem),
  uniqueBureauRoleName: unique('roles_bureau_name_unique').on(table.bureauId, table.name),
}));

export type Role = typeof roles.$inferSelect;
export type NewRole = typeof roles.$inferInsert;
```

### Permission Registry
```typescript
export const SYSTEM_PERMISSIONS = {
  // Payroll permissions
  PAYROLL_VIEW: { id: 'payroll:view', name: 'View Payroll', resource: 'payroll', action: 'view' },
  PAYROLL_CREATE: { id: 'payroll:create', name: 'Create Payroll', resource: 'payroll', action: 'create' },
  PAYROLL_EDIT: { id: 'payroll:edit', name: 'Edit Payroll', resource: 'payroll', action: 'edit' },
  PAYROLL_APPROVE: { id: 'payroll:approve', name: 'Approve Payroll', resource: 'payroll', action: 'approve' },
  PAYROLL_DELETE: { id: 'payroll:delete', name: 'Delete Payroll', resource: 'payroll', action: 'delete' },
  
  // Employee permissions
  EMPLOYEE_VIEW: { id: 'employee:view', name: 'View Employees', resource: 'employee', action: 'view' },
  EMPLOYEE_CREATE: { id: 'employee:create', name: 'Create Employees', resource: 'employee', action: 'create' },
  EMPLOYEE_EDIT: { id: 'employee:edit', name: 'Edit Employees', resource: 'employee', action: 'edit' },
  EMPLOYEE_DELETE: { id: 'employee:delete', name: 'Delete Employees', resource: 'employee', action: 'delete' },
  
  // Payment permissions
  PAYMENTS_APPROVE: { id: 'payments:approve', name: 'Approve Payments', resource: 'payments', action: 'approve' },
  
  // CIS permissions
  CIS_VIEW: { id: 'cis:view', name: 'View CIS', resource: 'cis', action: 'view' },
  CIS_MANAGE: { id: 'cis:manage', name: 'Manage CIS', resource: 'cis', action: 'manage' },
  
  // Report permissions
  REPORTS_VIEW: { id: 'reports:view', name: 'View Reports', resource: 'reports', action: 'view' },
  
  // Admin permissions
  USER_MANAGE: { id: 'user:manage', name: 'Manage Users', resource: 'user', action: 'manage' },
  ROLE_MANAGE: { id: 'role:manage', name: 'Manage Roles', resource: 'role', action: 'manage' },
} as const;
```

## Business Rules & Invariants
1. System roles (is_system=true) have bureau_id=null and cannot be modified
2. Custom roles must have bureau_id set and unique name within bureau
3. Permissions array in Role contains permission IDs from permissions table
4. Permission IDs follow pattern `{resource}:{action}` for consistency

## Edge Cases
1. **Duplicate role name within bureau** — Return 409 Conflict
2. **Invalid permission ID in array** — Validate against registry, reject if unknown
3. **Modifying system role** — Return 403 Forbidden
4. **Deleting role with assignments** — Either cascade or prevent, documented in slice-c

## Tests

### src/lib/db/schema/roles.test.ts
- `should create system role with null bureau_id`: Verifies system role creation
- `should create custom bureau role`: Verifies bureau-specific role
- `should enforce unique role name per bureau`: Verifies unique constraint
- `should prevent system role modification`: Verifies is_system protection

### src/lib/permissions/registry.test.ts
- `should have all expected permissions defined`: Verifies permission completeness
- `should have valid permission ID format`: Verifies id format {resource}:{action}

## Verification
```bash
npm run db:generate
npm run db:migrate
npm run test:unit src/lib/db/schema/roles.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Data Models → Role and Permission entities
- 02-09-identity-access-spec.md § Standard Roles → Role definitions
