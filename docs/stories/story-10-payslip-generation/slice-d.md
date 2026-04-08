# Slice d: Bulk Generation and Secure Storage

**Story:** story-10-payslip-generation
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** slice-c

---

## Goal

Implement bulk payslip PDF generation for pay runs and secure encrypted storage.

---

## Decision Checklist

- [x] Bulk: Batch generation for pay run
- [x] Storage: Encrypted at rest
- [x] Access: Role-based permissions
- [x] Retention: Configurable retention policy
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Bulk payslip generation
- 02-10-audit-compliance-spec.md — Security requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/pdf/bulk-generator.ts` | create | Batch generator |
| `src/lib/storage/secure-storage.ts` | create | Encrypted storage |
| `src/server/routers/bulk-payslips.ts` | create | Bulk tRPC |
| `src/tests/bulk-generation.test.ts` | create | Bulk tests |

---

## Responsibilities

1. Generate all payslips for pay run
2. Encrypt PDFs at rest
3. Store with access controls
4. Provide secure download URLs

---

## Contracts

### bulkPayslips.generate
- **Method:** tRPC mutation `bulkPayslips.generate`
- **Input:** `{ pay_run_id: uuid }`
- **Output:** `{ generated: int, failed: int }`

### SecureStorage
| Method | Description |
|--------|-------------|
| `store(fileId, buffer, metadata): void` | Encrypt and store |
| `retrieve(fileId): Buffer` | Decrypt and return |
| `getUrl(fileId, expiry): string` | Presigned URL |
| `delete(fileId): void` | Secure delete |

---

## Business Rules & Invariants

1. All PDFs encrypted with AES-256
2. Access logged to audit trail
3. URLs expire after 1 hour
4. Retention: 7 years minimum

---

## Edge Cases

1. **Partial failure** — Retry failed individually
2. **Large pay run** — Batch in chunks
3. **Storage full** — Alert and queue

---

## Tests

### bulk-generation.test.ts
- Bulk generate for pay run
- Encryption roundtrip
- Access control enforcement
- URL expiration

---

## Verification

```bash
npm run test:unit
npm run typecheck
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Bulk Generation
- 02-10-audit-compliance-spec.md § Security
