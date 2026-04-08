# Slice b: Bulk Reassign and Status Update

**Story:** story-08-batch-ops
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-a

---

## Goal

Implement bulk reassign and status update operations allowing bureau staff to change assigned processors and payroll status for multiple clients simultaneously.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, Zod 3.22.x, Radix UI Dialog 1.x, @tanstack/react-query 5.x
- [x] All SDK methods/API calls identified: tRPC batch.reassign, tRPC batch.updateStatus
- [x] All external service endpoints specified: tRPC batch mutations
- [x] All data contracts defined: BatchReassignInput, BatchUpdateStatusInput
- [x] All configuration/environment variables listed: BATCH_OPERATION_MAX_ITEMS (default: 100)
- [x] All error scenarios identified with handling strategy: Partial failures, validation
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 1: Reassign overdue payroll
- 02-04-bureau-operations-spec.md § API Contracts — Batch operations

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/batch.ts` | create | tRPC router for batch operations |
| `src/lib/services/batch-service.ts` | create | Batch processing service |
| `src/components/batch/bulk-reassign-dialog.tsx` | create | Reassign dialog |
| `src/components/batch/bulk-status-dialog.tsx` | create | Status update dialog |
| `src/components/batch/batch-confirmation.tsx` | create | Confirmation dialog |
| `src/lib/validation/batch.ts` | create | Zod schemas |
| `src/tests/server/batch-router.test.ts` | create | Router tests |

---

## Responsibilities

1. Implement batch.reassign to change processor for multiple clients
2. Implement batch.updateStatus to change payroll status for multiple clients
3. Create confirmation dialogs showing action summary
4. Handle partial failures (some succeed, some fail)
5. Log all batch operations for audit
6. Update UI optimistically, confirm on success

---

## Contracts

### batch.reassign

- **Method:** tRPC mutation `batch.reassign`
- **Input:** BatchReassignInput

```typescript
export const BatchReassignInput = z.object({
  employerIds: z.array(z.string().uuid()).max(100),
  assignedProcessorId: z.string().uuid(),
});
```

- **Output:** BatchOperationResult

```typescript
export const BatchOperationResult = z.object({
  total: z.number().int(),
  succeeded: z.number().int(),
  failed: z.number().int(),
  failures: z.array(z.object({
    employerId: z.string().uuid(),
    employerName: z.string(),
    error: z.string(),
    code: z.string(),
  })),
});
```

- **Errors:** BAD_REQUEST (validation), UNAUTHORIZED
- **Auth:** protectedProcedure with batch:reassign permission

### batch.updateStatus

- **Method:** tRPC mutation `batch.updateStatus`
- **Input:** BatchUpdateStatusInput

```typescript
export const BatchUpdateStatusInput = z.object({
  employerIds: z.array(z.string().uuid()).max(100),
  status: z.enum(['not_started', 'data_collection', 'calculated', 'in_review', 'approved', 'submitted', 'paid', 'complete']),
  note: z.string().optional(),
});
```

- **Output:** BatchOperationResult
- **Errors:** BAD_REQUEST, UNAUTHORIZED
- **Auth:** protectedProcedure with batch:update permission

### BatchService

```typescript
// src/lib/services/batch-service.ts
export class BatchService {
  async bulkReassign(
    bureauId: string,
    performedBy: string,
    input: BatchReassignInput
  ): Promise<BatchOperationResult> {
    // Validate all employerIds belong to bureau
    // Update in transaction batches (10 at a time)
    // Log operation to batch_audit_log
    // Return results
  }
  
  async bulkUpdateStatus(
    bureauId: string,
    performedBy: string,
    input: BatchUpdateStatusInput
  ): Promise<BatchOperationResult> {
    // Validate status transitions
    // Update in transaction batches
    // Create audit entries for each change
    // Return results
  }
  
  private async processInBatches<T, R>(
    items: T[],
    batchSize: number,
    processor: (batch: T[]) => Promise<R[]>
  ): Promise<R[]>;
}
```

### BulkReassignDialog Component

```typescript
// src/components/batch/bulk-reassign-dialog.tsx
interface BulkReassignDialogProps {
  employerIds: string[];
  employerNames: string[];
  isOpen: boolean;
  onClose: () => void;
  onComplete: (result: BatchOperationResult) => void;
}

// Shows:
// - "Reassign X clients" header
// - Processor dropdown (with current assignments shown)
// - Confirmation of action
// - Result display (success/failure count)
export function BulkReassignDialog(props: BulkReassignDialogProps): JSX.Element;
```

### BulkStatusDialog Component

```typescript
// src/components/batch/bulk-status-dialog.tsx
interface BulkStatusDialogProps {
  employerIds: string[];
  employerNames: string[];
  isOpen: boolean;
  onClose: () => void;
  onComplete: (result: BatchOperationResult) => void;
}

// Shows:
// - "Update status for X clients" header
// - Status dropdown
// - Optional note field
// - Warning about bulk status changes
export function BulkStatusDialog(props: BulkStatusDialogProps): JSX.Element;
```

### BatchConfirmation Component

```typescript
// src/components/batch/batch-confirmation.tsx
interface BatchConfirmationProps {
  title: string;
  description: string;
  itemCount: number;
  itemNames: string[]; // First few names
  actionLabel: string;
  onConfirm: () => void;
  onCancel: () => void;
}

// Shows:
// - Warning icon for destructive actions
// - Summary of action
// - "This will affect X clients: Name1, Name2, Name3..."
// - Confirm/Cancel buttons
export function BatchConfirmation(props: BatchConfirmationProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Maximum 100 clients per batch operation
2. All employerIds must belong to user's bureau
3. Partial failures reported individually
4. Audit log entry created for each affected client
5. Optimistic UI updates, rolled back on failure
6. Status transitions validated (cannot go backwards without override)

---

## Edge Cases

1. **All fail** — Show error, no optimistic update committed
2. **Some fail** — Show partial success, list failures
3. **Invalid processor** — Validation error before processing
4. **Concurrent modification** — Optimistic locking prevents conflicts

---

## Tests

### batch-router.test.ts

- Reassign multiple clients successfully
- Status update applies to all selected
- Partial failure returns correct counts
- Unauthorized user rejected
- Over limit (100+) returns error
- Audit log entries created

---

## Verification

```bash
npm run test:unit -- batch-router.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 1: Manager reassigns overdue payroll
