# Slice a: Batch Selection and UI Framework

**Story:** story-08-batch-ops
**Epic:** epic-04-bureau-operations
**Effort:** M
**Dependencies:** story-02-dashboard-ui

---

## Goal

Create the batch selection UI framework enabling multi-select of clients with select-all capability, batch action toolbar, and selection state management.

---

## Decision Checklist

- [x] All libraries/packages named: @tanstack/react-table 8.x, Zustand 4.x, Radix UI 1.x, Tailwind CSS 3.4
- [x] All SDK methods/API calls identified: React Table row selection API
- [x] All external service endpoints specified: N/A (UI layer)
- [x] All data contracts defined: SelectionState, BatchAction types
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Selection limits
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Non-Functional Requirements — Bulk actions
- 08-architecture-and-patterns.md — State management patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/batch/batch-selection-provider.tsx` | create | Selection state context |
| `src/components/batch/batch-toolbar.tsx` | create | Batch action toolbar |
| `src/components/batch/select-all-header.tsx` | create | Table header with select-all |
| `src/components/batch/selection-count-badge.tsx` | create | Selection indicator |
| `src/hooks/use-batch-selection.ts` | create | Selection state hook |
| `src/stores/batch-selection-store.ts` | create | Zustand store |
| `src/tests/components/batch/selection.test.tsx` | create | Selection tests |

---

## Responsibilities

1. Implement row-level checkboxes for client selection
2. Implement select-all functionality (page only and all results)
3. Create batch action toolbar showing selected count and actions
4. Manage selection state with Zustand
5. Support selection persistence across pagination
6. Enforce selection limits (max 100 items)

---

## Contracts

### BatchSelectionStore (Zustand)

```typescript
// src/stores/batch-selection-store.ts
interface BatchSelectionState {
  selectedIds: Set<string>;
  isAllSelected: boolean;
  totalAvailable: number;
  
  selectId: (id: string) => void;
  deselectId: (id: string) => void;
  toggleId: (id: string) => void;
  selectAll: (ids: string[]) => void;
  selectAllPages: (total: number) => void;
  clearSelection: () => void;
  setTotalAvailable: (total: number) => void;
  
  getSelectedCount: () => number;
  isSelected: (id: string) => boolean;
  getSelectedIds: () => string[];
}

export const useBatchSelectionStore = create<BatchSelectionState>((set, get) => ({
  selectedIds: new Set(),
  isAllSelected: false,
  totalAvailable: 0,
  
  selectId: (id) => set((state) => ({
    selectedIds: new Set([...state.selectedIds, id]),
    isAllSelected: false,
  })),
  
  deselectId: (id) => set((state) => {
    const newSelected = new Set(state.selectedIds);
    newSelected.delete(id);
    return { selectedIds: newSelected, isAllSelected: false };
  }),
  
  toggleId: (id) => {
    const { isSelected, selectId, deselectId } = get();
    isSelected(id) ? deselectId(id) : selectId(id);
  },
  
  selectAll: (ids) => set({ selectedIds: new Set(ids), isAllSelected: false }),
  
  selectAllPages: (total) => set({ isAllSelected: true, totalAvailable: total }),
  
  clearSelection: () => set({ selectedIds: new Set(), isAllSelected: false }),
  
  setTotalAvailable: (total) => set({ totalAvailable: total }),
  
  getSelectedCount: () => {
    const { isAllSelected, totalAvailable, selectedIds } = get();
    return isAllSelected ? totalAvailable : selectedIds.size;
  },
  
  isSelected: (id) => {
    const { isAllSelected, selectedIds } = get();
    return isAllSelected || selectedIds.has(id);
  },
  
  getSelectedIds: () => Array.from(get().selectedIds),
}));
```

### BatchToolbar Component

```typescript
// src/components/batch/batch-toolbar.tsx
interface BatchToolbarProps {
  onReassign: () => void;
  onSendReminder: () => void;
  onUpdateStatus: () => void;
  onExport: () => void;
  onClear: () => void;
}

// Shows:
// - "X clients selected" count
// - Reassign button
// - Send Reminder button
// - Update Status button
// - Export button
// - Clear selection button
export function BatchToolbar(props: BatchToolbarProps): JSX.Element;
```

### SelectAllHeader Component

```typescript
// src/components/batch/select-all-header.tsx
interface SelectAllHeaderProps {
  pageSize: number;
  totalCount: number;
  currentPageIds: string[];
}

// Checkbox states:
// - Unchecked: no selection
// - Checked: all on page selected
// - Indeterminate: some on page selected
// Dropdown for "Select all X clients"
export function SelectAllHeader(props: SelectAllHeaderProps): JSX.Element;
```

### useBatchSelection Hook

```typescript
// src/hooks/use-batch-selection.ts
interface UseBatchSelectionReturn {
  selectedCount: number;
  isAllSelected: boolean;
  hasSelection: boolean;
  selectedIds: string[];
  toggleSelection: (id: string) => void;
  selectAllOnPage: (ids: string[]) => void;
  selectAllResults: (total: number) => void;
  clearSelection: () => void;
  isSelected: (id: string) => boolean;
}

export function useBatchSelection(): UseBatchSelectionReturn;
```

---

## Business Rules & Invariants

1. Maximum 100 clients can be selected for batch operations
2. Select-all checkbox has three states: unchecked, checked, indeterminate
3. "Select all X clients" option shown when total > page size
4. Selection cleared when navigating away from page
5. Selection persists when changing pages
6. Disabled rows (incomplete data) cannot be selected

---

## Edge Cases

1. **0 results** — Selection controls disabled
2. **100+ selected** — Show warning, disable batch actions
3. **Page change with selection** — Maintain selection, update counts
4. **Filter change** — Clear selection (different result set)

---

## Tests

### selection.test.tsx

- Individual row selection works
- Select-all selects all on current page
- Select-all-pages selects entire result set
- Deselect individual removes from selection
- Clear selection resets all
- Selection persists across pagination

---

## Verification

```bash
npm run test:unit -- selection.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Non-Functional Requirements → Bulk actions
