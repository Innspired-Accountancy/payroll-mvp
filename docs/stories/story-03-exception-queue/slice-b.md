# Slice b: Automatic Exception Creation from Events

**Story:** story-03-exception-queue
**Epic:** epic-04-bureau-operations
**Effort:** L
**Dependencies:** slice-a

---

## Goal

Implement automatic exception creation from system events including HMRC submission failures, pension submission failures, payment failures, and missing data scenarios. Subscribe to relevant event streams and create appropriate exceptions.

---

## Decision Checklist

- [x] All libraries/packages named: Node.js EventEmitter, Drizzle ORM 0.30.x, date-fns 3.x
- [x] All SDK methods/API calls identified: EventEmitter.on(), exceptionService.createFromEvent()
- [x] All external service endpoints specified: Internal event bus only
- [x] All data contracts defined: HMRCFailureEvent, PensionFailureEvent, PaymentFailureEvent, MissingDataEvent
- [x] All configuration/environment variables listed: EXCEPTION_AUTO_CREATE_ENABLED (default: true)
- [x] All error scenarios identified with handling strategy: Event parsing failures, duplicate detection
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Integration Points — HMRC Submissions, Pension Module, Payments
- 02-04-bureau-operations-spec.md § User Journeys — Journey 2: Exception Queue Management

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/events/exception-handlers.ts` | create | Event handler implementations |
| `src/lib/services/auto-exception-service.ts` | create | Auto-creation business logic |
| `src/lib/rules/exception-rules.ts` | create | Rules for when to create exceptions |
| `src/jobs/missing-data-checker.ts` | create | Scheduled job for missing data detection |
| `src/tests/events/exception-handlers.test.ts` | create | Event handler unit tests |

---

## Responsibilities

1. Subscribe to HMRC submission failure events and create exceptions
2. Subscribe to pension submission failure events and create exceptions
3. Subscribe to payment failure events and create exceptions
4. Implement scheduled job to detect missing payroll data approaching deadlines
5. Implement duplicate detection to prevent exception flooding
6. Set appropriate severity based on failure type and timing

---

## Contracts

### Event Types

```typescript
// Event payloads from other modules
interface HMRCSubmissionFailedEvent {
  type: 'hmrc.submission_failed';
  payload: {
    employerId: string;
    bureauId: string;
    submissionId: string;
    submissionType: 'FPS' | 'EPS';
    errorCode: string;
    errorMessage: string;
    timestamp: Date;
  };
}

interface PensionSubmissionFailedEvent {
  type: 'pension.submission_failed';
  payload: {
    employerId: string;
    bureauId: string;
    pensionProvider: string;
    submissionId: string;
    errorCode: string;
    errorMessage: string;
    timestamp: Date;
  };
}

interface PaymentFailedEvent {
  type: 'payment.failed';
  payload: {
    employerId: string;
    bureauId: string;
    paymentId: string;
    paymentType: 'bacs' | ' FasterPayments';
    failureReason: string;
    amount: Decimal;
    timestamp: Date;
  };
}

interface PayrollDataMissingEvent {
  type: 'payroll.data_missing';
  payload: {
    employerId: string;
    bureauId: string;
    payPeriodId: string;
    missingFields: string[];
    daysUntilCutOff: number;
  };
}
```

### AutoExceptionService

```typescript
// src/lib/services/auto-exception-service.ts
export class AutoExceptionService {
  async handleHMRCFailure(event: HMRCSubmissionFailedEvent): Promise<void>;
  async handlePensionFailure(event: PensionSubmissionFailedEvent): Promise<void>;
  async handlePaymentFailure(event: PaymentFailedEvent): Promise<void>;
  async handleMissingData(event: PayrollDataMissingEvent): Promise<void>;
  
  private async createException(data: CreateExceptionData): Promise<void>;
  private async shouldCreateException(employerId: string, type: string): Promise<boolean>;
  private determineSeverity(type: string, context: unknown): ExceptionSeverity;
}
```

### Severity Rules

```typescript
// src/lib/rules/exception-rules.ts
export const severityRules = {
  hmrc_failure: (event: HMRCSubmissionFailedEvent): ExceptionSeverity => {
    if (event.payload.errorCode.startsWith('E')) return 'critical';
    if (event.payload.submissionType === 'FPS') return 'high';
    return 'medium';
  },
  
  pension_failure: (event: PensionSubmissionFailedEvent): ExceptionSeverity => {
    return 'high'; // Pension failures are always high priority
  },
  
  payment_failure: (event: PaymentFailedEvent): ExceptionSeverity => {
    return 'critical'; // Payment failures are critical
  },
  
  missing_data: (event: PayrollDataMissingEvent): ExceptionSeverity => {
    if (event.payload.daysUntilCutOff <= 1) return 'critical';
    if (event.payload.daysUntilCutOff <= 3) return 'high';
    if (event.payload.daysUntilCutOff <= 5) return 'medium';
    return 'low';
  },
};
```

### Missing Data Checker Job

```typescript
// src/jobs/missing-data-checker.ts
// Runs every 4 hours via node-cron or similar
export async function checkMissingData(): Promise<void> {
  // Find pay periods with cut-off within next 7 days
  // where status is not_started or data_collection
  // and required data is missing
  // Create missing_data exception for each
}
```

---

## Business Rules & Invariants

1. Duplicate detection: Don't create exception if open exception exists for same employer/type
2. HMRC failures: Critical if error code starts with 'E', High for FPS, Medium for EPS
3. Pension failures: Always High severity
4. Payment failures: Always Critical severity
5. Missing data severity based on days until cut-off
6. All auto-created exceptions have sourceModule set to originating module

---

## Edge Cases

1. **Rapid repeated failures** — Rate limit exception creation, batch similar errors
2. **Employer not found** — Log error, skip exception creation
3. **Exception creation fails** — Log to error tracking, alert ops team
4. **Job runs during maintenance** — Graceful degradation, resume on next run

---

## Tests

### exception-handlers.test.ts

- HMRC failure creates exception with correct severity
- Duplicate HMRC failure doesn't create second exception
- Pension failure creates high severity exception
- Missing data job creates exceptions for approaching deadlines
- Severity escalates as cut-off approaches
- Failed exception creation logs error

---

## Verification

```bash
npm run test:unit -- exception-handlers.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Integration Points → External system event handling
- 02-04-bureau-operations-spec.md § User Journeys → Exception Queue Management
