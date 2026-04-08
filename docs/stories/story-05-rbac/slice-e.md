# Slice e: Segregation of Duties

**Story:** story-05-rbac
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** slice-d

## Goal

Implement segregation of duties (SoD) enforcement to prevent conflicting roles being assigned to the same user, ensuring compliance with internal controls.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`
- [x] SDK methods identified: Array intersection, Set operations
- [x] External service endpoints: N/A
- [x] Data contracts defined: SoD rule structure, conflict result
- [x] Configuration variables: `SOD_VIOLATION_ACTION='block'` ('block' | 'warn')
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ User Journeys → Journey 4: Segregation Check
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Segregation of duties

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/roles/sod/rules.ts` | create | SoD rule definitions |
| `src/lib/roles/sod/check.ts` | create | SoD validation logic |
| `src/lib/roles/sod/matrix.ts` | create | Permission conflict matrix |

## Responsibilities
1. Define SoD rules (conflicting permission sets)
2. Validate role assignments against SoD rules
3. Block or warn on SoD violations
4. Provide audit logging for SoD checks
5. Support configurable enforcement levels

## Contracts

### SoD Rules
```typescript
// src/lib/roles/sod/rules.ts
import type { PermissionId } from '@/lib/permissions/types';

export interface SoDRule {
  id: string;
  name: string;
  description: string;
  permissionSetA: PermissionId[];
  permissionSetB: PermissionId[];
  severity: 'critical' | 'high' | 'medium';
  enforcement: 'block' | 'warn';
}

export const SOD_RULES: SoDRule[] = [
  {
    id: 'sod-payroll-create-approve',
    name: 'Payroll Creation and Approval',
    description: 'Users cannot both create/edit payroll and approve payroll',
    permissionSetA: ['payroll:create', 'payroll:edit'],
    permissionSetB: ['payroll:approve'],
    severity: 'critical',
    enforcement: 'block',
  },
  {
    id: 'sod-payment-prepare-approve',
    name: 'Payment Preparation and Approval',
    description: 'Users cannot prepare payments and approve payments',
    permissionSetA: ['payroll:create', 'payroll:edit'],
    permissionSetB: ['payments:approve'],
    severity: 'critical',
    enforcement: 'block',
  },
  {
    id: 'sod-employee-create-delete',
    name: 'Employee Creation and Deletion',
    description: 'Users should not both create and delete employees',
    permissionSetA: ['employee:create'],
    permissionSetB: ['employee:delete'],
    severity: 'medium',
    enforcement: 'warn',
  },
  {
    id: 'sod-payroll-cis',
    name: 'Payroll and CIS Separation',
    description: 'Large operations may require separate CIS and payroll processors',
    permissionSetA: ['payroll:edit', 'payroll:approve'],
    permissionSetB: ['cis:manage'],
    severity: 'medium',
    enforcement: 'warn',
  },
];

export function getSoDRules(): SoDRule[] {
  return SOD_RULES;
}

export function getSoDRuleById(id: string): SoDRule | undefined {
  return SOD_RULES.find(r => r.id === id);
}
```

### SoD Checker
```typescript
// src/lib/roles/sod/check.ts
import { SOD_RULES, type SoDRule } from './rules';
import type { PermissionId } from '@/lib/permissions/types';

export interface SoDViolation {
  rule: SoDRule;
  conflictingPermissions: PermissionId[];
  fromSetA: PermissionId[];
  fromSetB: PermissionId[];
}

export interface SoDCheckResult {
  compliant: boolean;
  violations: SoDViolation[];
  warnings: SoDViolation[];
}

export class SoDChecker {
  /**
   * Check permissions against SoD rules
   */
  static checkPermissions(permissions: PermissionId[]): SoDCheckResult {
    const violations: SoDViolation[] = [];
    const warnings: SoDViolation[] = [];
    
    for (const rule of SOD_RULES) {
      // Check if user has any permission from set A
      const hasSetA = rule.permissionSetA.some(p => permissions.includes(p));
      // Check if user has any permission from set B
      const hasSetB = rule.permissionSetB.some(p => permissions.includes(p));
      
      if (hasSetA && hasSetB) {
        const fromSetA = rule.permissionSetA.filter(p => permissions.includes(p));
        const fromSetB = rule.permissionSetB.filter(p => permissions.includes(p));
        
        const violation: SoDViolation = {
          rule,
          conflictingPermissions: [...fromSetA, ...fromSetB],
          fromSetA,
          fromSetB,
        };
        
        if (rule.enforcement === 'block') {
          violations.push(violation);
        } else {
          warnings.push(violation);
        }
      }
    }
    
    return {
      compliant: violations.length === 0,
      violations,
      warnings,
    };
  }
  
  /**
   * Check a proposed role assignment for SoD conflicts
   */
  static async checkRoleAssignment(
    userId: string,
    newRolePermissions: PermissionId[],
    existingPermissions: PermissionId[]
  ): Promise<SoDCheckResult> {
    // Combine existing and new permissions
    const combinedPermissions = Array.from(new Set([
      ...existingPermissions,
      ...newRolePermissions,
    ]));
    
    return this.checkPermissions(combinedPermissions);
  }
  
  /**
   * Check if two roles have SoD conflicts
   */
  static checkRoles(
    roleAPermissions: PermissionId[],
    roleBPermissions: PermissionId[]
  ): SoDCheckResult {
    const combined = Array.from(new Set([...roleAPermissions, ...roleBPermissions]));
    return this.checkPermissions(combined);
  }
  
  /**
   * Format violation for display
   */
  static formatViolation(violation: SoDViolation): string {
    const { rule, fromSetA, fromSetB } = violation;
    return `${rule.name}: Cannot have both [${fromSetA.join(', ')}] and [${fromSetB.join(', ')}]`;
  }
}
```

### Updated Assignment Service with SoD
```typescript
// Add to src/lib/roles/assignment.ts
import { SoDChecker } from './sod/check';

// In RoleAssignmentService.assign method, add before creating assignment:

// Check SoD compliance
const existingAssignments = await this.getUserRoles(input.userId);
const existingPermissions: string[] = [];

for (const assignment of existingAssignments) {
  existingPermissions.push(...assignment.role.permissions);
}

const newRole = await db.query.roles.findFirst({
  where: eq(roles.id, input.roleId),
});

if (newRole) {
  const sodCheck = await SoDChecker.checkRoleAssignment(
    input.userId,
    newRole.permissions,
    existingPermissions
  );
  
  if (!sodCheck.compliant) {
    const violations = sodCheck.violations.map(v => SoDChecker.formatViolation(v));
    throw new Error(`Segregation of duties violation: ${violations.join('; ')}`);
  }
  
  // Log warnings if any
  for (const warning of sodCheck.warnings) {
    await logSecurityEvent({
      type: 'SOD_WARNING',
      userId: input.userId,
      details: {
        ruleId: warning.rule.id,
        description: warning.rule.description,
      },
    });
  }
}
```

### tRPC SoD Router
```typescript
// src/server/trpc/routers/sod.ts
import { z } from 'zod';
import { authenticatedProcedure, router } from '../trpc';
import { requirePermission } from '../middleware/auth';
import { SoDChecker } from '@/lib/roles/sod/check';
import { SOD_RULES } from '@/lib/roles/sod/rules';
import { getUserPermissions } from '@/lib/permissions/check';

export const sodRouter = router({
  // Get all SoD rules
  listRules: authenticatedProcedure
    .use(requirePermission('role:manage'))
    .query(() => {
      return SOD_RULES;
    }),
  
  // Check current user for SoD violations
  checkUser: authenticatedProcedure
    .input(z.object({ userId: z.string().uuid() }))
    .query(async ({ input, ctx }) => {
      // Users can check themselves, or need role:manage permission
      if (input.userId !== ctx.user!.sub) {
        requirePermission('role:manage');
      }
      
      const permissions = await getUserPermissions(input.userId);
      const permissionIds = permissions.map(p => p.permissionId);
      
      const result = SoDChecker.checkPermissions(permissionIds);
      
      return {
        compliant: result.compliant,
        violations: result.violations.map(v => ({
          rule: v.rule,
          conflictingPermissions: v.conflictingPermissions,
          message: SoDChecker.formatViolation(v),
        })),
        warnings: result.warnings.map(v => ({
          rule: v.rule,
          message: SoDChecker.formatViolation(v),
        })),
      };
    }),
  
  // Preview SoD check for proposed assignment
  previewAssignment: authenticatedProcedure
    .use(requirePermission('user:manage'))
    .input(z.object({
      userId: z.string().uuid(),
      roleId: z.string().uuid(),
    }))
    .query(async ({ input }) => {
      const existingAssignments = await RoleAssignmentService.getUserRoles(input.userId);
      const existingPermissions = existingAssignments.flatMap(a => a.role.permissions);
      
      const newRole = await db.query.roles.findFirst({
        where: eq(roles.id, input.roleId),
      });
      
      if (!newRole) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: 'Role not found',
        });
      }
      
      const result = await SoDChecker.checkRoleAssignment(
        input.userId,
        newRole.permissions,
        existingPermissions
      );
      
      return {
        wouldBeCompliant: result.compliant,
        violations: result.violations,
        warnings: result.warnings,
      };
    }),
});

export type SodRouter = typeof sodRouter;
```

## Business Rules & Invariants
1. Critical SoD violations are blocked at assignment time
2. Medium severity conflicts generate warnings but are allowed
3. SoD checks run on every role assignment
4. Violations are logged for audit purposes
5. SoD rules are configurable in code

## Edge Cases
1. **Multiple violations** — Report all, not just first
2. **Existing violations** — Allow removal but warn on modification
3. **Rule modification** — Existing assignments grandfathered until change
4. **Circular permission sets** — Rules designed to avoid this
5. **Admin override** — Documented exception process required

## Tests

### src/lib/roles/sod/check.test.ts
- `should detect critical violation`: Verifies blocking
- `should allow compliant assignment`: Verifies approval
- `should generate warning for medium risk`: Verifies warning
- `should check combined permissions`: Verifies combination logic
- `should format violations for display`: Verifies formatting

## Verification
```bash
npm run test:unit src/lib/roles/sod/check.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § User Journeys → Journey 4: Segregation Check
- 02-09-identity-access-spec.md § Non-Functional Requirements → Segregation of duties
