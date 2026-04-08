# Slice a: Chasing Rules and Configuration

**Story:** story-05-client-chasing
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** story-04-task-tracking

---

## Goal

Create the chasing rules engine and configuration UI for automated client reminders. Allow per-client configuration of reminder timing, frequency, and escalation rules.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x, tRPC 11.x, React Hook Form 7.x
- [x] All SDK methods/API calls identified: tRPC chasing configuration CRUD
- [x] All external service endpoints specified: N/A (internal API)
- [x] All data contracts defined: ChasingConfig schema, ChasingRules
- [x] All configuration/environment variables listed: DEFAULT_CHASE_DAYS (default: 2)
- [x] All error scenarios identified with handling strategy: Validation errors
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 3: Automated Client Chasing
- 02-04-bureau-operations-spec.md § Data Models — Client configuration patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/chasing-config.ts` | create | Chasing configuration table |
| `src/server/routers/chasing-config.ts` | create | tRPC router for chasing config |
| `src/components/chasing/chasing-config-form.tsx` | create | Configuration form UI |
| `src/components/chasing/chasing-rules-editor.tsx` | create | Rules editor component |
| `src/lib/validation/chasing.ts` | create | Zod schemas |
| `src/tests/server/chasing-config.test.ts` | create | Router tests |

---

## Responsibilities

1. Define ChasingConfig table with per-employer settings
2. Implement default rules that apply when no custom config
3. Create configuration UI for bureau admins
4. Support enabling/disabling chasing per client
5. Configure timing (days before cut-off) and frequency

---

## Contracts

### ChasingConfig Schema

```typescript
// src/lib/db/schema/chasing-config.ts
export const chasingConfig = pgTable('chasing_config', {
  id: uuid('id').primaryKey().defaultRandom(),
  employerId: uuid('employer_id').notNull().references(() => employers.id).unique(),
  enabled: boolean('enabled').notNull().default(true),
  firstChaseDays: integer('first_chase_days').notNull().default(2), // days before cut-off
  secondChaseDays: integer('second_chase_days'), // days before cut-off for second chase
  escalationDays: integer('escalation_days').notNull().default(1), // days after first chase to escalate
  chaseTemplateId: uuid('chase_template_id').references(() => emailTemplates.id),
  reminderFrequency: varchar('reminder_frequency', { length: 20 }).notNull().default('once'), // once, daily, every_2_days
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});
```

### chasingConfig.get

- **Method:** tRPC query `chasingConfig.get`
- **Input:** z.object({ employerId: z.string().uuid() })
- **Output:** ChasingConfig (or default values if not set)
- **Errors:** NOT_FOUND (employer), UNAUTHORIZED
- **Auth:** protectedProcedure with chasing:view permission

### chasingConfig.update

- **Method:** tRPC mutation `chasingConfig.update`
- **Input:** z.object({ employerId: z.string().uuid(), config: ChasingConfigInput })

```typescript
export const ChasingConfigInput = z.object({
  enabled: z.boolean(),
  firstChaseDays: z.number().int().min(1).max(14),
  secondChaseDays: z.number().int().min(1).max(14).optional(),
  escalationDays: z.number().int().min(0).max(7),
  reminderFrequency: z.enum(['once', 'daily', 'every_2_days']),
});
```

- **Output:** ChasingConfig
- **Errors:** BAD_REQUEST, NOT_FOUND, UNAUTHORIZED
- **Auth:** protectedProcedure with chasing:configure permission

### ChasingRulesEngine

```typescript
// src/lib/services/chasing-rules.ts
export class ChasingRulesEngine {
  shouldChase(
    config: ChasingConfig,
    cutOffDate: Date,
    lastChaseDate?: Date,
    hasResponded: boolean
  ): { shouldChase: boolean; reason?: string };
  
  shouldEscalate(
    config: ChasingConfig,
    lastChaseDate: Date,
    hasResponded: boolean
  ): boolean;
  
  getNextChaseDate(
    config: ChasingConfig,
    cutOffDate: Date
  ): Date;
}
```

---

## Business Rules & Invariants

1. Default chasing is enabled with 2 days before cut-off timing
2. Second chase only sent if first chase didn't get response
3. Escalation happens when no response after escalationDays
4. Chasing disabled if cut-off date is not set
5. Weekend chase dates roll to preceding Friday
6. Config changes apply to future pay periods only

---

## Edge Cases

1. **No cut-off date set** — Cannot chase, log warning
2. **Client responds** — Reset chase cycle for next period
3. **Multiple pay periods** — Chase based on earliest cut-off
4. **Disable chasing** — Stop all future chases, don't affect sent

---

## Tests

### chasing-config.test.ts

- Get config returns defaults when not configured
- Update config saves custom settings
- Validation rejects invalid days (negative, >14)
- Rules engine correctly determines chase timing
- Weekend cut-off rolls correctly

---

## Verification

```bash
npm run test:unit -- chasing-config.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 3: Automated Client Chasing
