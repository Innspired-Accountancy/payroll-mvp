# Slice c: Permission and Session Logging

**Story:** story-08-auth-audit
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-b

## Goal

Implement logging for permission changes, role assignments, and session lifecycle events.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`
- [x] SDK methods identified: Log functions for role/session events
- [x] External service endpoints: N/A
- [x] Data contracts defined: Role/session event builders
- [x] Configuration variables: N/A
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Acceptance Criteria → Permission grant/revoke logging
- 02-09-identity-access-spec.md:§ Acceptance Criteria → Session create/terminate logging

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/audit/events/role.ts` | create | Role/permission event builders |
| `src/lib/audit/events/session.ts` | create | Session event builders |

## Responsibilities
1. Log role assignments and removals
2. Log permission denied events
3. Log session creation and termination
4. Log session timeouts
5. Integrate with role and session services

## Contracts

### Role Event Builders
```typescript
// src/lib/audit/events/role.ts
import { logAuditEventAsync } from '../logger';
import { AuditEventCategories, AuditEventTypes, AuditSeverities } from '../types';
import type { LogContext } from '../logger';

export function logRoleAssigned(
  actorId: string,
  targetUserId: string,
  roleId: string,
  roleName: string,
  scopeType: string,
  scopeId: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.ROLE_ASSIGNED,
    category: AuditEventCategories.AUTHORIZATION,
    severity: AuditSeverities.INFO,
    actorId,
    targetId: targetUserId,
    targetType: 'user',
    description: `Role "${roleName}" assigned to user`,
    success: true,
    details: {
      roleId,
      roleName,
      scopeType,
      scopeId,
    },
  }, context);
}

export function logRoleRemoved(
  actorId: string,
  targetUserId: string,
  roleId: string,
  roleName: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.ROLE_REMOVED,
    category: AuditEventCategories.AUTHORIZATION,
    severity: AuditSeverities.INFO,
    actorId,
    targetId: targetUserId,
    targetType: 'user',
    description: `Role "${roleName}" removed from user`,
    success: true,
    details: {
      roleId,
      roleName,
    },
  }, context);
}

export function logPermissionDenied(
  userId: string,
  permission: string,
  resource: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.PERMISSION_DENIED,
    category: AuditEventCategories.AUTHORIZATION,
    severity: AuditSeverities.WARNING,
    actorId: userId,
    targetType: resource,
    description: `Permission denied: ${permission}`,
    success: false,
    errorCode: 'PERMISSION_DENIED',
    details: {
      requiredPermission: permission,
      resource,
    },
  }, context);
}

export function logSoDViolation(
  actorId: string,
  targetUserId: string,
  ruleId: string,
  ruleName: string,
  conflictingPermissions: string[],
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.SOD_VIOLATION,
    category: AuditEventCategories.SECURITY,
    severity: AuditSeverities.ERROR,
    actorId,
    targetId: targetUserId,
    targetType: 'user',
    description: `Segregation of duties violation: ${ruleName}`,
    success: false,
    details: {
      ruleId,
      ruleName,
      conflictingPermissions,
    },
  }, context);
}
```

### Session Event Builders
```typescript
// src/lib/audit/events/session.ts
import { logAuditEventAsync } from '../logger';
import { AuditEventCategories, AuditEventTypes, AuditSeverities } from '../types';
import type { LogContext } from '../logger';

export function logSessionCreated(
  userId: string,
  sessionId: string,
  deviceType: string,
  browser: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.SESSION_CREATED,
    category: AuditEventCategories.SESSION,
    severity: AuditSeverities.INFO,
    actorId: userId,
    targetId: sessionId,
    targetType: 'session',
    description: 'New session created',
    success: true,
    sessionId,
    details: {
      deviceType,
      browser,
    },
  }, context);
}

export function logSessionTerminated(
  userId: string,
  sessionId: string,
  reason: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.SESSION_TERMINATED,
    category: AuditEventCategories.SESSION,
    severity: AuditSeverities.INFO,
    actorId: userId,
    targetId: sessionId,
    targetType: 'session',
    description: `Session terminated: ${reason}`,
    success: true,
    sessionId,
    details: {
      reason,
    },
  }, context);
}

export function logSessionExpired(
  userId: string,
  sessionId: string,
  reason: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.SESSION_EXPIRED,
    category: AuditEventCategories.SESSION,
    severity: AuditSeverities.INFO,
    actorId: userId,
    targetId: sessionId,
    targetType: 'session',
    description: `Session expired: ${reason}`,
    success: false,
    sessionId,
    details: {
      reason,
    },
  }, context);
}

export function logTokenReuseDetected(
  sessionId: string,
  jti: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.TOKEN_REUSE_DETECTED,
    category: AuditEventCategories.SECURITY,
    severity: AuditSeverities.CRITICAL,
    targetId: sessionId,
    targetType: 'session',
    description: 'Refresh token reuse detected - possible theft',
    success: false,
    sessionId,
    details: {
      tokenJti: jti,
      action: 'session_family_revoked',
    },
  }, context);
}

export function logRateLimitExceeded(
  ipAddress: string,
  endpoint: string,
  limit: number,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.RATE_LIMIT_EXCEEDED,
    category: AuditEventCategories.SECURITY,
    severity: AuditSeverities.WARNING,
    targetType: 'endpoint',
    description: `Rate limit exceeded for ${endpoint}`,
    success: false,
    details: {
      endpoint,
      limit,
      ipAddress,
    },
  }, context);
}
```

## Business Rules & Invariants
1. All role changes logged with actor and target
2. Permission denials logged for security monitoring
3. Session lifecycle fully tracked
4. Security events (token reuse) logged as critical
5. Context includes IP and user agent when available

## Edge Cases
1. **System-initiated events** — actorId null, system as target
2. **Batch operations** — Log each individually or as batch event
3. **Session not found** — Log with available info
4. **Nested role changes** — Log at each level
5. **High frequency events** — Consider sampling for rate limits

## Tests

### src/lib/audit/events/role.test.ts
- `should log role assignment`: Verifies event
- `should log permission denial`: Verifies denial
- `should log SoD violation`: Verifies violation

## Verification
```bash
npm run test:unit src/lib/audit/events/role.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Acceptance Criteria → Permission/session logging
