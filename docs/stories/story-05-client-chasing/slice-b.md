# Slice b: Automated Reminder Job

**Story:** story-05-client-chasing
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement the automated reminder job that runs on a schedule to detect clients needing reminders and send chase emails. Integrate with the email service for delivery.

---

## Decision Checklist

- [x] All libraries/packages named: node-cron 3.x, Drizzle ORM 0.30.x, date-fns 3.x, Nodemailer 6.x
- [x] All SDK methods/API calls identified: cron.schedule(), emailService.send(), chasingRulesEngine.shouldChase()
- [x] All external service endpoints specified: SMTP/email service via Nodemailer
- [x] All data contracts defined: ChaseJobPayload, EmailSendInput
- [x] All configuration/environment variables listed: SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, CRON_CHASE_SCHEDULE (0 9 * * 1-5)
- [x] All error scenarios identified with handling strategy: Email delivery failures, retry logic
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 3: Automated Client Chasing
- 02-04-bureau-operations-spec.md § Integration Points — Email Service

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/jobs/chase-job.ts` | create | Main chase job implementation |
| `src/lib/services/chase-processor.ts` | create | Chase detection and processing |
| `src/lib/services/email-service.ts` | create | Email delivery service |
| `src/lib/templates/chase-email.ts` | create | Chase email template |
| `src/lib/db/schema/chase-log.ts` | create | Chase history logging |
| `src/tests/jobs/chase-job.test.ts` | create | Job tests |

---

## Responsibilities

1. Schedule daily job (weekdays at 9am) to check for chases
2. Query for pay periods with approaching cut-off dates
3. Apply chasing rules to determine if client should be chased
4. Generate personalized chase emails
5. Send emails via configured email service
6. Log all chase attempts with status

---

## Contracts

### ChaseJob

```typescript
// src/jobs/chase-job.ts
export async function runChaseJob(): Promise<ChaseJobResult> {
  // Runs every weekday at 9am
  // 1. Find pay periods needing data collection
  // 2. Apply chasing rules for each
  // 3. Send chase emails
  // 4. Log results
}

export interface ChaseJobResult {
  totalChecked: number;
  chased: number;
  failed: number;
  errors: Array<{ employerId: string; error: string }>;
}
```

### ChaseProcessor Service

```typescript
// src/lib/services/chase-processor.ts
export class ChaseProcessor {
  async findClientsToChase(date: Date): Promise<ChaseCandidate[]>;
  
  async processChase(
    candidate: ChaseCandidate,
    config: ChasingConfig
  ): Promise<ChaseResult>;
  
  private async buildEmailContent(
    candidate: ChaseCandidate,
    template: EmailTemplate
  ): Promise<{ subject: string; body: string }>;
  
  private async recordChase(
    employerId: string,
    chaseType: string,
    status: 'sent' | 'failed'
  ): Promise<void>;
}

interface ChaseCandidate {
  employerId: string;
  employerName: string;
  contactEmail: string;
  payPeriodId: string;
  cutOffDate: Date;
  daysUntilCutOff: number;
  lastChaseDate?: Date;
  chaseCount: number;
}
```

### EmailService

```typescript
// src/lib/services/email-service.ts
export class EmailService {
  private transporter: nodemailer.Transporter;
  
  async send(options: EmailSendOptions): Promise<EmailResult>;
  
  async sendTemplate(
    templateId: string,
    to: string,
    variables: Record<string, string>
  ): Promise<EmailResult>;
}

interface EmailSendOptions {
  to: string;
  from: string;
  subject: string;
  html: string;
  text?: string;
  replyTo?: string;
}

interface EmailResult {
  messageId: string;
  accepted: string[];
  rejected: string[];
}
```

### Chase Log Schema

```typescript
// src/lib/db/schema/chase-log.ts
export const chaseLog = pgTable('chase_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  employerId: uuid('employer_id').notNull().references(() => employers.id),
  payPeriodId: uuid('pay_period_id').notNull().references(() => payPeriods.id),
  chaseType: varchar('chase_type', { length: 50 }).notNull(), // first, second, escalation
  chaseDate: timestamp('chase_date').notNull().defaultNow(),
  status: varchar('chase_status', { length: 20 }).notNull(), // sent, failed, bounced
  emailSubject: varchar('email_subject', { length: 255 }),
  messageId: varchar('message_id', { length: 255 }),
  errorMessage: text('error_message'),
  responseReceived: boolean('response_received').notNull().default(false),
  responseDate: timestamp('response_date'),
});
```

### Chase Email Template

```typescript
// src/lib/templates/chase-email.ts
export const chaseEmailTemplate = {
  subject: 'Reminder: Payroll data required by {{cutOffDate}} - {{employerName}}',
  html: `
    <p>Dear {{contactName}},</p>
    <p>This is a reminder that we need your payroll data by <strong>{{cutOffDate}}</strong> to ensure your employees are paid on time.</p>
    <p>Please submit the following via the client portal:</p>
    <ul>
      <li>Timesheets</li>
      <li>Variable pay (bonuses, commissions)</li>
      <li>New starter/leaver information</li>
      <li>Any other payroll changes</li>
    </ul>
    <p><a href="{{portalLink}}" style="...">Submit Payroll Data</a></p>
    <p>If you have any questions, please contact us.</p>
    <p>Best regards,<br>{{bureauName}}</p>
  `,
};
```

---

## Business Rules & Invariants

1. Job runs at 9am weekdays only (no weekend chasing)
2. Only chase clients where payroll data is actually missing
3. Respect client timezone for "business hours" calculation
4. Failed sends are retried up to 3 times
5. Bounced emails trigger exception creation
6. Chase log entry created for every attempt

---

## Edge Cases

1. **Email service down** — Queue for retry, alert ops team
2. **Invalid email address** — Log error, create exception
3. **Client already responded** — Don't chase, mark complete
4. **Multiple pay periods** — Send single consolidated chase

---

## Tests

### chase-job.test.ts

- Job finds clients approaching cut-off
- Correct template variables populated
- Email sent via configured service
- Failed sends are logged with error
- Already responded clients not chased

---

## Verification

```bash
npm run test:unit -- chase-job.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 3: Automated Client Chasing
- 02-04-bureau-operations-spec.md § Integration Points → Email Service
