# Slice c: UserRoleAssignment with Scoping

**Story:** story-01-user-model
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-b

## Goal

Create the UserRoleAssignment junction table that enables many-to-many relationships between users and roles with multi-tenant scoping support.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`
- [x] SDK methods identified: `pgTable()`, `uuid()`, `varchar()`, `timestamp()`, `foreignKey()`, `index()`
- [x] External service endpoints: N/A (database only)
- [x] Data contracts defined: UserRoleAssignment schema with scope types
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Data Models → UserRoleAssignment entity
- 02-09-identity-access-spec.md:§ User Journeys → Role assignment with scope

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/user-role-assignments.ts` | create | Junction table schema |
| `src/lib/db/types/user-role-assignment.ts` | create | TypeScript types |
| `src/lib/db/relations.ts` | create/update | Drizzle relations definition |

## Responsibilities
1. Define UserRoleAssignment with scope_type and scope_id for tenant scoping
2. Create foreign key relationships to users and roles tables
3. Support scope types: 'bureau', 'employer', 'employee'
4. Track assignment metadata (assigned_by, assigned_at)
5. Enforce unique user+role+scope combinations

## Contracts

### UserRoleAssignment Table Schema
```typescript
export const userRoleAssignments = pgTable('user_role_assignments', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  roleId: uuid('role_id').notNull().references(() => roles.id, { onDelete: 'cascade' }),
  scopeType: varchar('scope_type', { length: 20 }).notNull(), // 'bureau' | 'employer' | 'employee'
  scopeId: uuid('scope_id').notNull(),
  assignedBy: uuid('assigned_by').notNull().references(() => users.id),
  assignedAt: timestamp('assigned_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  userIdx: index('ura_user_idx').on(table.userId),
  roleIdx: index('ura_role_idx').on(table.roleId),
  scopeIdx: index('ura_scope_idx').on(table.scopeType, table.scopeId),
  uniqueUserRoleScope: unique('ura_user_role_scope_unique').on(table.userId, table.roleId, table.scopeType, table.scopeId),
}));

export type UserRoleAssignment = typeof userRoleAssignments.$inferSelect;
export type NewUserRoleAssignment = typeof userRoleAssignments.$inferInsert;
export type ScopeType = 'bureau' | 'employer' | 'employee';
```

### Relations Definition
```typescript
export const usersRelations = relations(users, ({ many }) => ({
  roleAssignments: many(userRoleAssignments),
}));

export const rolesRelations = relations(roles, ({ many }) => ({
  userAssignments: many(userRoleAssignments),
}));

export const userRoleAssignmentsRelations = relations(userRoleAssignments, ({ one }) => ({
  user: one(users, { fields: [userRoleAssignments.userId], references: [users.id] }),
  role: one(roles, { fields: [userRoleAssignments.roleId], references: [roles.id] }),
  assigner: one(users, { fields: [userRoleAssignments.assignedBy], references: [users.id] }),
}));
```

### Scope Validation Logic
```typescript
export function validateScopeType(userType: UserType, scopeType: ScopeType): boolean {
  const validScopes: Record<UserType, ScopeType[]> = {
    bureau: ['bureau', 'employer'],
    client: ['employer'],
    employee: ['employee'],
  };
  return validScopes[userType]?.includes(scopeType) ?? false;
}

export function getScopeIdForUser(user: User, scopeType: ScopeType): string | null {
  switch (scopeType) {
    case 'bureau': return user.bureauId;
    case 'employer': return user.employerId;
    case 'employee': return user.employeeId;
    default: return null;
  }
}
```

## Business Rules & Invariants
1. Scope type must be compatible with user's user_type (bureau users can have bureau or employer scope)
2. scope_id must reference the appropriate entity based on scope_type
3. A user cannot have duplicate role assignments for the same scope
4. When a user is deleted, all their role assignments are cascaded (onDelete: cascade)
5. When a role is deleted, all assignments to that role are cascaded

## Edge Cases
1. **Duplicate assignment** — Return 409 Conflict with unique constraint violation
2. **Invalid scope type for user** — Return 400 Bad Request with validation error
3. **scope_id doesn't match user's tenant** — Return 400 (e.g., employer scope for bureau user with different employer)
4. **Assigning to self** — Allowed for admin, but logged

## Tests

### src/lib/db/schema/user-role-assignments.test.ts
- `should create role assignment with bureau scope`: Verifies bureau scoping
- `should create role assignment with employer scope`: Verifies employer scoping
- `should enforce unique user+role+scope`: Verifies duplicate prevention
- `should cascade delete on user removal`: Verifies cascade behavior
- `should validate scope type compatibility`: Verifies user type vs scope type

## Verification
```bash
npm run db:generate
npm run db:migrate
npm run test:unit src/lib/db/schema/user-role-assignments.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Data Models → UserRoleAssignment entity
