# Slice a: Audit Log Schema and Storage

**Story:** story-08-auth-audit
**Epic:** epic-05-identity-access
**Effort:** S
**Dependencies:** story-02-password-auth

## Goal

Create the audit log table schema for storing all authentication and authorization events with immutable records.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `uuid@latest`
- [x] SDK methods identified: `pgTable()`, `uuid()`, `timestamp()`, `jsonb()`, `index()`
- [x] External service endpoints: N/A
- [x] Data contracts defined: Audit event schema, event types
- [x] Configuration variables: `AUDIT_LOG_RETENTION_DAYS=90`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ Acceptance Criteria → Audit log table schema
- 02-09-identity-access-spec.md:§ Non-Functional Requirements → Audit trail

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/audit-logs.ts` | create | Audit log table schema |
| `src/lib/audit/types.ts` | create | Audit event type definitions |

## Responsibilities
1. Define audit log table with all required fields
2. Create indexes for common query patterns
3. Define event type constants
4. Support structured JSON details
5. Ensure audit records are append-only

## Contracts

### Audit Log Schema
```typescript
// src/lib/db/schema/audit-logs.ts
import { pgTable, uuid, timestamp, varchar, boolean, jsonb } from 'drizzle-orm/pg-core';
import { users } from './users';
import { relations } from 'drizzle-orm';

export const auditLogs = pgTable('audit_logs', {
  id: uuid('id').defaultRandom().primaryKey(),
  
  // Event classification
  eventType: varchar('event_type', { length: 50 }).notNull(),
  eventCategory: varchar('event_category', { length: 20 }).notNull(), // 'auth', 'authorization', 'session', 'user'
  severity: varchar('severity', { length: 10 }).notNull().default('info'), // 'info', 'warning', 'error', 'critical'
  
  // Actor (who performed the action)
  actorId: uuid('actor_id').references(() => users.id),
  actorEmail: varchar('actor_email', { length: 255 }),
  actorIp: varchar('actor_ip', { length: 45 }),
  actorUserAgent: varchar('actor_user_agent', { length: 500 }),
  
  // Target (what was affected)
  targetId: varchar('target_id', { length: 36 }),
  targetType: varchar('target_type', { length: 30 }), // 'user', 'role', 'session', etc.
  
  // Event details
  description: varchar('description', { length: 500 }),
  details: jsonb('details').$type<Record<string, any>>(),
  
  // Result
  success: boolean('success').notNull().default(true),
  errorCode: varchar('error_code', { length: 50 }),
  errorMessage: varchar('error_message', { length: 500 }),
  
  // Context
  sessionId: uuid('session_id'),
  requestId: varchar('request_id', { length: 36 }),
  
  // Timestamp (separate from created_at for clarity)
  timestamp: timestamp('timestamp', { withTimezone: true }).notNull().defaultNow(),
  
  // Immutable record marker
  immutable: boolean('immutable').notNull().default(true),
}, (table) => ({
  timestampIdx: index('audit_timestamp_idx').on(table.timestamp),
  actorIdx: index('audit_actor_idx').on(table.actorId),
  eventTypeIdx: index('audit_event_type_idx').on(table.eventType),
  categoryIdx: index('audit_category_idx').on(table.eventCategory),
  targetIdx: index('audit_target_idx').on(table.targetType, table.targetId),
  severityIdx: index('audit_severity_idx').on(table.severity),
}));

export const auditLogsRelations = relations(auditLogs, ({ one }) => ({
  actor: one(users, { fields: [auditLogs.actorId], references: [users.id] }),
}));

export type AuditLog = typeof auditLogs.$inferSelect;
export type NewAuditLog = typeof auditLogs.$inferInsert;
```

### Audit Event Types
```typescript
// src/lib/audit/types.ts
export const AuditEventCategories = {
  AUTH: 'auth',
  AUTHORIZATION: 'authorization',
  SESSION: 'session',
  USER: 'user',
  ROLE: 'role',
  MFA: 'mfa',
  SECURITY: 'security',
} as const;

export const AuditEventTypes = {
  // Auth events
  LOGIN_SUCCESS: 'LOGIN_SUCCESS',
  LOGIN_FAILURE: 'LOGIN_FAILURE',
  LOGOUT: 'LOGOUT',
  PASSWORD_RESET_REQUESTED: 'PASSWORD_RESET_REQUESTED',
  PASSWORD_RESET_COMPLETED: 'PASSWORD_RESET_COMPLETED',
  PASSWORD_CHANGED: 'PASSWORD_CHANGED',
  ACCOUNT_LOCKED: 'ACCOUNT_LOCKED',
  ACCOUNT_UNLOCKED: 'ACCOUNT_UNLOCKED',
  
  // Authorization events
  PERMISSION_DENIED: 'PERMISSION_DENIED',
  ROLE_ASSIGNED: 'ROLE_ASSIGNED',
  ROLE_REMOVED: 'ROLE_REMOVED',
  SOD_VIOLATION: 'SOD_VIOLATION',
  
  // Session events
  SESSION_CREATED: 'SESSION_CREATED',
  SESSION_TERMINATED: 'SESSION_TERMINATED',
  SESSION_EXPIRED: 'SESSION_EXPIRED',
  SESSION_REVIVED: 'SESSION_REVIVED',
  
  // User events
  USER_CREATED: 'USER_CREATED',
  USER_UPDATED: 'USER_UPDATED',
  USER_DELETED: 'USER_DELETED',
  USER_INVITED: 'USER_INVITED',
  USER_ACTIVATED: 'USER_ACTIVATED',
  USER_DEACTIVATED: 'USER_DEACTIVATED',
  
  // MFA events
  MFA_ENABLED: 'MFA_ENABLED',
  MFA_DISABLED: 'MFA_DISABLED',
  MFA_VERIFIED: 'MFA_VERIFIED',
  MFA_FAILED: 'MFA_FAILED',
  MFA_BACKUP_CODE_USED: 'MFA_BACKUP_CODE_USED',
  
  // Security events
  SUSPICIOUS_ACTIVITY: 'SUSPICIOUS_ACTIVITY',
  RATE_LIMIT_EXCEEDED: 'RATE_LIMIT_EXCEEDED',
  TOKEN_REUSE_DETECTED: 'TOKEN_REUSE_DETECTED',
} as const;

export const AuditSeverities = {
  INFO: 'info',
  WARNING: 'warning',
  ERROR: 'error',
  CRITICAL: 'critical',
} as const;

export interface AuditEvent {
  type: keyof typeof AuditEventTypes;
  category: keyof typeof AuditEventCategories;
  severity?: keyof typeof AuditSeverities;
  actorId?: string;
  actorEmail?: string;
  targetId?: string;
  targetType?: string;
  description?: string;
  details?: Record<string, any>;
  success?: boolean;
  errorCode?: string;
  errorMessage?: string;
  sessionId?: string;
}
```

## Business Rules & Invariants
1. Audit logs are append-only (no updates or deletes)
2. Immutable flag set on all records
3. Timestamps in UTC with timezone info
4. PII minimized in logs (emails ok, passwords never)
5. Indexes support common query patterns

## Edge Cases
1. **Actor is anonymous** — actorId null, track IP instead
2. **Database write fails** — Log to fallback (file/external)
3. **Very large details** — Truncate or store reference
4. **Missing target** — targetId null for general events
5. **Clock skew** — Use database timestamp (now())

## Tests

### src/lib/db/schema/audit-logs.test.ts
- `should create audit log entry`: Verifies insert
- `should enforce required fields`: Verifies constraints
- `should index timestamp`: Verifies index usage
- `should store JSON details`: Verifies JSONB

## Verification
```bash
npm run db:generate
npm run test:unit src/lib/db/schema/audit-logs.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § Acceptance Criteria → Audit log schema
