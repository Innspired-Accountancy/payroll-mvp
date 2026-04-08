# Slice a: Communication Data Model and API

**Story:** story-06-communications
**Epic:** epic-04-bureau-operations
**Effort:** S
**Dependencies:** story-05-client-chasing

---

## Goal

Create the communication history data model and API for tracking all client interactions including automated reminders, manual emails, portal notifications, and responses.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x, tRPC 11.x
- [x] All SDK methods/API calls identified: tRPC communication CRUD
- [x] All external service endpoints specified: N/A (internal API)
- [x] All data contracts defined: ClientCommunication schema, CommunicationCreateInput
- [x] All configuration/environment variables listed: DATABASE_URL
- [x] All error scenarios identified with handling strategy: Validation errors
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Data Models — ClientCommunication entity
- 02-04-bureau-operations-spec.md § API Contracts — Communication logging

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/communications.ts` | create | Communication history table |
| `src/server/routers/communications.ts` | create | tRPC router |
| `src/lib/validation/communications.ts` | create | Zod schemas |
| `src/lib/services/communication-service.ts` | create | Business logic |
| `src/tests/server/communications-router.test.ts` | create | Router tests |

---

## Responsibilities

1. Define ClientCommunication table with all interaction fields
2. Implement communication.create for logging all communications
3. Implement communication.list with filtering by date, type, channel
4. Implement communication.markAsOpened for tracking
5. Implement communication.markAsResponded for response tracking
6. Support export for audit purposes

---

## Contracts

### ClientCommunication Schema

```typescript
// src/lib/db/schema/communications.ts
export const communicationTypeEnum = pgEnum('communication_type', [
  'reminder',
  'chase',
  'approval_request',
  'notification',
  'manual_email',
  'portal_message'
]);

export const communicationChannelEnum = pgEnum('communication_channel', [
  'email',
  'portal',
  'sms'
]);

export const communicationSentByEnum = pgEnum('communication_sent_by', [
  'system',
  'user'
]);

export const clientCommunications = pgTable('client_communications', {
  id: uuid('id').primaryKey().defaultRandom(),
  employerId: uuid('employer_id').notNull().references(() => employers.id),
  communicationType: communicationTypeEnum('communication_type').notNull(),
  channel: communicationChannelEnum('channel').notNull(),
  subject: varchar('subject', { length: 500 }).notNull(),
  content: text('content').notNull(),
  contentPreview: varchar('content_preview', { length: 200 }),
  sentBy: communicationSentByEnum('sent_by').notNull(),
  sentByUserId: uuid('sent_by_user_id').references(() => users.id),
  recipientEmail: varchar('recipient_email', { length: 255 }),
  messageId: varchar('message_id', { length: 255 }),
  sentAt: timestamp('sent_at').notNull().defaultNow(),
  openedAt: timestamp('opened_at'),
  responseReceived: boolean('response_received').notNull().default(false),
  responseAt: timestamp('response_at'),
  ipAddress: varchar('ip_address', { length: 45 }),
  userAgent: text('user_agent'),
  metadata: jsonb('metadata'), // tracking pixels, links clicked, etc.
});
```

### communications.create

- **Method:** tRPC mutation `communications.create`
- **Input:** CommunicationCreateInput

```typescript
export const CommunicationCreateInput = z.object({
  employerId: z.string().uuid(),
  communicationType: z.enum(['reminder', 'chase', 'approval_request', 'notification', 'manual_email', 'portal_message']),
  channel: z.enum(['email', 'portal', 'sms']),
  subject: z.string().min(1).max(500),
  content: z.string().min(1),
  sentBy: z.enum(['system', 'user']),
  sentByUserId: z.string().uuid().optional(),
  recipientEmail: z.string().email().optional(),
  messageId: z.string().optional(),
});
```

- **Output:** ClientCommunication
- **Errors:** BAD_REQUEST, NOT_FOUND, UNAUTHORIZED
- **Auth:** protectedProcedure with communication:create permission

### communications.list

- **Method:** tRPC query `communications.list`
- **Input:** CommunicationListInput

```typescript
export const CommunicationListInput = z.object({
  employerId: z.string().uuid(),
  type: z.enum(['reminder', 'chase', 'approval_request', 'notification', 'manual_email', 'portal_message']).optional(),
  channel: z.enum(['email', 'portal', 'sms']).optional(),
  sentBy: z.enum(['system', 'user']).optional(),
  dateFrom: z.string().date().optional(),
  dateTo: z.string().date().optional(),
  responseReceived: z.boolean().optional(),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
});
```

- **Output:** Paginated communication list
- **Errors:** UNAUTHORIZED
- **Auth:** protectedProcedure with communication:view permission

### communications.markAsOpened

- **Method:** tRPC mutation `communications.markAsOpened`
- **Input:** z.object({ id: z.string().uuid(), openedAt: z.string().datetime() })
- **Output:** ClientCommunication
- **Errors:** NOT_FOUND
- **Auth:** Public (called from email tracking pixel)

### communications.markAsResponded

- **Method:** tRPC mutation `communications.markAsResponded`
- **Input:** z.object({ id: z.string().uuid() })
- **Output:** ClientCommunication
- **Errors:** NOT_FOUND
- **Auth:** protectedProcedure

---

## Business Rules & Invariants

1. All emails sent via system are automatically logged
2. Manual emails logged when sent through platform
3. Portal messages logged when created
4. openedAt set only once (first open)
5. responseReceived set when reply detected or portal response received
6. contentPreview auto-generated from content (first 200 chars)

---

## Edge Cases

1. **Email forwarded** — Original recipient tracked, not forwarder
2. **Multiple opens** — Only first open recorded
3. **Bounced email** — Logged with status 'bounced'
4. **Long content** — Truncated for preview, full in content field

---

## Tests

### communications-router.test.ts

- Create communication logs correctly
- List returns communications for employer
- Mark as opened sets openedAt
- Mark as responded sets responseReceived
- Unauthorized user cannot view communications

---

## Verification

```bash
npm run test:unit -- communications-router.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Data Models → ClientCommunication entity
