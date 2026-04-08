# Slice d: Seed Data and Indexes

**Story:** story-01-user-model
**Epic:** epic-05-identity-access
**Effort:** XS
**Dependencies:** slice-c

## Goal

Create seed data for system roles and verify all database indexes are in place for optimal query performance.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `drizzle-seed` (or custom seed script)
- [x] SDK methods identified: `db.insert()`, `db.select()`, SQL `CREATE INDEX`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Seed data structures
- [x] Configuration variables: `NODE_ENV` for conditional seeding
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Standard Roles → All system role definitions

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/seeds/system-roles.ts` | create | System roles seed data |
| `src/lib/db/seeds/system-permissions.ts` | create | Permissions seed data |
| `src/lib/db/seeds/index.ts` | create | Seed runner |
| `scripts/seed.ts` | create | CLI seed script |

## Responsibilities
1. Create seed data for all system permissions
2. Create seed data for all system roles with their permission sets
3. Verify all indexes are defined in migrations
4. Document index usage patterns for common queries

## Contracts

### System Roles Seed Data
```typescript
export const SYSTEM_ROLES_SEED = [
  {
    name: 'Platform Admin',
    description: 'Full system access and administration',
    permissions: Object.values(SYSTEM_PERMISSIONS).map(p => p.id),
    isSystem: true,
  },
  {
    name: 'Bureau Owner',
    description: 'Full access within their bureau',
    permissions: [
      SYSTEM_PERMISSIONS.PAYROLL_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_CREATE.id,
      SYSTEM_PERMISSIONS.PAYROLL_EDIT.id,
      SYSTEM_PERMISSIONS.PAYROLL_APPROVE.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_VIEW.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_CREATE.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_EDIT.id,
      SYSTEM_PERMISSIONS.PAYMENTS_APPROVE.id,
      SYSTEM_PERMISSIONS.CIS_VIEW.id,
      SYSTEM_PERMISSIONS.CIS_MANAGE.id,
      SYSTEM_PERMISSIONS.REPORTS_VIEW.id,
      SYSTEM_PERMISSIONS.USER_MANAGE.id,
      SYSTEM_PERMISSIONS.ROLE_MANAGE.id,
    ],
    isSystem: true,
  },
  {
    name: 'Payroll Manager',
    description: 'Manage payroll and employees for assigned employers',
    permissions: [
      SYSTEM_PERMISSIONS.PAYROLL_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_CREATE.id,
      SYSTEM_PERMISSIONS.PAYROLL_EDIT.id,
      SYSTEM_PERMISSIONS.PAYROLL_APPROVE.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_VIEW.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_CREATE.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_EDIT.id,
      SYSTEM_PERMISSIONS.REPORTS_VIEW.id,
    ],
    isSystem: true,
  },
  {
    name: 'Payroll Processor',
    description: 'Process payroll for assigned employers',
    permissions: [
      SYSTEM_PERMISSIONS.PAYROLL_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_CREATE.id,
      SYSTEM_PERMISSIONS.PAYROLL_EDIT.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_VIEW.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_CREATE.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_EDIT.id,
    ],
    isSystem: true,
  },
  {
    name: 'Reviewer',
    description: 'Review and view payroll',
    permissions: [
      SYSTEM_PERMISSIONS.PAYROLL_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_APPROVE.id,
    ],
    isSystem: true,
  },
  {
    name: 'Approver',
    description: 'Approve payroll and payments',
    permissions: [
      SYSTEM_PERMISSIONS.PAYROLL_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_APPROVE.id,
      SYSTEM_PERMISSIONS.PAYMENTS_APPROVE.id,
    ],
    isSystem: true,
  },
  {
    name: 'CIS Operator',
    description: 'Manage CIS submissions',
    permissions: [
      SYSTEM_PERMISSIONS.CIS_VIEW.id,
      SYSTEM_PERMISSIONS.CIS_MANAGE.id,
    ],
    isSystem: true,
  },
  // Client Portal Roles
  {
    name: 'Client Admin',
    description: 'Admin access to their employer data',
    permissions: [
      SYSTEM_PERMISSIONS.EMPLOYEE_VIEW.id,
      SYSTEM_PERMISSIONS.EMPLOYEE_CREATE.id,
      SYSTEM_PERMISSIONS.PAYROLL_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_APPROVE.id,
    ],
    isSystem: true,
  },
  {
    name: 'Client Manager',
    description: 'View and approve payroll data',
    permissions: [
      SYSTEM_PERMISSIONS.EMPLOYEE_VIEW.id,
      SYSTEM_PERMISSIONS.REPORTS_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_VIEW.id,
      SYSTEM_PERMISSIONS.PAYROLL_APPROVE.id,
    ],
    isSystem: true,
  },
  {
    name: 'Client Viewer',
    description: 'View-only access to reports and employees',
    permissions: [
      SYSTEM_PERMISSIONS.EMPLOYEE_VIEW.id,
      SYSTEM_PERMISSIONS.REPORTS_VIEW.id,
    ],
    isSystem: true,
  },
];
```

### Seed Function
```typescript
export async function seedSystemData(db: PostgresJsDatabase) {
  // Seed permissions first
  const existingPermissions = await db.select().from(permissions);
  if (existingPermissions.length === 0) {
    await db.insert(permissions).values(Object.values(SYSTEM_PERMISSIONS));
  }
  
  // Seed system roles
  const existingRoles = await db.select().from(roles).where(eq(roles.isSystem, true));
  if (existingRoles.length === 0) {
    await db.insert(roles).values(SYSTEM_ROLES_SEED);
  }
}
```

### Index Documentation
```typescript
// Users table indexes for common queries:
// - emailIdx: Login by email lookup
// - bureauIdx: List all users in a bureau
// - employerIdx: List all users for an employer
// - statusIdx: Filter by active/pending status

// Roles table indexes:
// - bureauIdx: List custom roles for a bureau
// - isSystemIdx: Quick filter for system roles

// UserRoleAssignments indexes:
// - userIdx: Get all roles for a user
// - roleIdx: Get all users with a role
// - scopeIdx: Filter by tenant scope
```

## Business Rules & Invariants
1. System roles cannot be deleted or modified by bureau admins
2. Seed data is idempotent (checks for existing data before insert)
3. Permissions must be seeded before roles (foreign key constraint)

## Edge Cases
1. **Running seed on production** — Script should check NODE_ENV and require explicit flag
2. **Partial seed (some data exists)** — Skip existing, don't fail
3. **Permission ID changes** — Seed uses current registry, old IDs may become orphaned

## Tests

### src/lib/db/seeds/seeds.test.ts
- `should seed all system permissions`: Verifies permission count
- `should seed all system roles`: Verifies role count
- `should be idempotent (rerunnable)`: Verifies no duplicates on re-run
- `should have correct permission assignments`: Verifies role-permission mappings

## Verification
```bash
npm run db:seed
npm run test:unit src/lib/db/seeds/seeds.test.ts
npm run db:studio  # Verify data visually
```

## Source Sections
- 02-09-identity-access-spec.md § Standard Roles → Bureau and Client roles
