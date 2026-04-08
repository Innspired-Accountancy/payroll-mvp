# Slice a: Permission Registry and Definitions

**Story:** story-05-rbac
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** story-01-user-model

## Goal

Implement the complete permission registry with all system permissions defined, typed, and queryable. Establish the foundation for role-based access control.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`
- [x] SDK methods identified: Object.freeze(), TypeScript const assertions
- [x] External service endpoints: N/A
- [x] Data contracts defined: Permission type, registry structure, action types
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Data Models → Permission entity
- 02-09-identity-access-spec.md:§ Standard Roles → All permission references

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/permissions/registry.ts` | create | Permission definitions |
| `src/lib/permissions/types.ts` | create | Permission TypeScript types |
| `src/lib/permissions/resources.ts` | create | Resource definitions |
| `src/lib/permissions/actions.ts` | create | Action type definitions |
| `src/lib/db/seeds/permissions.ts` | create | Permission seed data |

## Responsibilities
1. Define all system permissions with IDs, names, descriptions
2. Organize permissions by resource and action
3. Create TypeScript types for compile-time checking
4. Provide permission lookup utilities
5. Seed permissions to database

## Contracts

### Permission Types
```typescript
// src/lib/permissions/types.ts
export type PermissionAction = 'view' | 'create' | 'edit' | 'delete' | 'approve' | 'manage';

export type PermissionResource = 
  | 'payroll' 
  | 'employee' 
  | 'payments' 
  | 'cis' 
  | 'reports' 
  | 'user' 
  | 'role'
  | 'employer'
  | 'bureau'
  | 'payslip'
  | 'leave'
  | 'profile';

export interface Permission {
  id: string;              // e.g., 'payroll:view'
  name: string;            // Human-readable name
  description: string;
  resource: PermissionResource;
  action: PermissionAction;
}

export type PermissionId = 
  | 'payroll:view' | 'payroll:create' | 'payroll:edit' | 'payroll:delete' | 'payroll:approve'
  | 'employee:view' | 'employee:create' | 'employee:edit' | 'employee:delete'
  | 'payments:approve'
  | 'cis:view' | 'cis:manage'
  | 'reports:view'
  | 'user:manage'
  | 'role:manage'
  | 'employer:view' | 'employer:manage'
  | 'bureau:manage'
  | 'payslip:view_own'
  | 'leave:request'
  | 'profile:edit_own';
```

### Permission Registry
```typescript
// src/lib/permissions/registry.ts
import type { Permission, PermissionId } from './types';

export const PERMISSIONS = {
  // Payroll permissions
  PAYROLL_VIEW: {
    id: 'payroll:view',
    name: 'View Payroll',
    description: 'View payroll runs, calculations, and summaries',
    resource: 'payroll',
    action: 'view',
  },
  PAYROLL_CREATE: {
    id: 'payroll:create',
    name: 'Create Payroll',
    description: 'Create new payroll runs',
    resource: 'payroll',
    action: 'create',
  },
  PAYROLL_EDIT: {
    id: 'payroll:edit',
    name: 'Edit Payroll',
    description: 'Edit draft payroll runs',
    resource: 'payroll',
    action: 'edit',
  },
  PAYROLL_DELETE: {
    id: 'payroll:delete',
    name: 'Delete Payroll',
    description: 'Delete payroll runs',
    resource: 'payroll',
    action: 'delete',
  },
  PAYROLL_APPROVE: {
    id: 'payroll:approve',
    name: 'Approve Payroll',
    description: 'Approve payroll runs for submission',
    resource: 'payroll',
    action: 'approve',
  },
  
  // Employee permissions
  EMPLOYEE_VIEW: {
    id: 'employee:view',
    name: 'View Employees',
    description: 'View employee records',
    resource: 'employee',
    action: 'view',
  },
  EMPLOYEE_CREATE: {
    id: 'employee:create',
    name: 'Create Employees',
    description: 'Create new employee records',
    resource: 'employee',
    action: 'create',
  },
  EMPLOYEE_EDIT: {
    id: 'employee:edit',
    name: 'Edit Employees',
    description: 'Edit employee records',
    resource: 'employee',
    action: 'edit',
  },
  EMPLOYEE_DELETE: {
    id: 'employee:delete',
    name: 'Delete Employees',
    description: 'Delete employee records',
    resource: 'employee',
    action: 'delete',
  },
  
  // Payment permissions
  PAYMENTS_APPROVE: {
    id: 'payments:approve',
    name: 'Approve Payments',
    description: 'Approve payment batches',
    resource: 'payments',
    action: 'approve',
  },
  
  // CIS permissions
  CIS_VIEW: {
    id: 'cis:view',
    name: 'View CIS',
    description: 'View CIS submissions and verifications',
    resource: 'cis',
    action: 'view',
  },
  CIS_MANAGE: {
    id: 'cis:manage',
    name: 'Manage CIS',
    description: 'Create and submit CIS verifications',
    resource: 'cis',
    action: 'manage',
  },
  
  // Report permissions
  REPORTS_VIEW: {
    id: 'reports:view',
    name: 'View Reports',
    description: 'View and generate reports',
    resource: 'reports',
    action: 'view',
  },
  
  // User management permissions
  USER_MANAGE: {
    id: 'user:manage',
    name: 'Manage Users',
    description: 'Create, edit, and manage users',
    resource: 'user',
    action: 'manage',
  },
  
  // Role management permissions
  ROLE_MANAGE: {
    id: 'role:manage',
    name: 'Manage Roles',
    description: 'Create and manage roles and permissions',
    resource: 'role',
    action: 'manage',
  },
  
  // Employer permissions
  EMPLOYER_VIEW: {
    id: 'employer:view',
    name: 'View Employers',
    description: 'View employer/client data',
    resource: 'employer',
    action: 'view',
  },
  EMPLOYER_MANAGE: {
    id: 'employer:manage',
    name: 'Manage Employers',
    description: 'Create and manage employers',
    resource: 'employer',
    action: 'manage',
  },
  
  // Bureau permissions
  BUREAU_MANAGE: {
    id: 'bureau:manage',
    name: 'Manage Bureau',
    description: 'Full bureau administration',
    resource: 'bureau',
    action: 'manage',
  },
  
  // Employee self-service permissions
  PAYSLIP_VIEW_OWN: {
    id: 'payslip:view_own',
    name: 'View Own Payslips',
    description: 'View own payslip history',
    resource: 'payslip',
    action: 'view',
  },
  LEAVE_REQUEST: {
    id: 'leave:request',
    name: 'Request Leave',
    description: 'Submit leave requests',
    resource: 'leave',
    action: 'create',
  },
  PROFILE_EDIT_OWN: {
    id: 'profile:edit_own',
    name: 'Edit Own Profile',
    description: 'Edit own personal details',
    resource: 'profile',
    action: 'edit',
  },
} as const satisfies Record<string, Permission>;

// Type for permission keys
export type PermissionKey = keyof typeof PERMISSIONS;

// Array of all permissions for iteration
export const ALL_PERMISSIONS = Object.values(PERMISSIONS) as Permission[];

// Lookup by ID
export function getPermissionById(id: PermissionId): Permission | undefined {
  return ALL_PERMISSIONS.find(p => p.id === id);
}

// Get permissions by resource
export function getPermissionsByResource(resource: string): Permission[] {
  return ALL_PERMISSIONS.filter(p => p.resource === resource);
}

// Validate permission ID
export function isValidPermissionId(id: string): id is PermissionId {
  return ALL_PERMISSIONS.some(p => p.id === id);
}

// Get permission IDs for a role (helper for role definitions)
export function getPermissionIds(keys: PermissionKey[]): PermissionId[] {
  return keys.map(key => PERMISSIONS[key].id);
}
```

### Permission Seeds
```typescript
// src/lib/db/seeds/permissions.ts
import { db } from '@/lib/db';
import { permissions } from '@/lib/db/schema';
import { ALL_PERMISSIONS } from '@/lib/permissions/registry';

export async function seedPermissions(): Promise<void> {
  const existing = await db.select().from(permissions);
  
  if (existing.length > 0) {
    console.log('Permissions already seeded');
    return;
  }
  
  await db.insert(permissions).values(
    ALL_PERMISSIONS.map(p => ({
      id: p.id,
      name: p.name,
      description: p.description,
      resource: p.resource,
      action: p.action,
    }))
  );
  
  console.log(`Seeded ${ALL_PERMISSIONS.length} permissions`);
}
```

## Business Rules & Invariants
1. Permission IDs follow `{resource}:{action}` format
2. All permissions are immutable (defined in code)
3. New permissions require code changes and migration
4. Permission registry is the source of truth

## Edge Cases
1. **Invalid permission ID** — Return undefined from lookup
2. **Duplicate permission ID** — TypeScript prevents at compile time
3. **Permission referenced but not in registry** — Runtime validation catches
4. **Database out of sync with registry** — Seed script syncs on deploy

## Tests

### src/lib/permissions/registry.test.ts
- `should have all expected permissions`: Verifies completeness
- `should have valid permission ID format`: Verifies format
- `should lookup permission by ID`: Verifies lookup
- `should filter by resource`: Verifies resource query
- `should validate permission IDs`: Verifies validation

## Verification
```bash
npm run test:unit src/lib/permissions/registry.test.ts
npm run db:seed  # Seeds permissions
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Data Models → Permission entity
- 02-09-identity-access-spec.md § Standard Roles → Permission references
