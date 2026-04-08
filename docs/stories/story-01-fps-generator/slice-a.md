# Slice a: FPS Data Model and Database Schema

**Story:** story-01-fps-generator
**Epic:** epic-02-hmrc-submissions
**Effort:** S
**Dependencies:** None

---

## Goal

Define the HmrcSubmission data model and related database schema to store FPS/EPS submissions with full audit trail, correlation tracking, and support for corrections.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, PostgreSQL 15.x, UUID v7
- [x] Data contracts defined: HmrcSubmission, HmrcSubmissionError Zod schemas
- [x] Database relations: Foreign keys to employers, paye_schemes, pay_runs
- [x] Configuration: Database connection via DATABASE_URL environment variable
- [x] Error scenarios: Database constraint violations, relation integrity
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmission entity definition
- 02-02-hmrc-submissions-spec.md § API Contracts → Submission response shapes

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/hmrc-submissions.ts` | create | Drizzle table schema for submissions |
| `src/lib/db/schema/hmrc-submission-errors.ts` | create | Drizzle table schema for errors |
| `src/lib/db/schema/index.ts` | update | Export new schemas |
| `src/lib/validation/hmrc-submissions.ts` | create | Zod schemas for type safety |

---

## Responsibilities

1. Define HmrcSubmission table with all RTI-required fields
2. Define HmrcSubmissionError table for validation/rejection errors
3. Establish foreign key relationships to employers and PAYE schemes
4. Create indexes for common query patterns (status, correlation_id, tax_year)
5. Support submission versioning for corrections

---

## Contracts

### HmrcSubmission Schema (Drizzle)
```typescript
{
  id: uuid('id').primaryKey().defaultRandom(),
  employer_id: uuid('employer_id').notNull().references(() => employers.id),
  paye_scheme_id: uuid('paye_scheme_id').notNull().references(() => payeSchemes.id),
  pay_run_id: uuid('pay_run_id').references(() => payRuns.id),
  type: varchar('type', { length: 10 }).notNull(), // 'fps' | 'eps'
  tax_year: varchar('tax_year', { length: 7 }).notNull(), // '2026-27'
  tax_period: integer('tax_period').notNull(), // 1-12 for monthly, 1-52 for weekly
  status: varchar('status', { length: 20 }).notNull().default('draft'),
  // 'draft' | 'validated' | 'submitted' | 'acknowledged' | 'accepted' | 'rejected' | 'resubmitted'
  correlation_id: varchar('correlation_id', { length: 50 }).notNull().unique(),
  hmrc_correlation_id: varchar('hmrc_correlation_id', { length: 100 }),
  payload_xml: text('payload_xml').notNull(),
  payload_hash: varchar('payload_hash', { length: 64 }).notNull(), // SHA256
  response_xml: text('response_xml'),
  submitted_at: timestamp('submitted_at', { withTimezone: true }),
  acknowledged_at: timestamp('acknowledged_at', { withTimezone: true }),
  submission_version: integer('submission_version').notNull().default(1),
  parent_submission_id: uuid('parent_submission_id').references(() => hmrcSubmissions.id),
  late_reason: varchar('late_reason', { length: 1 }), // 'H'|'I'|'J'|'K'|'L'|'M'
  is_final: boolean('is_final').notNull().default(false),
  created_at: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updated_at: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow()
}
```

### HmrcSubmissionError Schema (Drizzle)
```typescript
{
  id: uuid('id').primaryKey().defaultRandom(),
  submission_id: uuid('submission_id').notNull().references(() => hmrcSubmissions.id, { onDelete: 'cascade' }),
  error_code: varchar('error_code', { length: 20 }).notNull(),
  error_message: text('error_message').notNull(),
  severity: varchar('severity', { length: 10 }).notNull(), // 'warning' | 'blocker'
  employee_id: uuid('employee_id').references(() => employees.id),
  field_path: varchar('field_path', { length: 500 }), // XPath in XML
  resolved: boolean('resolved').notNull().default(false),
  created_at: timestamp('created_at', { withTimezone: true }).notNull().defaultNow()
}
```

### Zod Validation Schema
```typescript
const hmrcSubmissionSchema = z.object({
  id: z.string().uuid(),
  employer_id: z.string().uuid(),
  paye_scheme_id: z.string().uuid(),
  pay_run_id: z.string().uuid().optional(),
  type: z.enum(['fps', 'eps']),
  tax_year: z.string().regex(/^\d{4}-\d{2}$/),
  tax_period: z.number().int().min(1).max(52),
  status: z.enum(['draft', 'validated', 'submitted', 'acknowledged', 'accepted', 'rejected', 'resubmitted']),
  correlation_id: z.string().max(50),
  submission_version: z.number().int().min(1).default(1),
  is_final: z.boolean().default(false)
});
```

---

## Business Rules & Invariants

1. correlation_id must be globally unique and immutable after creation
2. payload_hash is SHA256 of payload_xml, verified before storage
3. parent_submission_id only set for corrections (submission_version > 1)
4. status transitions must follow state machine (no draft → accepted)
5. is_final can only be true for submissions in tax period containing 5 April
6. late_reason required for submissions after 19th of tax month

---

## Edge Cases

1. **Duplicate correlation_id** — Database unique constraint prevents insertion
2. **Orphaned parent submission** — Foreign key with ON DELETE RESTRICT
3. **Very large XML payload** — TEXT type supports up to 1GB
4. **Concurrent submissions** — Row-level locking on pay_run_id

---

## Tests

### hmrc-submissions.schema.test.ts
- Create submission with valid data
- Reject duplicate correlation_id (expect unique constraint error)
- Cascade delete errors when submission deleted
- Foreign key constraint on invalid employer_id
- Payload hash length validation (64 chars)

---

## Verification

```bash
npm run db:generate  # Generate Drizzle migrations
npm run db:migrate   # Apply migrations
npm run typecheck    # Verify TypeScript types
npm run test src/lib/db/schema/hmrc-submissions.schema.test.ts
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmission entity
- 02-02-hmrc-submissions-spec.md § Data Models → HmrcSubmissionError entity
