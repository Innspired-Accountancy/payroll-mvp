# Slice c: PostgreSQL RLS Policies

**Story:** story-06-tenant-isolation
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-b

## Goal

Implement PostgreSQL Row-Level Security (RLS) policies as defense-in-depth for tenant isolation, ensuring database-level enforcement of tenant boundaries.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `drizzle-kit@latest`
- [x] SDK methods identified: SQL `CREATE POLICY`, `ALTER TABLE ENABLE ROW LEVEL SECURITY`
- [x] External service endpoints: N/A
- [x] Data contracts defined: RLS policy definitions per table
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Multi-tenant scoping

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/migrations/rls_policies.sql` | create | RLS policy definitions |
| `src/lib/db/rls/policies.ts` | create | Policy generation helpers |
| `scripts/apply-rls.ts` | create | RLS application script |

## Responsibilities
1. Define RLS policies for all tenant-scoped tables
2. Create policies for SELECT, INSERT, UPDATE, DELETE
3. Generate migration SQL for policies
4. Support application-level tenant setting for RLS
5. Provide bypass for admin operations

## Contracts

### RLS Policy Definitions
```sql
-- src/lib/db/migrations/rls_policies.sql

-- Enable RLS on tenant-scoped tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE employers ENABLE ROW LEVEL SECURITY;
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
ALTER TABLE payroll_runs ENABLE ROW LEVEL SECURITY;
ALTER TABLE payslips ENABLE ROW LEVEL SECURITY;

-- Create application role for RLS
CREATE ROLE app_user NOLOGIN;

-- Users table policies
CREATE POLICY users_bureau_isolation ON users
  FOR ALL
  TO app_user
  USING (
    (current_setting('app.current_user_type', true) = 'bureau' AND 
     bureau_id = current_setting('app.current_bureau_id', true)::uuid)
    OR
    (current_setting('app.current_user_type', true) IN ('client', 'employee') AND 
     employer_id = current_setting('app.current_employer_id', true)::uuid)
  );

-- Employers table policies  
CREATE POLICY employers_bureau_isolation ON employers
  FOR ALL
  TO app_user
  USING (
    (current_setting('app.current_user_type', true) = 'bureau' AND 
     bureau_id = current_setting('app.current_bureau_id', true)::uuid)
    OR
    (current_setting('app.current_user_type') IN ('client', 'employee') AND 
     id = current_setting('app.current_employer_id', true)::uuid)
  );

-- Employees table policies
CREATE POLICY employees_employer_isolation ON employees
  FOR ALL
  TO app_user
  USING (
    employer_id = current_setting('app.current_employer_id', true)::uuid
  );

-- Payroll runs policies
CREATE POLICY payroll_runs_employer_isolation ON payroll_runs
  FOR ALL
  TO app_user
  USING (
    employer_id = current_setting('app.current_employer_id', true)::uuid
  );

-- Payslips policies
CREATE POLICY payslips_employer_isolation ON payslips
  FOR ALL
  TO app_user
  USING (
    employer_id = current_setting('app.current_employer_id', true)::uuid
  );

-- Grant permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON employers TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON employees TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON payroll_runs TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON payslips TO app_user;
```

### RLS Helper Functions
```typescript
// src/lib/db/rls/policies.ts
import { sql } from 'drizzle-orm';
import { db } from '@/lib/db';

export interface RLSContext {
  userType: 'bureau' | 'client' | 'employee';
  bureauId?: string;
  employerId?: string;
}

/**
 * Set RLS context for current database session
 */
export async function setRLSContext(context: RLSContext): Promise<void> {
  await db.execute(sql`
    SELECT set_config('app.current_user_type', ${context.userType}, false);
    SELECT set_config('app.current_bureau_id', ${context.bureauId || ''}, false);
    SELECT set_config('app.current_employer_id', ${context.employerId || ''}, false);
  `);
}

/**
 * Clear RLS context (for admin operations)
 */
export async function clearRLSContext(): Promise<void> {
  await db.execute(sql`
    SELECT set_config('app.current_user_type', '', false);
    SELECT set_config('app.current_bureau_id', '', false);
    SELECT set_config('app.current_employer_id', '', false);
  `);
}

/**
 * Execute function with admin bypass (no RLS)
 */
export async function withAdminBypass<T>(callback: () => Promise<T>): Promise<T> {
  await clearRLSContext();
  try {
    return await callback();
  } finally {
    // Context should be reset by caller
  }
}
```

## Business Rules & Invariants
1. RLS policies apply to all queries from app_user
2. Application must set tenant context before queries
3. Bureau users see all data for their bureau
4. Client/employee users see only their employer's data
5. Admin operations bypass RLS with elevated role

## Edge Cases
1. **Missing context** — RLS blocks all rows
2. **Invalid UUID format** — Query fails with error
3. **Admin needs cross-tenant report** — Use bypass or elevated role
4. **Policy misconfiguration** — Database error on query
5. **Performance impact** — Index tenant_id columns

## Tests

### src/lib/db/rls/policies.test.ts
- `should enforce bureau isolation`: Verifies bureau filtering
- `should enforce employer isolation`: Verifies employer filtering
- `should block cross-tenant access`: Verifies isolation
- `should allow admin bypass`: Verifies bypass
- `should block all rows without context`: Verifies strictness

## Verification
```bash
npm run db:migrate  # Applies RLS policies
npm run test:integration src/lib/db/rls/policies.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Non-Functional Requirements → Tenant isolation
