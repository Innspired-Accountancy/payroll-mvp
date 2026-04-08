# Slice a: NEST API Client and Authentication

**Story:** story-05-nest-integration
**Epic:** epic-03-pension-ae
**Effort:** M
**Dependencies:** story-03-enrolment-workflow, story-04-contributions

---

## Goal

Create the NEST API client with authentication, request/response handling, and error management for NEST Web Services integration.

---

## Decision Checklist

- [x] HTTP client: axios 1.6.x with interceptors
- [x] Auth method: NEST OAuth 2.0 client credentials flow
- [x] Token storage: In-memory with refresh logic
- [x] Base URL: https://ws.nestpensions.org.uk/api/
- [x] API version: v1 (current NEST API version)
- [x] Content-Type: application/json for API, multipart/form-data for files
- [x] Retry logic: axios-retry with exponential backoff (3 retries)
- [x] Error handling: NEST-specific error codes mapped to application errors
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- NEST Web Services Developer Guide
- 02-03-pension-auto-enrolment-spec.md — Integration Points

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/nest/client.ts` | create | NEST API HTTP client |
| `src/lib/nest/auth.ts` | create | OAuth 2.0 authentication |
| `src/lib/nest/types.ts` | create | NEST API type definitions |
| `src/lib/nest/errors.ts` | create | NEST error handling |
| `src/lib/nest/config.ts` | create | NEST configuration |
| `src/tests/nest/client.test.ts` | create | Client unit tests |

---

## Responsibilities

1. Authenticate with NEST OAuth 2.0 endpoint
2. Manage access token lifecycle (obtain, refresh, cache)
3. Build and send API requests with proper headers
4. Handle HTTP errors and NEST-specific error codes
5. Implement retry logic for transient failures
6. Parse and validate NEST API responses

---

## Contracts

### NestClient
- **Constructor:** `new NestClient(config: NestConfig)`
- **Methods:**
  - `authenticate(): Promise<AuthToken>`
  - `get(path: string, params?: object): Promise<Response>`
  - `post(path: string, body: object): Promise<Response>`
  - `postFile(path: string, file: Buffer, filename: string): Promise<Response>`

### NestConfig
| Field | Type | Description |
|-------|------|-------------|
| baseUrl | string | "https://ws.nestpensions.org.uk/api/v1" |
| authUrl | string | "https://ws.nestpensions.org.uk/auth/token" |
| username | string | NEST organisation username |
| password | string | NEST organisation password |
| organisationId | string | NEST organisation ID |
| timeout | number | Request timeout in ms (default 30000) |

### Auth Flow
1. POST to /auth/token with client credentials
2. Receive { access_token, expires_in, token_type }
3. Cache token in memory
4. Use token in Authorization: Bearer header
5. Refresh token when expired or on 401 response

### NEST Error Codes
| Code | Meaning | Action |
|------|---------|--------|
| INVALID_ORGANISATION | Organisation not found | Check credentials |
| INVALID_EMPLOYEE_REF | Employee not found | Verify enrolment first |
| DUPLICATE_SUBMISSION | Already submitted | Check submission status |
| VALIDATION_ERROR | Data validation failed | Review and correct |
| SYSTEM_ERROR | NEST internal error | Retry with backoff |

---

## Business Rules & Invariants

1. Token cached until expiry minus 5-minute buffer
2. All requests include organisation ID in headers
3. 401 responses trigger token refresh and retry
4. 5xx responses trigger exponential backoff retry
5. Non-retryable errors (4xx except 401) fail immediately
6. All requests logged with correlation ID

---

## Edge Cases

1. **Token refresh fails** — Clear cache, re-authenticate from scratch
2. **Multiple concurrent requests with expired token** — Queue until refresh complete
3. **NEST API timeout** — Retry with increased timeout
4. **Organisation suspended** — Fail with specific error code

---

## Tests

### client.test.ts
- Successful authentication
- Token caching and reuse
- Automatic token refresh on expiry
- Request with authentication header
- Retry on 5xx error
- No retry on 4xx error
- Timeout handling

---

## Verification

```bash
cd "/Users/josephstephenson-mouzo/Projects/03 - development/16 - payroll mvp"
npm run test:unit src/tests/nest/client.test.ts
npm run typecheck
npm run lint
npm run build
```

---

## Source Sections

- story-01-pension-schemes/slice-a.md → NEST API credentials storage
