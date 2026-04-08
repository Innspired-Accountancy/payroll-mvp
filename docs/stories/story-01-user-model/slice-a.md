# Slice a: User Entity Schema and Migrations

**Story:** story-01-user-model
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** none

## Goal

Create the User entity Drizzle ORM schema with all required fields, generate database migrations, and establish the foundation for identity management.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `drizzle-kit@latest`, `postgres@latest`, `uuid@latest`
- [x] SDK methods identified: `pgTable()`, `uuid()`, `varchar()`, `timestamp()`, `boolean()`, `integer()`, `index()`, `unique()`
- [x] External service endpoints: N/A (database only)
- [x] Data contracts defined: User table schema with TypeScript types
- [x] Configuration variables: `DATABASE_URL` environment variable
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Data Models → User entity fields
- 02-09-identity-access-spec.md:§ User Journeys → User data requirements

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/users.ts` | create | User table Drizzle schema |
| `src/lib/db/types/user.ts` | create | TypeScript types for User |
| `src/lib/db/migrations/0001_user_table.sql` | create | Generated migration |
| `src/lib/db/index.ts` | update | Export user schema |

## Responsibilities
1. Define User table with all fields per spec
2. Create proper indexes for email lookups and tenant queries
3. Generate migration files with drizzle-kit
4. Define TypeScript interfaces for type safety
5. Set up soft-delete pattern via status field

## Contracts

### User Table Schema (Drizzle)
```typescript
export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  passwordHash: varchar('password_hash', { length: 255 }),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  userType: varchar('user_type', { length: 20 }).notNull(), // 'bureau' | 'client' | 'employee'
  bureauId: uuid('bureau_id').references(() => bureaus.id),
  employerId: uuid('employer_id').references(() => employers.id),
  employeeId: uuid('employee_id').references(() => employees.id),
  status: varchar('status', { length: 20 }).notNull().default('pending'), // 'active' | 'inactive' | 'pending'
  mfaEnabled: boolean('mfa_enabled').notNull().default(false),
  mfaSecret: varchar('mfa_secret', { length: 255 }),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  passwordChangedAt: timestamp('password_changed_at', { withTimezone: true }),
  failedLoginAttempts: integer('failed_login_attempts').notNull().default(0),
  lockedUntil: timestamp('locked_until', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  emailIdx: index('users_email_idx').on(table.email),
  bureauIdx: index('users_bureau_idx').on(table.bureauId),
  employerIdx: index('users_employer_idx').on(table.employerId),
  statusIdx: index('users_status_idx').on(table.status),
}));
```

### TypeScript Types
```typescript
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
export type UserType = 'bureau' | 'client' | 'employee';
export type UserStatus = 'active' | 'inactive' | 'pending';
```

## Business Rules & Invariants
1. Email must be unique across all users (case-insensitive)
2. One of bureau_id, employer_id, or employee_id must be set based on user_type
3. password_hash is nullable for pending invitations
4. mfa_secret is encrypted when stored (handled in story-04-mfa)
5. status transitions: pending → active, active ↔ inactive

## Edge Cases
1. **Duplicate email on insert** — Return 409 Conflict with clear error message
2. **Missing required tenant relation** — Validate at application layer, return 400
3. **Long names (>100 chars)** — Truncate or return validation error
4. **Invalid user_type value** — Constraint violation, return 400

## Tests

### src/lib/db/schema/users.test.ts
- `should create user with all required fields`: Verifies insert with valid data
- `should enforce unique email constraint`: Verifies duplicate email rejection
- `should query user by email index`: Verifies email index performance
- `should handle soft delete via status`: Verifies status field filtering

## Verification
```bash
npm run db:generate
npm run db:migrate
npm run test:unit src/lib/db/schema/users.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Data Models → User entity schema
