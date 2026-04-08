# Slice c: Batch Reminders and Data Exports

**Story:** story-08-batch-ops
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement batch reminder sending and data export functionality allowing bureau staff to chase multiple clients simultaneously and export client data for external use.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, Zod 3.22.x, CSV stringify 6.x, XLSX 0.18.x
- [x] All SDK methods/API calls identified: tRPC batch.sendReminders, tRPC batch.export
- [x] All external service endpoints specified: tRPC batch mutations
- [x] All data contracts defined: BatchSendRemindersInput, BatchExportInput
- [x] All configuration/environment variables listed: BATCH_EXPORT_MAX_ITEMS (default: 1000)
- [x] All error scenarios identified with handling strategy: Email failures, export generation
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § API Contracts — POST /api/v1/employers/{id}/send-reminder
- 02-04-bureau-operations-spec.md § Non-Functional Requirements — Bulk actions

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/batch.ts` | update | Add reminder and export procedures |
| `src/lib/services/batch-reminder-service.ts` | create | Batch reminder processing |
| `src/lib/services/export-service.ts` | create | Export generation service |
| `src/components/batch/bulk-reminder-dialog.tsx` | create | Batch reminder UI |
| `src/components/batch/export-dialog.tsx` | create | Export dialog |
| `src/tests/server/batch-reminders.test.ts` | create | Reminder tests |

---

## Responsibilities

1. Implement batch.sendReminders to send chase emails to multiple clients
2. Implement batch.export to generate CSV/Excel with client data
3. Support custom reminder message for batch sends
4. Support column selection for exports
5. Handle email delivery failures gracefully
6. Generate downloadable export files

---

## Contracts

### batch.sendReminders

- **Method:** tRPC mutation `batch.sendReminders`
- **Input:** BatchSendRemindersInput

```typescript
export const BatchSendRemindersInput = z.object({
  employerIds: z.array(z.string().uuid()).max(100),
  templateId: z.string().uuid(),
  customMessage: z.string().optional(),
  subject: z.string().optional(),
});
```

- **Output:** BatchOperationResult (with email-specific fields)

```typescript
export const BatchReminderResult = z.object({
  total: z.number().int(),
  sent: z.number().int(),
  failed: z.number().int(),
  skipped: z.number().int(), // Already responded, etc.
  failures: z.array(z.object({
    employerId: z.string().uuid(),
    employerName: z.string(),
    error: z.string(),
  })),
});
```

- **Errors:** BAD_REQUEST, UNAUTHORIZED
- **Auth:** protectedProcedure with batch:remind permission

### batch.export

- **Method:** tRPC mutation `batch.export`
- **Input:** BatchExportInput

```typescript
export const BatchExportInput = z.object({
  employerIds: z.array(z.string().uuid()).max(1000),
  format: z.enum(['csv', 'xlsx']),
  columns: z.array(z.enum([
    'employer_name',
    'paye_scheme',
    'current_status',
    'pay_date',
    'cut_off_date',
    'assigned_processor',
    'has_exceptions',
    'days_overdue',
  ])),
});
```

- **Output:** ExportResult

```typescript
export const ExportResult = z.object({
  downloadUrl: z.string().url(),
  fileName: z.string(),
  recordCount: z.number().int(),
  expiresAt: z.string().datetime(), // URL expiry
});
```

- **Errors:** BAD_REQUEST, UNAUTHORIZED
- **Auth:** protectedProcedure with batch:export permission

### BatchReminderService

```typescript
// src/lib/services/batch-reminder-service.ts
export class BatchReminderService {
  async sendBatchReminders(
    bureauId: string,
    performedBy: string,
    input: BatchSendRemindersInput
  ): Promise<BatchReminderResult> {
    // Load template
    // For each employer:
    //   - Check if already responded
    //   - Personalize message
    //   - Send email
    //   - Log communication
    // Return results
  }
  
  private async shouldSkipReminder(employerId: string): Promise<boolean> {
    // Check if client already responded this period
    // Check if reminder sent within last 24 hours
  }
}
```

### ExportService

```typescript
// src/lib/services/export-service.ts
export class ExportService {
  async generateExport(
    bureauId: string,
    input: BatchExportInput
  ): Promise<ExportResult> {
    // Query client data
    // Transform to export format
    // Generate CSV or XLSX
    // Upload to temporary storage
    // Return download URL
  }
  
  private generateCSV(data: unknown[]): string {
    // Use csv-stringify
  }
  
  private generateXLSX(data: unknown[]): Buffer {
    // Use xlsx library
  }
  
  private transformForExport(
    clients: ClientPayrollStatus[],
    columns: ExportColumn[]
  ): unknown[];
}
```

### BulkReminderDialog Component

```typescript
// src/components/batch/bulk-reminder-dialog.tsx
interface BulkReminderDialogProps {
  employerIds: string[];
  employerNames: string[];
  isOpen: boolean;
  onClose: () => void;
  onComplete: (result: BatchReminderResult) => void;
}

// Shows:
// - Template selection
// - Custom message textarea (optional)
// - Subject line edit
// - Preview
// - Send button
export function BulkReminderDialog(props: BulkReminderDialogProps): JSX.Element;
```

### ExportDialog Component

```typescript
// src/components/batch/export-dialog.tsx
interface ExportDialogProps {
  employerIds: string[];
  isOpen: boolean;
  onClose: () => void;
  onExport: (result: ExportResult) => void;
}

// Shows:
// - Format selection (CSV, Excel)
// - Column checklist
// - Record count
// - Export button
// - Download link on complete
export function ExportDialog(props: ExportDialogProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Reminders skipped if client already responded
2. Rate limit: max 1 reminder per client per 24 hours
3. Export files available for 24 hours
4. CSV uses UTF-8 encoding with BOM
5. Excel exports include headers and formatting
6. Export includes only selected columns

---

## Edge Cases

1. **All clients already responded** — Show info, no emails sent
2. **Export too large** — Paginate into multiple files
3. **Email service unavailable** — Queue for retry
4. **Invalid template variables** — Show preview error

---

## Tests

### batch-reminders.test.ts

- Send reminders to multiple clients
- Skips clients who already responded
- Custom message included in emails
- Export generates valid CSV
- Export generates valid Excel
- Download URL expires after 24 hours

---

## Verification

```bash
npm run test:unit -- batch-reminders.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § API Contracts → Manual reminder endpoint
