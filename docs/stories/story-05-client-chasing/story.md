# Story: Client Chasing - Automated Reminders

**Epic:** epic-04-bureau-operations
**Priority:** 5 of 8
**Sprint:** Month 4
**Dependencies:** story-04-task-tracking

---

## Goal

Implement automated client chasing functionality to reduce manual follow-up work. System monitors approaching deadlines for payroll data submission and automatically sends reminder emails to clients. Escalates to bureau staff when clients don't respond within configured timeframes.

---

## Acceptance Criteria

- [ ] Chasing rules are configurable per client (timing, frequency, escalation)
- [ ] System detects when payroll data is incomplete X days before cut-off
- [ ] Automated reminder emails are sent via configured email service
- [ ] Reminder templates are customizable with client name, deadline, and portal link
- [ ] Escalation occurs when no response received within 24 hours of first chase
- [ ] Bureau dashboard shows chase status for each client
- [ ] Manual chase can be triggered by bureau staff with custom message
- [ ] All communications are logged for audit trail
- [ ] Unsubscribe/opt-out is handled per email regulations
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Emails flagged as spam | Medium | High | SPF/DKIM setup, rate limiting, reputation monitoring |
| Clients receiving excessive reminders | Medium | Medium | Smart frequency capping, preference center, opt-out |
| Escalation rules too aggressive or passive | Low | Medium | Configurable thresholds, manual override, feedback loop |
| Email delivery failures | Medium | Medium | Retry logic, bounce handling, alternative notification |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Chasing rules and configuration | M | story-04-task-tracking |
| slice-b | Automated reminder job | M | slice-a |
| slice-c | Escalation and dashboard integration | S | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Chasing rules engine and configuration UI
- [slice-b.md](./slice-b.md) — Automated reminder job and email delivery
- [slice-c.md](./slice-c.md) — Escalation workflow and dashboard status
