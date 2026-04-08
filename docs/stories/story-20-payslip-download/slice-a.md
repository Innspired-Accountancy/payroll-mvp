# Slice a: Secure PDF Download with Audit Logging

**Story:** story-20-payslip-download
**Epic:** epic-06-employee-portal
**Effort:** S
**Dependencies:** story-19-payslip-view (payslip viewing infrastructure)

---

## Goal

Implement secure PDF download functionality for payslips with audit logging. Downloads must use signed URLs, enforce data isolation, complete within 5 seconds, and log all access for compliance.

---

## Decision Checklist

- [x] All libraries/packages named: @react-pdf/renderer 3.4.x (or pre-generated PDFs), uuid 9.0.x
- [x] SDK methods/API calls identified: payslips.download(id), storage.getSignedUrl()
- [x] External service endpoints: None (VPS file storage)
- [x] Data contracts defined: PayslipDownloadResponse, SignedUrlConfig
- [x] Configuration variables: DOWNLOAD_URL_EXPIRY=900, PDF_STORAGE_PATH=/storage/payslips
- [x] Error scenarios identified: File not found, permission denied, generation timeout
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-05-employee-portal-spec.md — Journey 1: View Payslip (download step)
- 02-11-reporting-documents-spec.md — PDF generation, secure document storage
- 02-10-audit-compliance-spec.md — Download audit logging

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/api/payslips/[id]/download/route.ts` | create | Download API route with auth |
| `src/components/payslips/download-button.tsx` | create | Download button with loading state |
| `src/lib/storage/payslip-storage.ts` | create | File storage utilities |
| `src/lib/pdf/payslip-pdf.ts` | create | PDF generation or retrieval |
| `src/server/routers/payslips.ts` | update | Add download procedure |

---

## Responsibilities

1. Generate or retrieve PDF for payslip on download request
2. Validate employee has access to requested payslip
3. Generate secure download URL with 15-minute expiry
4. Stream PDF to browser with correct content-type
5. Log download with timestamp, IP, payslip ID
6. Set filename as "EMP{number}-{date}-payslip.pdf"

---

## Contracts

### payslips.getDownloadUrl
- **Method:** tRPC query `payslips.getDownloadUrl`
- **Input:**
  ```typescript
  {
    id: string;           // UUID of payslip
  }
  ```
- **Output:**
  ```typescript
  {
    downloadUrl: string;  // Signed URL: /api/payslips/uuid/download?token=xyz
    expiresAt: string;    // ISO timestamp
    filename: string;     // "EMP001-2026-04-30-payslip.pdf"
  }
  ```
- **Errors:**
  - `UNAUTHORIZED` — Invalid session
  - `NOT_FOUND` — Payslip doesn't exist or no PDF available
  - `FORBIDDEN` — Payslip belongs to different employee
- **Auth:** Protected procedure with employee session validation

### GET /api/payslips/{id}/download
- **Method:** HTTP GET (API route)
- **Query:**
  ```
  token: string          // Signed JWT token (15-min expiry)
  ```
- **Response:**
  - Success: `Content-Type: application/pdf`, PDF stream
  - Error: JSON error with status code
- **Headers:**
  - `Content-Disposition: attachment; filename="EMP001-2026-04-30-payslip.pdf"`
- **Errors:**
  - `401` — Invalid or expired token
  - `404` — PDF not found
  - `410` — Token expired

### PDF Filename Format
```typescript
`EMP${employeeNumber}-${payDate}-payslip.pdf`
// Example: EMP001-2026-04-30-payslip.pdf
```

---

## Business Rules & Invariants

1. Download token expires after 15 minutes (900 seconds)
2. Token is single-use (optional: implement one-time tokens)
3. Filename includes employee number and pay date for organization
4. PDF content matches on-screen payslip exactly
5. Download logged before streaming begins
6. Failed downloads also logged with error reason

---

## Edge Cases

1. **PDF not yet generated** — Generate on-demand (max 5 second timeout)
2. **Large PDF file** — Stream response, don't buffer in memory
3. **Concurrent downloads** — Allow, each gets own signed URL
4. **Download interrupted** — Client can request new URL, re-download
5. **Browser blocks download** — Open in new tab as fallback

---

## Tests

### payslips.router.test.ts
- getDownloadUrl returns valid signed URL
- URL contains correct filename
- Expires at correct timestamp
- Rejects unauthorized payslip access

### download-route.test.ts
- Valid token streams PDF
- Expired token returns 410
- Invalid token returns 401
- Sets correct Content-Disposition header

### download-button.test.tsx
- Click triggers download
- Loading state during fetch
- Error message on failure
- Success triggers browser download

---

## Verification

```bash
# Type checking
npx tsc --noEmit

# Linting
npx next lint

# Tests
npx vitest run src/server/routers/payslips.test.ts
npx vitest run src/app/api/payslips/download/route.test.ts
npx vitest run src/components/payslips/download-button.test.tsx
```

---

## Source Sections

- epic-06-employee-portal/epic-plan.md § Payslip Download → PDF download with audit
- 02-05-employee-portal-spec.md § API Contracts → download_url in payslip list
- 02-05-employee-portal-spec.md § Non-Functional → PDF download <5 seconds
- 02-11-reporting-documents-spec.md § Security → Time-limited download URLs
