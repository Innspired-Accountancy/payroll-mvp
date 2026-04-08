# Slice d: Assignment and Resolution Workflow

**Story:** story-03-exception-queue
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** slice-c

---

## Goal

Implement the exception assignment and resolution workflow allowing bureau staff to assign exceptions to team members, add notes, and mark exceptions as resolved with documentation.

---

## Decision Checklist

- [x] All libraries/packages named: Radix UI Dialog 1.x, @tanstack/react-query 5.x, React Hook Form 7.x, Zod 3.22.x
- [x] All SDK methods/API calls identified: tRPC exceptions.assign, tRPC exceptions.resolve, tRPC users.list
- [x] All external service endpoints specified: tRPC mutations for assign and resolve
- [x] All data contracts defined: ExceptionAssignInput, ExceptionResolveInput
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Validation errors, conflicts
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Journey 2: Assign to team member or resolve directly
- 02-04-bureau-operations-spec.md § API Contracts — POST /api/v1/exceptions/{id}/assign

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/exceptions/exception-detail-modal.tsx` | create | Exception detail view modal |
| `src/components/exceptions/assign-exception-dialog.tsx` | create | Assignment dialog |
| `src/components/exceptions/resolve-exception-dialog.tsx` | create | Resolution dialog |
| `src/components/exceptions/exception-notes.tsx` | create | Notes display component |
| `src/components/exceptions/assignee-selector.tsx` | create | User selection dropdown |
| `src/app/(bureau)/exceptions/[id]/page.tsx` | create | Exception detail page |
| `src/tests/components/exceptions/assignment-workflow.test.tsx` | create | Workflow tests |

---

## Responsibilities

1. Create exception detail view with full information
2. Implement assign dialog with user selection
3. Implement resolve dialog with notes requirement
4. Show exception history and audit trail
5. Provide quick actions from table (assign to me, resolve)
6. Update dashboard counts on assignment/resolution

---

## Contracts

### ExceptionDetailModal Component

```typescript
// src/components/exceptions/exception-detail-modal.tsx
interface ExceptionDetailModalProps {
  exceptionId: string;
  isOpen: boolean;
  onClose: () => void;
  onAssign: () => void;
  onResolve: () => void;
}

// Uses: api.exceptions.getById.useQuery(exceptionId)
export function ExceptionDetailModal(props: ExceptionDetailModalProps): JSX.Element;
```

### AssignExceptionDialog Component

```typescript
// src/components/exceptions/assign-exception-dialog.tsx
interface AssignExceptionDialogProps {
  exceptionId: string;
  currentAssignee?: { id: string; name: string };
  isOpen: boolean;
  onClose: () => void;
  onAssigned: () => void;
}

// Uses: 
// - api.users.list.useQuery({ bureauId, role: 'processor' })
// - api.exceptions.assign.useMutation()
export function AssignExceptionDialog(props: AssignExceptionDialogProps): JSX.Element;
```

### ResolveExceptionDialog Component

```typescript
// src/components/exceptions/resolve-exception-dialog.tsx
interface ResolveExceptionDialogProps {
  exceptionId: string;
  isOpen: boolean;
  onClose: () => void;
  onResolved: () => void;
}

// Form schema:
const resolveSchema = z.object({
  resolutionNotes: z.string().min(10, 'Please provide detailed resolution notes'),
});

// Uses: api.exceptions.resolve.useMutation()
export function ResolveExceptionDialog(props: ResolveExceptionDialogProps): JSX.Element;
```

### AssigneeSelector Component

```typescript
// src/components/exceptions/assignee-selector.tsx
interface AssigneeSelectorProps {
  bureauId: string;
  value?: string;
  onChange: (userId: string) => void;
  exclude?: string[];
}

// Uses: api.users.list.useQuery({ bureauId })
export function AssigneeSelector(props: AssigneeSelectorProps): JSX.Element;
```

### QuickActionButtons Component

```typescript
// src/components/exceptions/quick-action-buttons.tsx
interface QuickActionButtonsProps {
  exceptionId: string;
  status: ExceptionStatus;
  assignedTo?: string;
  currentUserId: string;
  onAssignToMe: () => void;
  onResolve: () => void;
  onReassign: () => void;
}

// Shows:
// - "Assign to Me" if unassigned or assigned to other
// - "Resolve" if assigned to current user
// - "Reassign" always visible
export function QuickActionButtons(props: QuickActionButtonsProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Resolution notes minimum 10 characters required
2. Assignment updates status from 'open' to 'assigned'
3. Re-assignment allowed without status change
4. Only assigned user or admin can resolve
5. Resolved exceptions cannot be re-opened (create new if needed)
6. Assignment triggers notification to assignee

---

## Edge Cases

1. **Self-assignment** — Quick button "Assign to Me" available
2. **Already resolved** — Dialog shows error, disable actions
3. **User leaves bureau** — Exceptions reassigned to manager
4. **Concurrent assignment** — Last write wins, notify first assignee

---

## Tests

### assignment-workflow.test.tsx

- Assign dialog shows user list
- Assignment updates exception status
- Resolve requires notes of minimum length
- Cannot resolve already resolved exception
- "Assign to Me" button assigns to current user
- Resolution notes displayed in detail view

---

## Verification

```bash
npm run test:unit -- assignment-workflow.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Journey 2: Exception Queue Management
- 02-04-bureau-operations-spec.md § API Contracts → Exception assignment endpoint
