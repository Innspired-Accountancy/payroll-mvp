# Slice a: Evidence Data Model and Integrity

**Story:** story-09-evidence-store
**Epic:** epic-02-hmrc-submissions
**Effort:** S
**Dependencies:** None

---

## Goal

Design immutable evidence storage data model with integrity controls. Store submission payloads, HMRC responses, and acknowledgement data with SHA256 hashing for tamper detection.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Node.js crypto module
- [x] SDK methods identified: crypto.createHash('sha256'), db.insert()
- [x] External service endpoints: N/A (internal storage)
- [x] Data contracts defined: EvidenceRecord, EvidenceIntegrity interfaces
- [x] Configuration: EVIDENCE_RETENTION_YEARS=6
- [x] Error scenarios: Hash mismatch, storage failure, integrity violation
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Retain evidence for 6 years
- 02-10-audit-compliance-spec.md — Evidence retention requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/evidence/types.ts` | create | Evidence type definitions |
| `src/lib/hmrc/evidence/integrity.ts` | create | Hash generation and verification |
| `src/lib/db/schema/evidence.ts` | create | Evidence table schema |

---

## Responsibilities

1. Define evidence data model with immutability constraints
2. Implement SHA256 hashing for payload integrity
3. Store all submission-related evidence
4. Provide integrity verification
5. Support 6-year retention policy

---

## Contracts

### Evidence Data Model
```typescript
interface EvidenceRecord {
  id: string; // UUID
  submissionId: string; // FK to hmrc_submissions
  
  // Evidence content
  evidenceType: 'submission_payload' | 'hmrc_response' | 'acknowledgement' | 'error_response';
  content: string; // XML or JSON content
  contentHash: string; // SHA256 of content
  
  // Metadata
  createdAt: Date;
  retentionUntil: Date; // 6 years from creation
  
  // Integrity
  hashAlgorithm: 'sha256';
  
  // Storage
  storagePath?: string; // S3/MinIO path if offloaded
  compressed: boolean;
}

// Database schema (Drizzle)
const evidenceTable = pgTable('hmrc_evidence', {
  id: uuid('id').primaryKey().defaultRandom(),
  submissionId: uuid('submission_id').notNull().references(() => hmrcSubmissions.id),
  evidenceType: varchar('evidence_type', { length: 30 }).notNull(),
  content: text('content'), // Nullable if stored externally
  contentHash: varchar('content_hash', { length: 64 }).notNull(),
  hashAlgorithm: varchar('hash_algorithm', { length: 10 }).notNull().default('sha256'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  retentionUntil: timestamp('retention_until', { withTimezone: true }).notNull(),
  storagePath: varchar('storage_path', { length: 500 }),
  compressed: boolean('compressed').notNull().default(false)
});
```

### Integrity Service
```typescript
class EvidenceIntegrityService {
  // Generate SHA256 hash
  generateHash(content: string): string {
    return crypto.createHash('sha256').update(content, 'utf8').digest('hex');
  }
  
  // Verify content matches stored hash
  verify(record: EvidenceRecord): boolean {
    const calculatedHash = this.generateHash(record.content);
    return calculatedHash === record.contentHash;
  }
  
  // Calculate retention date (6 years)
  calculateRetentionDate(createdAt: Date): Date {
    return addYears(createdAt, 6);
  }
}
```

### Evidence Types
| Type | Content | When Stored |
|------|---------|-------------|
| submission_payload | FPS/EPS XML | On submission |
| hmrc_response | HTTP response | On receiving response |
| acknowledgement | Ack XML | On status update |
| error_response | Error XML | On rejection |
| validation_result | Validation JSON | On validation |

---

## Business Rules & Invariants

1. Evidence records are immutable (no updates, only inserts)
2. All evidence must have SHA256 hash for integrity
3. Retention period is 6 years from creation
4. Content hash verified on retrieval
5. Evidence linked to submission for audit trail
6. Compression for payloads >100KB

---

## Edge Cases

1. **Very large payload (10MB+)** — Compress, store reference
2. **Hash mismatch on retrieval** — Log alert, flag for investigation
3. **Retention period expired** — Soft delete (quarantine before purge)
4. **Duplicate evidence** — Hash-based deduplication
5. **Evidence for deleted submission** — CASCADE restriction, archive first

---

## Tests

### evidence-integrity.test.ts
- Generate consistent SHA256 hash
- Verify matching content
- Detect tampered content (hash mismatch)
- Calculate retention date correctly
- Store and retrieve evidence record

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/evidence/integrity.test.ts
npm run lint src/lib/hmrc/evidence/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Compliance
- 02-10-audit-compliance-spec.md — Evidence retention requirements
