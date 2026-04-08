# Slice e: Reopen Authorization and Versioning

**Story:** story-08-pay-run-lifecycle
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-d

---

## Goal

Implement pay run reopen functionality with authorization, reason capture, and versioning to preserve original calculation.

---

## Decision Checklist

- [x] Authorization: Reopen permission required
- [x] Reason: Mandatory reason capture
- [x] Versioning: New version number, preserve original
- [x] Impact: Downstream system notification
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Correction handling
- 02-10-audit-compliance-spec.md — Change audit

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/reopen-pay-run.ts` | create | Reopen tRPC |
| `src/components/pay-run/reopen-dialog.tsx` | create | Reopen UI |
| `src/tests/reopen.test.ts` | create | Reopen tests |

---

## Responsibilities

1. Verify reopen authorization
2. Require reason for reopen
3. Create new calculation version
4. Preserve original approved version
5. Analyze downstream impact

---

## Contracts

### payRuns.reopen
- **Method:** tRPC mutation `payRuns.reopen`
- **Input:** `{ id: uuid, reason: string }`
- **Output:** Reopened pay run (new version)
- **Errors:** FORBIDDEN (no permission), BAD_REQUEST (missing reason)

### PayRunVersion
| Field | Type | Description |
|-------|------|-------------|
| pay_run_id | uuid | Pay run FK |
| version | int | Version number (1, 2, 3...) |
| status | enum | final, superseded |
| superseded_by | uuid | Newer version (if superseded) |
| created_at | DateTime | Version timestamp |

### ReopenRecord
| Field | Type | Description |
|-------|------|-------------|
| pay_run_id | uuid | Original pay run |
| reopened_by | uuid | User FK |
| reopened_at | DateTime | Timestamp |
| reason | string | Mandatory reason |
| new_version_id | uuid | New pay run version |

---

## Business Rules & Invariants

1. Only approved pay runs can be reopened
2. Reason is mandatory (min 10 characters)
3. Original version marked superseded, not deleted
4. New version starts in calculated state
5. Downstream systems notified of correction

---

## Edge Cases

1. **Multiple reopens** — Version 3, 4, etc.
2. **Reopen after FPS sent** — Flag for resubmission
3. **Reopen after payments** — Flag for adjustment

---

## Tests

### reopen.test.ts
- Reopen with valid permission and reason
- Reject reopen without reason
- Reject reopen without permission
- Version increment
- Original preserved as superseded
- Downstream impact flagged

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Correction Handling
- 02-10-audit-compliance-spec.md § Change Audit
