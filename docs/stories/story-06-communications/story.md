# Story: Communications - Email Logging and History

**Epic:** epic-04-bureau-operations
**Priority:** 6 of 8
**Sprint:** Month 4
**Dependencies:** story-05-client-chasing

---

## Goal

Build a comprehensive communication history system that logs all client interactions including automated reminders, manual emails, portal notifications, and client responses. Provide search and filtering capabilities for audit and reference purposes.

---

## Acceptance Criteria

- [ ] Communication data model supports email, portal, and SMS channels
- [ ] All automated reminder emails are logged with content, recipient, and timestamp
- [ ] All manual bureau emails are logged with sender, recipient, and content
- [ ] Email open tracking captures when client opens communication
- [ ] Response tracking marks communications as responded when client replies
- [ ] Communication history is viewable per client with chronological ordering
- [ ] Search supports filtering by date range, type, channel, and content
- [ ] Export capability for audit documentation
- [ ] GDPR compliance for communication retention and deletion
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Email tracking pixel blocked by clients | High | Low | Alternative engagement metrics, portal read receipts |
| Large communication history impacting performance | Medium | Medium | Archival policies, pagination, search indexing |
| Privacy compliance issues | Low | High | GDPR review, retention policies, consent tracking |
| Missing communication logs due to integration failure | Medium | High | Transactional logging, health monitoring, manual entry fallback |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Communication data model and API | S | story-05-client-chasing |
| slice-b | Communication history UI | S | slice-a |
| slice-c | Email tracking integration | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Communication data model and logging API
- [slice-b.md](./slice-b.md) — Communication history viewer UI
- [slice-c.md](./slice-c.md) — Email open/response tracking
