# Slice b: P60 PDF Download and Audit Logging

**Story:** story-21-p60-access
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** slice-a (P60 list and view)

---

## Goal

Implement secure PDF download for P60 documents with audit logging. P60 downloads follow the same security pattern as payslips with signed URLs, data isolation, and comprehensive access logging.

---

## Decision Checklist

- [x] All libraries/packages named: @react-pdf/renderer 3.4.x, uuid 9.0.x
- [x] SDK methods/API calls identified: documents.getP60DownloadUrl(id), storage.retrieve()
- [x] External service endpoints: None (VPS file storage)
- [x] Data contracts defined: P60DownloadResponse, P60PdfConfig
- [x] Configuration variables: DOWNLOAD_URL_EXPIRY=900, PDF_STORAGE_PATH=/storage/p60s
- [x] Error scenarios identified: File not found, not yet available, permission denied
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 2: Access P60 (download step)
- 02-11-reporting-documents-spec.md — PDF generation, secure storage
- 02-10-audit-compliance-spec.md — Download audit requirements

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/api/documents/p60/[id]/download/route.ts` | create | P60 download API route |
| `src/components/documents/p60-download-button.tsx` | create | P60 download button |
| `src/lib/pdf/p60-pdf.ts` | create | P60 PDF generation |
| `src/server/routers/documents.ts` | update | Add P60 download procedure |
| `src/lib/storage/p60-storage.ts` | create | P60 file storage utilities |

---

## Responsibilities

1. Generate or retrieve P60 PDF on download request
2. Validate employee has access to requested P60
3. Enforce availability date restriction (not before 31 May)
4. Generate secure download URL with 15-minute expiry
5. Stream PDF with correct HMRC P60 layout
6. Log download with timestamp, IP, document ID
7. Set filename as "P60-{taxYear}-{employeeNumber}.pdf"

---

## Contracts

### documents.getP60DownloadUrl
- **Method:** tRPC query `documents.getP60DownloadUrl`
- **Input:**
  ```typescript
  {
    id: string;           // UUID of P60 record
  }
  ```
- **Output:**
  ```typescript
  {
    downloadUrl: string;  // Signed URL
    expiresAt: string;    // ISO timestamp
    filename: string;     // "P60-2025-2026-EMP001.pdf"
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — P60 doesn't exist
  - `FORBIDDEN` — P60 belongs to different employee
  - `NOT_AVAILABLE` — Before 31 May availability date
- **Auth:** Protected procedure with employee session validation

### GET /api/documents/p60/{id}/download
- **Method:** HTTP GET (API route)
- **Query:**
  ```
  token: string          // Signed JWT token
  ```
- **Response:**
  - Success: `Content-Type: application/pdf`, PDF stream
  - Error: JSON error with status code
- **Headers:**
  - `Content-Disposition: attachment; filename="P60-2025-2026-EMP001.pdf"`
- **Errors:**
  - `401` — Invalid or expired token
  - `404` — PDF not found
  - `403` — Not available yet
  - `410` — Token expired

### P60 PDF Layout
```
┌─────────────────────────────────────────────────────┐
│  P60 - End of Year Certificate                      │
│  Tax Year: 2025-2026                                │
├─────────────────────────────────────────────────────┤
│  Employer: ABC Ltd                                  │
│  PAYE Reference: 123/AB45678                        │
├─────────────────────────────────────────────────────┤
│  Employee: John Smith                               │
│  NI Number: AB123456C                               │
│  Works Number: EMP001                               │
├─────────────────────────────────────────────────────┤
│  1. Pay: £35,000.00                                 │
│  2. Tax deducted: £5,400.00                         │
│  3. Final tax code: 1257L                           │
│  4. NIC table letter: A                             │
│  5. Employee NIC: £3,240.00                         │
│  6. Statutory payments: £0.00                       │
└─────────────────────────────────────────────────────┘
```

---

## Business Rules & Invariants

1. P60 PDF follows HMRC standard layout (boxes 1-6)
2. Download only available from 31 May following tax year
3. Filename format: P60-{taxYear}-{employeeNumber}.pdf
4. PDF content matches official HMRC P60 format
5. Download logged for compliance audit trail
6. Same security controls as payslip downloads

---

## Edge Cases

1. **Download before 31 May** — Return 403 with availability date
2. **PDF generation pending** — Show "generating" status, queue for generation
3. **Amended P60** — Generate new PDF, maintain version history
4. **Browser preview vs download** — Default to download (attachment)
5. **Mobile download** — Ensure mobile browsers handle PDF correctly

---

## Tests

### documents.router.test.ts
- getP60DownloadUrl returns valid signed URL
- Rejects before availability date
- Correct filename format

### p60-download-route.test.ts
- Valid token streams PDF
- Expired token returns 410
- Not available returns 403

### p60-download-button.test.tsx
- Disabled before availability date
- Loading state during fetch
- Success triggers download

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/documents.test.ts
npx vitest run src/app/api/documents/p60/download/route.test.ts
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § P60 Access → PDF download
- 02-05-employee-portal-spec.md § Journey 2: Access P60 → Download PDF
- 02-11-reporting-documents-spec.md § Security → Time-limited download URLs
