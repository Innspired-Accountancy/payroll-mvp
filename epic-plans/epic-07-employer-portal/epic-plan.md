# Epic: Employer/Client Portal

**Date:** 2026-04-08
**Sprint(s):** Months 5-6
**Dependencies:** epic-01-core-payroll, epic-04-bureau-operations

---

## Scope & Deliverables

Client-facing portal for employer admins and managers to submit variable pay data, approve payrolls, view reports, and manage starters/leavers.

### In Scope
- Variable pay data submission
- Payroll approval workflow
- Starter/leaver management
- Report viewing
- Document access
- Multi-company access (for groups)

### Out of Scope
- Direct payroll editing (bureau only)
- Configuration changes (bureau only)
- Advanced analytics (phase 2)

---

## Decisions

### Access Levels

| Role | Permissions |
|------|-------------|
| Admin | Submit data, approve payroll, manage employees |
| Manager | Approve payroll, view reports |
| Viewer | View reports, documents |

### Approval Workflow

| Action | Approver | Notification |
|--------|----------|--------------|
| Payroll submission | Bureau processor | Dashboard alert |
| Payroll approval | Manager/Admin | Email + dashboard |
| Bank change | Bureau | Task created |
| Starter added | Bureau | Dashboard alert |

---

## Build Order (Story Sequence)

| # | Story | Description | Effort | Dependencies |
|---|-------|-------------|--------|--------------|
| 1 | story-01-client-auth | Portal login, roles | M | epic-05 |
| 2 | story-02-dashboard | Client dashboard view | M | epic-04 |
| 3 | story-03-variable-pay-form | Hours/bonuses input | M | epic-01 |
| 4 | story-04-payroll-approval | Review and approve | M | epic-01 |
| 5 | story-05-starter-form | New employee capture | M | epic-01 |
| 6 | story-06-leaver-form | Leaving process | S | epic-01 |
| 7 | story-07-reports | Cost reports, summaries | M | epic-01 |
| 8 | story-08-documents | P30, summary access | S | epic-01 |

---

## Decision Completeness Checklist

- [x] Access levels defined (Admin/Manager/Viewer)
- [x] Approval workflow specified
- [x] Permissions clearly scoped
- [x] No "TBD", slash-notation, or placeholder text remaining
