# Slice a: Evidence Record Storage

**Story:** story-08-evidence
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** story-02-assessment-engine, story-03-enrolment-workflow

---

## Goal

Implement immutable evidence record storage with tamper-evident hashes and automated retention management.

---

## Decision Checklist

- [x] Storage: PostgreSQL with append-only pattern
- [x] Immutability: No UPDATE/DELETE on evidence tables
- [x] Tamper evidence: SHA-256 hash of record content
- [x] Retention: 6 years assessments, 4 years opt-outs (configurable)
- [x] Archival: Compress and move to cold storage after retention
- [x] Hash chain: Optional blockchain-style chaining for critical records
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Non-Functional Requirements (Compliance)
- The Pensions Regulator: "Record keeping requirements"

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/evidence/evidenceStore.ts` | create | Evidence recording utilities |
| `src/lib/evidence/hashing.ts` | create | SHA-256 hash generation |
| `src/lib/evidence/retention.ts` | create | Retention policy management |
| `src/lib/db/schema/evidenceRecords.ts` | create | Evidence tables |
| `src/lib/jobs/retentionEnforcement.ts` | create | Automated retention job |
| `src/tests/evidence/evidenceStore.test.ts` | create | Evidence tests |

---

## Responsibilities

1. Record evidence with content hash
2. Store immutable assessment records
3. Store immutable enrolment records
4. Store immutable opt-out records
5. Store communication dispatch records
6. Enforce retention policies
7. Archive expired records

---

## Contracts

### recordEvidence()
- **Method:** `recordEvidence(input: EvidenceInput): Promise<EvidenceRecord>`
- **Input:** Evidence data to store
- **Output:** Created evidence record with hash
- **Hash:** SHA-256 of canonical JSON representation

### EvidenceInput Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| record_type | enum | Yes | "assessment" / "enrolment" / "opt_out" / "communication" |
| employee_id | UUID | Yes | Related employee |
| employer_id | UUID | Yes | Related employer |
| reference_id | UUID | Yes | ID of source record |
| payload | JSONB | Yes | Evidence content |
| retention_years | number | Yes | 6 for assessment, 4 for opt-out |

### EvidenceRecord
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Evidence record ID |
| record_type | enum | Type of evidence |
| employee_id | UUID | Employee reference |
| employer_id | UUID | Employer reference |
| reference_id | UUID | Source record ID |
| payload | JSONB | Evidence content |
| content_hash | string | SHA-256 hash of payload |
| recorded_at | DateTime | When evidence stored |
| retain_until | Date | Retention expiry date |

### verifyEvidence()
- **Method:** `verifyEvidence(evidenceId: UUID): Promise<boolean>`
- **Action:** Recompute hash and compare to stored hash
- **Returns:** True if hash matches (not tampered)

---

## Business Rules & Invariants

1. Evidence records are immutable (no updates or deletes)
2. Content hash computed at creation and stored
3. Retention date calculated from record date
4. Records archived (not deleted) after retention period
5. Hash verification available for audit

---

## Edge Cases

1. **Duplicate evidence** — Idempotent: return existing record
2. **Very large payload** — Compress before storage
3. **Hash collision** — Extremely unlikely, log warning
4. **Clock skew** — Use database timestamp, not application

---

## Tests

### evidenceStore.test.ts
- Evidence recording with hash
- Hash verification success
- Hash tampering detection
- Retention date calculation
- Duplicate handling

---

## Verification

```bash
cd "/Users/josephstephenson-mouzo/Projects/03 - development/16 - payroll mvp"
npm run test:unit src/tests/evidence/evidenceStore.test.ts
npm run typecheck
npm run lint
npm run build
```

---

## Source Sections

- 02-03-pension-auto-enrolment-spec.md § Compliance → Record retention
- story-02-assessment-engine → Assessment data to preserve
- story-03-enrolment-workflow → Enrolment data to preserve
