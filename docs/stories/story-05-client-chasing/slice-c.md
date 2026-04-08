# Slice c: Escalation and Dashboard Integration

**Story:** story-05-client-chasing
**Epic:** epic-04-bureau-operations
**Effort:** S
**Dependencies:** slice-b

---

## Goal

Implement the escalation workflow when clients don't respond to reminders and integrate chasing status into the dashboard. Create exceptions for non-responsive clients and provide manual chase capabilities.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, @tanstack/react-query 5.x, Radix UI 1.x
- [x] All SDK methods/API calls identified: exceptionService.create(), dashboard.refresh()
- [x] All external service endpoints specified: tRPC chasing.sendManual
- [x] All data contracts defined: EscalationEvent, ManualChaseInput
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Escalation failures
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 3: System escalates to bureau team
- 02-04-bureau-operations-spec.md § API Contracts — POST /api/v1/employers/{id}/send-reminder

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/services/escalation-service.ts` | create | Escalation logic and exception creation |
| `src/components/chasing/chase-status-widget.tsx` | create | Dashboard chase status display |
| `src/components/chasing/manual-chase-dialog.tsx` | create | Manual chase UI |
| `src/server/routers/chasing.ts` | create | tRPC router for manual chasing |
| `src/tests/services/escalation-service.test.ts` | create | Escalation tests |

---

## Responsibilities

1. Create escalation service that generates exceptions for non-responsive clients
2. Add chase status indicator to dashboard client list
3. Implement manual chase dialog for ad-hoc reminders
4. Update ClientPayrollStatus with chase tracking fields
5. Notify assigned processor of escalations

---

## Contracts

### EscalationService

```typescript
// src/lib/services/escalation-service.ts
export class EscalationService {
  async escalateNonResponsiveClient(
    employerId: string,
    payPeriodId: string,
    lastChaseDate: Date
  ): Promise<void>;
  
  private async createEscalationException(
    employerId: string,
    details: EscalationDetails
  ): Promise<void>;
  
  async checkAndEscalate(date: Date): Promise<EscalationResult[]>;
}

interface EscalationDetails {
  payPeriodId: string;
  chaseCount: number;
  lastChaseDate: Date;
  daysSinceChase: number;
}

interface EscalationResult {
  employerId: string;
  escalated: boolean;
  exceptionId?: string;
  reason?: string;
}
```

### ChaseStatusWidget Component

```typescript
// src/components/chasing/chase-status-widget.tsx
interface ChaseStatusWidgetProps {
  employerId: string;
}

// Displays:
// - Last chased date
// - Chase count this period
// - Response received status
// - Days until escalation
export function ChaseStatusWidget({ employerId }: ChaseStatusWidgetProps): JSX.Element;
```

### ManualChaseDialog Component

```typescript
// src/components/chasing/manual-chase-dialog.tsx
interface ManualChaseDialogProps {
  employerId: string;
  isOpen: boolean;
  onClose: () => void;
  onSent: () => void;
}

// Form:
// - Recipient email (prefilled, editable)
// - Subject (prefilled from template, editable)
// - Message (template with placeholders)
// - Preview before send
export function ManualChaseDialog(props: ManualChaseDialogProps): JSX.Element;
```

### chasing.sendManual

- **Method:** tRPC mutation `chasing.sendManual`
- **Input:** ManualChaseInput

```typescript
export const ManualChaseInput = z.object({
  employerId: z.string().uuid(),
  to: z.string().email(),
  subject: z.string().min(1),
  message: z.string().min(1),
  payPeriodId: z.string().uuid().optional(),
});
```

- **Output:** { sent: boolean; messageId: string }
- **Errors:** BAD_REQUEST, NOT_FOUND, UNAUTHORIZED
- **Auth:** protectedProcedure with chasing:send permission

### ClientPayrollStatus Update

```typescript
// Add to existing schema:
export const clientPayrollStatus = pgTable('client_payroll_status', {
  // ... existing fields
  lastChaseDate: timestamp('last_chase_date'),
  chaseCount: integer('chase_count').default(0),
  responseReceived: boolean('response_received').default(false),
  escalationTriggered: boolean('escalation_triggered').default(false),
});
```

---

## Business Rules & Invariants

1. Escalation creates high severity 'missing_data' exception
2. Exception assigned to client's default processor
3. Manual chase uses same template but allows customization
4. Response received resets chase count for next period
5. Escalation only happens once per pay period
6. Dashboard shows chase status in real-time

---

## Edge Cases

1. **Client responds after escalation** — Mark exception resolved, notify processor
2. **Manual chase during auto-chase** — Both allowed, track separately
3. **No processor assigned** — Escalate to bureau manager
4. **Multiple chases same day** — Rate limit to prevent spam

---

## Tests

### escalation-service.test.ts

- Non-responsive client escalated after threshold
- Exception created with correct severity and assignment
- Already escalated client not re-escalated
- Manual chase sends custom email
- Response received resets chase state

---

## Verification

```bash
npm run test:unit -- escalation-service.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 3: System escalates to bureau team
- 02-04-bureau-operations-spec.md § API Contracts → Manual reminder endpoint
