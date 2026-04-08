# Slice b: Communication History Viewer UI

**Story:** story-06-communications
**Epic:** epic-04-bureau-operations
**Effort:** S
**Dependencies:** slice-a

---

## Goal

Create the communication history viewer UI that displays all client interactions in chronological order with filtering and search capabilities. Support timeline view and detailed message view.

---

## Decision Checklist

- [x] All libraries/packages named: @tanstack/react-query 5.x, date-fns 3.x, Tailwind CSS 3.4, Radix UI 1.x
- [x] All SDK methods/API calls identified: tRPC communications.list
- [x] All external service endpoints specified: tRPC communications.list
- [x] All data contracts defined: CommunicationListOutput from slice-a
- [x] All configuration/environment variables listed: N/A
- [x] All error scenarios identified with handling strategy: Loading, error, empty states
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § User Journeys — Communication history patterns
- 08-architecture-and-patterns.md — UI component patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/components/communications/communication-history.tsx` | create | Main history component |
| `src/components/communications/communication-timeline.tsx` | create | Timeline view |
| `src/components/communications/communication-item.tsx` | create | Single communication row |
| `src/components/communications/communication-filters.tsx` | create | Filter panel |
| `src/components/communications/communication-detail-modal.tsx` | create | Detail view modal |
| `src/components/communications/channel-badge.tsx` | create | Channel indicator |
| `src/components/communications/sent-by-badge.tsx` | create | Sender indicator |
| `src/tests/components/communications/history.test.tsx` | create | Component tests |

---

## Responsibilities

1. Create communication history timeline view
2. Display communications with sender, subject, date, channel, and status
3. Implement filters for type, channel, date range, and response status
4. Show opened/read status with timestamps
5. Provide full message content view
6. Support chronological and reverse chronological ordering

---

## Contracts

### CommunicationHistory Component

```typescript
// src/components/communications/communication-history.tsx
interface CommunicationHistoryProps {
  employerId: string;
  compact?: boolean; // If true, shows condensed view
}

// Uses: api.communications.list.useQuery({ employerId })
export function CommunicationHistory({ employerId, compact }: CommunicationHistoryProps): JSX.Element;
```

### CommunicationTimeline Component

```typescript
// src/components/communications/communication-timeline.tsx
interface CommunicationTimelineProps {
  communications: ClientCommunication[];
  onSelect: (communication: ClientCommunication) => void;
}

// Group by date, show timeline markers
export function CommunicationTimeline(props: CommunicationTimelineProps): JSX.Element;
```

### CommunicationItem Component

```typescript
// src/components/communications/communication-item.tsx
interface CommunicationItemProps {
  communication: ClientCommunication;
  onClick: () => void;
  isSelected?: boolean;
}

// Shows:
// - Channel icon (email/portal/sms)
// - Subject
// - Sent date/time
// - Sent by (system badge or user name)
// - Opened status (checkmark with date)
// - Response status
export function CommunicationItem(props: CommunicationItemProps): JSX.Element;
```

### CommunicationFilters Component

```typescript
// src/components/communications/communication-filters.tsx
interface CommunicationFiltersProps {
  filters: CommunicationFilters;
  onChange: (filters: CommunicationFilters) => void;
}

export interface CommunicationFilters {
  type?: CommunicationType;
  channel?: CommunicationChannel;
  sentBy?: 'system' | 'user';
  dateFrom?: Date;
  dateTo?: Date;
  responseReceived?: boolean;
}

export function CommunicationFilters(props: CommunicationFiltersProps): JSX.Element;
```

### ChannelBadge Component

```typescript
// src/components/communications/channel-badge.tsx
interface ChannelBadgeProps {
  channel: 'email' | 'portal' | 'sms';
}

// Icons:
// - email: Envelope icon
// - portal: ComputerDesktop icon
// - sms: ChatBubble icon
export function ChannelBadge({ channel }: ChannelBadgeProps): JSX.Element;
```

### SentByBadge Component

```typescript
// src/components/communications/sent-by-badge.tsx
interface SentByBadgeProps {
  sentBy: 'system' | 'user';
  userName?: string;
}

// Shows:
// - system: "Automated" badge
// - user: User name with avatar
export function SentByBadge({ sentBy, userName }: SentByBadgeProps): JSX.Element;
```

---

## Business Rules & Invariants

1. Timeline groups communications by date (Today, Yesterday, Date)
2. Unread communications show bold subject
3. System communications show "Automated" badge
4. Opened status shows time of first open
5. Response received shows checkmark with timestamp
6. Clicking item opens full content view

---

## Edge Cases

1. **No communications** — Show empty state with message
2. **Very long list** — Virtualized list for performance
3. **Email with no subject** — Show "(No subject)" placeholder
4. **Large content** — Modal with scroll for full message

---

## Tests

### history.test.tsx

- Timeline renders communications in order
- Channel badges display correctly
- Filter by type updates list
- Filter by date range works correctly
- Clicking item opens detail modal
- Empty state shows when no communications

---

## Verification

```bash
npm run test:unit -- history.test.tsx
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § User Journeys → Communication logging
