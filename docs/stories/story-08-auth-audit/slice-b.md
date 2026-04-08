# Slice b: Authentication Event Logging

**Story:** story-08-auth-audit
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** slice-a

## Goal

Implement logging for all authentication events including successful/failed logins, password resets, account lockouts, and MFA events.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`
- [x] SDK methods identified: `db.insert()`, async logging function
- [x] External service endpoints: N/A
- [x] Data contracts defined: Log function signature, event builders
- [x] Configuration variables: `AUDIT_LOG_ASYNC=true`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Acceptance Criteria → Login success/failure logging
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Audit trail

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/audit/logger.ts` | create | Audit logging service |
| `src/lib/audit/events/auth.ts` | create | Authentication event builders |

## Responsibilities
1. Log login success/failure with reason codes
2. Log password reset events
3. Log account lockout/unlock events
4. Integrate with auth router
5. Support async logging (fire-and-forget)

## Contracts

### Audit Logger
```typescript
// src/lib/audit/logger.ts
import { db } from '@/lib/db';
import { auditLogs } from '@/lib/db/schema';
import type { AuditEvent } from './types';

export interface LogContext {
  actorId?: string;
  actorEmail?: string;
  ipAddress?: string;
  userAgent?: string;
  sessionId?: string;
  requestId?: string;
}

/**
 * Log an audit event
 */
export async function logAuditEvent(
  event: AuditEvent,
  context?: LogContext
): Promise<void> {
  try {
    await db.insert(auditLogs).values({
      eventType: event.type,
      eventCategory: event.category,
      severity: event.severity || 'info',
      actorId: event.actorId || context?.actorId,
      actorEmail: event.actorEmail || context?.actorEmail,
      actorIp: context?.ipAddress,
      actorUserAgent: context?.userAgent,
      targetId: event.targetId,
      targetType: event.targetType,
      description: event.description,
      details: event.details,
      success: event.success ?? true,
      errorCode: event.errorCode,
      errorMessage: event.errorMessage,
      sessionId: event.sessionId || context?.sessionId,
      requestId: context?.requestId,
    });
  } catch (error) {
    // Fallback logging - don't let audit failures break the app
    console.error('Failed to write audit log:', error);
    console.error('Audit event:', JSON.stringify(event));
    
    // Could also write to file or external service here
  }
}

/**
 * Fire-and-forget audit logging (for non-critical paths)
 */
export function logAuditEventAsync(
  event: AuditEvent,
  context?: LogContext
): void {
  // Don't await - log in background
  logAuditEvent(event, context).catch(err => {
    console.error('Async audit log failed:', err);
  });
}
```

### Auth Event Builders
```typescript
// src/lib/audit/events/auth.ts
import { logAuditEvent, type LogContext } from '../logger';
import { AuditEventCategories, AuditEventTypes, AuditSeverities } from '../types';

export function logLoginSuccess(
  userId: string,
  email: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.LOGIN_SUCCESS,
    category: AuditEventCategories.AUTH,
    severity: AuditSeverities.INFO,
    actorId: userId,
    actorEmail: email,
    targetId: userId,
    targetType: 'user',
    description: 'User logged in successfully',
    success: true,
  }, context);
}

export function logLoginFailure(
  email: string,
  reason: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.LOGIN_FAILURE,
    category: AuditEventCategories.AUTH,
    severity: AuditSeverities.WARNING,
    actorEmail: email,
    targetType: 'user',
    description: `Login failed: ${reason}`,
    success: false,
    errorCode: reason,
  }, context);
}

export function logLogout(
  userId: string,
  email: string,
  sessionId: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.LOGOUT,
    category: AuditEventCategories.AUTH,
    severity: AuditSeverities.INFO,
    actorId: userId,
    actorEmail: email,
    targetId: userId,
    targetType: 'user',
    description: 'User logged out',
    success: true,
    sessionId,
  }, context);
}

export function logPasswordResetRequested(
  userId: string,
  email: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.PASSWORD_RESET_REQUESTED,
    category: AuditEventCategories.AUTH,
    severity: AuditSeverities.INFO,
    actorId: userId,
    actorEmail: email,
    targetId: userId,
    targetType: 'user',
    description: 'Password reset requested',
    success: true,
  }, context);
}

export function logPasswordResetCompleted(
  userId: string,
  email: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.PASSWORD_RESET_COMPLETED,
    category: AuditEventCategories.AUTH,
    severity: AuditSeverities.INFO,
    actorId: userId,
    actorEmail: email,
    targetId: userId,
    targetType: 'user',
    description: 'Password reset completed',
    success: true,
  }, context);
}

export function logAccountLocked(
  userId: string,
  email: string,
  reason: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.ACCOUNT_LOCKED,
    category: AuditEventCategories.SECURITY,
    severity: AuditSeverities.WARNING,
    actorId: userId,
    actorEmail: email,
    targetId: userId,
    targetType: 'user',
    description: `Account locked: ${reason}`,
    success: false,
    details: { reason },
  }, context);
}

export function logMFAEnabled(
  userId: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.MFA_ENABLED,
    category: AuditEventCategories.MFA,
    severity: AuditSeverities.INFO,
    actorId: userId,
    targetId: userId,
    targetType: 'user',
    description: 'MFA enabled for user',
    success: true,
  }, context);
}

export function logMFAFailed(
  userId: string,
  reason: string,
  context: LogContext
): void {
  logAuditEventAsync({
    type: AuditEventTypes.MFA_FAILED,
    category: AuditEventCategories.MFA,
    severity: AuditSeverities.WARNING,
    actorId: userId,
    targetId: userId,
    targetType: 'user',
    description: `MFA verification failed: ${reason}`,
    success: false,
    errorCode: reason,
  }, context);
}
```

## Business Rules & Invariants
1. All auth events logged with actor and context
2. Failed login attempts logged with reason
3. Password changes logged for audit trail
4. Async logging for performance (don't block requests)
5. Fallback logging if database fails

## Edge Cases
1. **Log write fails** — Fallback to console, don't fail request
2. **Anonymous user** — Use email/IP for identification
3. **Missing context** — Log what we have, nulls ok
4. **High volume** — Batch inserts or use queue (future)
5. **Sensitive data** — Sanitize before logging

## Tests

### src/lib/audit/logger.test.ts
- `should log event to database`: Verifies insert
- `should handle missing context`: Verifies null handling
- `should not throw on failure`: Verifies error handling
- `should log auth event`: Verifies event logging

## Verification
```bash
npm run test:unit src/lib/audit/logger.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Acceptance Criteria → Authentication logging
