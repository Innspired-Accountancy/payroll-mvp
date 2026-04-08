# Slice a: Pension Scheme Data Model and API

**Story:** story-01-pension-schemes
**Epic:** epic-03-pension-ae
**Effort:** S
**Dependencies:** None

---

## Goal

Create the PensionScheme data model and tRPC API endpoints for CRUD operations with proper validation, tenant isolation, and encrypted API credential storage.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x, zod-to-json-schema 3.22.x
- [x] Data contracts defined: PensionSchemeCreate, PensionSchemeUpdate, PensionSchemeResponse Zod schemas
- [x] tRPC procedures: pensionSchemes.create, pensionSchemes.getById, pensionSchemes.update, pensionSchemes.listByEmployer, pensionSchemes.setDefault
- [x] Tenant isolation: employer_id foreign key + tRPC context check
- [x] Error scenarios: TRPCError with codes BAD_REQUEST, NOT_FOUND, CONFLICT, FORBIDDEN
- [x] Encryption: AES-256-GCM for api_config JSONB field using environment key
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-03-pension-auto-enrolment-spec.md — Entity: PensionScheme section
- 08-architecture-and-patterns.md — Multi-tenant isolation, encryption patterns

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/pensionSchemes.ts` | create | Drizzle table schema with indexes |
| `src/lib/db/schema/pensionEnums.ts` | create | Pension-related enums (provider, earnings_basis, relief_method) |
| `src/server/routers/pensionSchemes.ts` | create | tRPC procedures for scheme management |
| `src/lib/validation/pensionSchemes.ts` | create | Zod schemas (shared client/server) |
| `src/lib/encryption/fieldEncryption.ts` | create | AES-256-GCM encryption utility for sensitive fields |
| `src/lib/types/pension.ts` | create | TypeScript type definitions |
| `src/tests/pensionSchemes.test.ts` | create | Vitest integration tests |

---

## Responsibilities

1. Define PensionScheme model with all configuration fields
2. Implement CRUD API endpoints with proper authorization
3. Validate contribution rates (0-100% or valid fixed amounts)
4. Enforce single default scheme per employer via database constraint
5. Encrypt/decrypt api_config field transparently
6. Return proper error responses with actionable messages

---

## Contracts

### pensionSchemes.create
- **Method:** tRPC mutation `pensionSchemes.create`
- **Input:** PensionSchemeCreate Zod schema
- **Output:** PensionScheme (inserted record with decrypted api_config)
- **Errors:** 
  - BAD_REQUEST (validation failure)
  - CONFLICT (duplicate scheme name within employer)
  - FORBIDDEN (missing pension:scheme:create permission)
- **Auth:** Protected procedure requiring pension:scheme:create permission

### PensionSchemeCreate Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| employer_id | UUID | Yes | FK to Employer |
| provider | enum | Yes | "nest" or "other" |
| scheme_name | string(100) | Yes | Display name |
| employer_reference | string(50) | Yes | Provider reference |
| employee_contribution_rate | decimal(5,2) | Yes | Percentage 0-100 or fixed amount |
| employer_contribution_rate | decimal(5,2) | Yes | Percentage 0-100 or fixed amount |
| earnings_basis | enum | Yes | "qualifying" / "banded" / "total" |
| relief_method | enum | Yes | "relief_at_source" / "net_pay" |
| api_config | object | No | { nestUsername, nestPassword, nestOrganisationId } encrypted |
| is_default | boolean | No | Default false |

### pensionSchemes.setDefault
- **Method:** tRPC mutation `pensionSchemes.setDefault`
- **Input:** `{ schemeId: UUID }`
- **Output:** `{ success: true, previousDefaultId: UUID \| null }`
- **Errors:** NOT_FOUND (scheme doesn't exist), FORBIDDEN (wrong employer)

---

## Business Rules & Invariants

1. employer_id is mandatory (tenant isolation)
2. Only one default scheme per employer (enforced by partial unique index)
3. Contribution rates must be >= 0 and <= 100 for percentages
4. If earnings_basis is "qualifying", must use qualifying earnings bands
5. api_config is encrypted at rest using AES-256-GCM
6. scheme_name must be unique within employer (case-insensitive)
7. NEST schemes require nestOrganisationId in api_config

---

## Edge Cases

1. **Duplicate scheme name** — Return 409 with "Scheme name already exists for this employer"
2. **Setting default when another exists** — Unset previous default automatically
3. **Invalid contribution rate** — Return 400 with field-level error (e.g., "Must be between 0 and 100")
4. **Encrypted field query** — Cannot query inside encrypted api_config; use separate columns for searchable fields
5. **Null api_config** — Allowed for schemes not yet configured for API submission

---

## Tests

### pensionSchemes.test.ts
- Create scheme with valid data (expect 200, encrypted api_config)
- Create scheme with duplicate name (expect 409)
- Create scheme with invalid contribution rate > 100 (expect 400)
- Get scheme by ID (expect 200, decrypted api_config)
- List schemes by employer (expect 200, filtered by tenant)
- Set scheme as default (expect 200, previous default unset)
- Update scheme configuration (expect 200, audit trail)
- Attempt cross-tenant access (expect 403)
- Encryption round-trip test (save encrypted, retrieve decrypted)

---

## Verification

```bash
cd "/Users/josephstephenson-mouzo/Projects/03 - development/16 - payroll mvp"
npm run test:unit src/tests/pensionSchemes.test.ts
npm run typecheck
npm run lint
npm run build
```

---

## Source Sections

- 02-03-pension-auto-enrolment-spec.md § Entity: PensionScheme → Data model
- 02-03-pension-auto-enrolment-spec.md § API Contracts → Scheme management pattern
