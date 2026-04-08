# Slice d: Progress Tracking and Error Handling

**Story:** story-08-batch-ops
**Epic:** epic-04-bureau-operations
**Effort:** S
**Dependencies:** slice-b, slice-c

---

## Goal

Implement progress tracking for long-running batch operations and comprehensive error handling for partial failures. Provide clear feedback on operation status and allow retry of failed items.

---

## Decision Checklist

- [x] All libraries/packages named: Radix UI Progress 1.x, React 18, @tanstack/react-query 5.x
- [x] All SDK methods/API calls identified: Polling pattern, batch status queries
- [x] All external service endpoints specified: tRPC batch.getStatus
- [x] All data contracts defined: BatchOperationStatus, BatchProgress
- [x] All configuration/environment variables listed: BATCH_POLL_INTERVAL (default: 1000ms)
- [x] All error scenarios identified with handling strategy: Retry logic, compensation
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Non-Functional Requirements — Bulk actions UX
- 08-architecture-and-patterns.md — Error handling patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/batch/batch-progress-dialog.tsx` | create | Progress modal |
| `src/components/batch/batch-results-dialog.tsx` | create | Results display |
| `src/components/batch/retry-failed-button.tsx` | create | Retry action |
| `src/components/batch/failure-list.tsx` | create | Failed items list |
| `src/hooks/use-batch-operation.ts` | create | Batch operation hook |
| `src/stores/batch-operation-store.ts` | create | Operation state store |
| `src/tests/components/batch/progress.test.tsx` | create | Progress tests |

---

## Responsibilities

1. Create progress dialog showing operation status
2. Implement polling for long-running operations
3. Display results with success/failure counts
4. Allow retry of failed items
5. Show detailed error messages for failures
6. Support cancellation of in-progress operations

---

## Contracts

### BatchOperationStatus

```typescript
// Types for batch operation tracking
export type BatchOperationState = 
  | 'pending'
  | 'in_progress'
  | 'completed'
  | 'partial_failure'
  | 'failed'
  | 'cancelled';

export interface BatchOperationStatus {
  operationId: string;
  state: BatchOperationState;
  total: number;
  processed: number;
  succeeded: number;
  failed: number;
  startedAt: Date;
  completedAt?: Date;
  estimatedCompletion?: Date;
  failures: BatchFailure[];
}

export interface BatchFailure {
  employerId: string;
  employerName: string;
  error: string;
  code: string;
  retryable: boolean;
}
```

### useBatchOperation Hook

```typescript
// src/hooks/use-batch-operation.ts
interface UseBatchOperationReturn {
  status: BatchOperationStatus | null;
  isLoading: boolean;
  error: Error | null;
  startOperation: (params: BatchOperationParams) => Promise<void>;
  cancelOperation: () => void;
  retryFailed: () => Promise<void>;
}

interface BatchOperationParams {
  type: 'reassign' | 'updateStatus' | 'sendReminders' | 'export';
  input: unknown;
}

export function useBatchOperation(): UseBatchOperationReturn {
  // Manages operation lifecycle
  // Polls for status updates
  // Handles retry logic
}
```

### BatchProgressDialog Component

```typescript
// src/components/batch/batch-progress-dialog.tsx
interface BatchProgressDialogProps {
  isOpen: boolean;
  status: BatchOperationStatus;
  onCancel: () => void;
}

// Shows:
// - Progress bar with percentage
// - "X of Y processed"
// - Estimated time remaining
// - Cancel button (if in_progress)
// - Real-time updates
export function BatchProgressDialog(props: BatchProgressDialogProps): JSX.Element;
```

### BatchResultsDialog Component

```typescript
// src/components/batch/batch-results-dialog.tsx
interface BatchResultsDialogProps {
  isOpen: boolean;
  onClose: () => void;
  result: BatchOperationResult;
  onRetry?: () => void;
  onExportFailures?: () => void;
}

// Shows:
// - Success count with checkmark
// - Failure count with warning
// - Expandable failure list
// - Retry failed button
// - Export failures to CSV
export function BatchResultsDialog(props: BatchResultsDialogProps): JSX.Element;
```

### FailureList Component

```typescript
// src/components/batch/failure-list.tsx
interface FailureListProps {
  failures: BatchFailure[];
  onRetryItem?: (employerId: string) => void;
}

// Shows:
// - Table of failed items
// - Client name
// - Error message
// - Error code
// - Retry button per item (if retryable)
export function FailureList({ failures, onRetryItem }: FailureListProps): JSX.Element;
```

### RetryFailedButton Component

```typescript
// src/components/batch/retry-failed-button.tsx
interface RetryFailedButtonProps {
  failedIds: string[];
  operationType: string;
  originalInput: unknown;
  onRetryComplete: (result: BatchOperationResult) => void;
}

// Retries only failed items
// Shows progress of retry
// Merges results with original
export function RetryFailedButton(props: RetryFailedButtonProps): JSX.Element;
```

### BatchOperationStore (Zustand)

```typescript
// src/stores/batch-operation-store.ts
interface BatchOperationStore {
  activeOperations: Map<string, BatchOperationStatus>;
  
  addOperation: (status: BatchOperationStatus) => void;
  updateOperation: (operationId: string, update: Partial<BatchOperationStatus>) => void;
  removeOperation: (operationId: string) => void;
  getOperation: (operationId: string) => BatchOperationStatus | undefined;
}

export const useBatchOperationStore = create<BatchOperationStore>((set, get) => ({
  activeOperations: new Map(),
  
  addOperation: (status) => set((state) => {
    const newMap = new Map(state.activeOperations);
    newMap.set(status.operationId, status);
    return { activeOperations: newMap };
  }),
  
  updateOperation: (id, update) => set((state) => {
    const existing = state.activeOperations.get(id);
    if (!existing) return state;
    const newMap = new Map(state.activeOperations);
    newMap.set(id, { ...existing, ...update });
    return { activeOperations: newMap };
  }),
  
  removeOperation: (id) => set((state) => {
    const newMap = new Map(state.activeOperations);
    newMap.delete(id);
    return { activeOperations: newMap };
  }),
  
  getOperation: (id) => get().activeOperations.get(id),
}));
```

---

## Business Rules & Invariants

1. Progress updates every 1 second during operation
2. Operations can be cancelled before completion
3. Failed items can be retried individually or in batch
4. Error messages include actionable guidance
5. Non-retryable errors (validation, permissions) shown immediately
6. Operation audit log retained for 90 days

---

## Edge Cases

1. **All items fail** — Show error state, offer full retry
2. **Operation hangs** — Timeout after 5 minutes, show error
3. **Cancel during processing** — Graceful stop, report partial results
4. **Browser refresh** — Restore operation state from server

---

## Tests

### progress.test.tsx

- Progress bar updates with percentage
- Cancel button stops operation
- Results dialog shows correct counts
- Failure list displays error details
- Retry failed items works correctly

---

## Verification

```bash
npm run test:unit -- progress.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Non-Functional Requirements → Bulk actions UX
