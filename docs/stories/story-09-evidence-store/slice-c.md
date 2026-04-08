# Slice c: Evidence Retrieval and Export

**Story:** story-09-evidence-store
**Epic:** epic-02-hmrc-submissions
**Effort:** XS
**Dependencies:** slice-b

---

## Goal

Implement evidence retrieval API with search, filtering, and export functionality. Support audit requests and compliance exports with integrity verification.

---

## Decision Checklist

- [x] All libraries/packages named: tRPC 11.x, JSZip 3.10.x (for export)
- [x] SDK methods identified: tRPC query procedures, zip generation
- [x] External service endpoints: N/A (internal)
- [x] Data contracts defined: EvidenceQuery, EvidenceExport, EvidencePackage interfaces
- [x] Configuration: EVIDENCE_EXPORT_MAX_SIZE_MB=50
- [x] Error scenarios: Evidence not found, integrity failure, export too large
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Support audit
- 02-10-audit-compliance-spec.md — Evidence export requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/routers/hmrc-evidence.ts` | create | Evidence tRPC router |
| `src/lib/hmrc/evidence/export.ts` | create | Export package generator |
| `src/lib/hmrc/evidence/search.ts` | create | Evidence search service |

---

## Responsibilities

1. Query evidence by submission, employer, tax year
2. Verify integrity on retrieval
3. Generate evidence packages for audit
4. Support date range filtering
5. Export as ZIP with manifest

---

## Contracts

### hmrc.getEvidence
- **Method:** tRPC query `hmrc.getEvidence`
- **Input:**
  ```typescript
  const getEvidenceInput = z.object({
    submissionId: z.string().uuid(),
    evidenceType: z.enum(['submission_payload', 'hmrc_response', 'acknowledgement', 'error_response', 'validation_result']).optional()
  });
  ```
- **Output:**
  ```typescript
  const getEvidenceOutput = z.array(z.object({
    id: z.string().uuid(),
    evidenceType: z.string(),
    content: z.string(),
    contentHash: z.string(),
    createdAt: z.string().datetime(),
    integrity: z.object({
      verified: z.boolean(),
      hashAlgorithm: z.string()
    })
  }));
  ```

### hmrc.searchEvidence
- **Method:** tRPC query `hmrc.searchEvidence`
- **Input:**
  ```typescript
  const searchEvidenceInput = z.object({
    employerId: z.string().uuid(),
    taxYear: z.string().optional(),
    dateFrom: z.string().datetime().optional(),
    dateTo: z.string().datetime().optional(),
    evidenceTypes: z.array(z.string()).optional(),
    limit: z.number().int().max(100).default(50),
    offset: z.number().int().default(0)
  });
  ```
- **Output:** Paginated evidence list with integrity status

### hmrc.exportEvidence
- **Method:** tRPC mutation `hmrc.exportEvidence`
- **Input:**
  ```typescript
  const exportEvidenceInput = z.object({
    employerId: z.string().uuid(),
    taxYear: z.string().optional(),
    dateFrom: z.string().datetime().optional(),
    dateTo: z.string().datetime().optional(),
    format: z.enum(['zip', 'json']).default('zip')
  });
  ```
- **Output:**
  ```typescript
  z.object({
    exportId: z.string().uuid(),
    downloadUrl: z.string().url(),
    expiresAt: z.string().datetime(),
    sizeBytes: z.number().int(),
    recordCount: z.number().int(),
    integrity: z.object({
      manifestHash: z.string(),
      algorithm: z.string()
    })
  });
  ```

### Evidence Package Structure (ZIP)
```
evidence-export-{exportId}.zip
├── manifest.json           # Index of all evidence
├── integrity.sha256        # Hash of manifest
└── evidence/
    ├── {submissionId}/
    │   ├── payload.xml     # Submission payload
    │   ├── response.xml    # HMRC response
    │   └── metadata.json   # Timestamps, hashes
    └── ...
```

### Manifest Format
```typescript
interface EvidenceManifest {
  exportId: string;
  generatedAt: Date;
  generatedBy: string;
  query: EvidenceQuery;
  records: Array<{
    id: string;
    submissionId: string;
    evidenceType: string;
    filename: string;
    contentHash: string;
    createdAt: Date;
  }>;
  integrity: {
    algorithm: 'sha256';
    manifestHash: string;
  };
}
```

---

## Business Rules & Invariants

1. All retrieved evidence is integrity-verified
2. Integrity failures are logged and flagged
3. Export includes manifest with hashes for verification
4. Export URLs expire after 24 hours
5. Max export size: 50MB (paginate if exceeded)
6. Export access requires hmrc:evidence:export permission

---

## Edge Cases

1. **Evidence not found** — Return 404 with available evidence list
2. **Integrity check fails** — Flag in response, log alert
3. **Export too large** — Paginate, return multiple export URLs
4. **Partial export failure** — Include successfully exported, list failures
5. **Export URL expired** — Return 410, require regeneration

---

## Tests

### evidence.router.test.ts
- Get evidence by submission ID
- Search evidence with filters
- Export evidence package
- Verify integrity in export
- Handle evidence not found

---

## Verification

```bash
npm run typecheck
npm run test src/server/routers/hmrc-evidence.router.test.ts
npm run lint src/server/routers/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Retain evidence for 6 years
- 02-10-audit-compliance-spec.md — Evidence export for compliance
