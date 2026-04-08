# Epic: HMRC Submissions

**Date:** 2026-04-08
**Sprint(s):** Months 3-4
**Dependencies:** epic-01-core-payroll (must be approved → triggers FPS)

---

## Scope & Deliverables

RTI FPS/EPS generation, validation, submission to HMRC, and evidence retention. Year-end final submission handling and correction workflows.

### In Scope
- FPS XML generation from pay run data
- EPS XML generation for adjustments
- HMRC schema validation
- Submission state machine (polling, retries)
- Error handling and mapping
- Year-end final submission
- Correction workflows (additional FPS)
- Evidence retention

### Out of Scope
- DPS inbound notices (phase 2)
- P9/Tax code notices automation (phase 2)

---

## Decisions

### Libraries & Packages

| Package | Version | Rationale | License |
|---------|---------|-----------|---------|
| fast-xml-parser | 4.x | XML generation and parsing | MIT |
| axios | 1.6.x | HTTP client for HMRC gateway | MIT |
| p-retry | 6.x | Retry logic with backoff | MIT |
| zod | 3.22.x | Response validation | MIT |

### HMRC Integration

| Aspect | Decision | Details |
|--------|----------|---------|
| Transport | HTTPS POST | HMRC PAYE Online XML API |
| Auth | HMRC Gateway | User ID, password, XML submission |
| Polling | Every 30s for 24h | Exponential backoff on errors |
| Retries | 3 attempts | Transient errors only |

### XML Handling

| Element | Approach |
|---------|----------|
| Generation | fast-xml-parser XMLBuilder |
| Validation | XSD validation via xml2js |
| Parsing | fast-xml-parser XMLParser |
| Storage | Original XML in filesystem (VPS) |

### Data Contracts

#### FPSSubmission
| Field | Type | Description |
|-------|------|-------------|
| correlation_id | UUID | Platform ID |
| tax_year | String | "2026-27" |
| tax_month | Int | 1-12 |
| pay_date | Date | Payment date |
| employees | List[FPSEmployee] | Per-employee data |

#### HMRC Response
| Field | Type | Description |
|-------|------|-------------|
| status | Enum | accepted/rejected/pending |
| hmrc_ref | String | HMRC correlation ID |
| errors | List[Error] | Validation errors |
| timestamp | DateTime | Response time |

---

## Build Order (Story Sequence)

| # | Story | Description | Effort | Dependencies |
|---|-------|-------------|--------|--------------|
| 1 | story-01-fps-generator | FPS XML generation from pay run | L | epic-01 complete |
| 2 | story-02-fps-validation | XSD validation, business rules | M | story-01 |
| 3 | story-03-hmrc-gateway | HTTPS submission, auth | M | story-02 |
| 4 | story-04-polling | Status polling, state machine | M | story-03 |
| 5 | story-05-error-handling | Error parsing, user guidance | M | story-04 |
| 6 | story-06-eps-generator | EPS for adjustments | L | story-01 |
| 7 | story-07-year-end | Final submission handling | M | story-05, story-06 |
| 8 | story-08-corrections | Additional FPS workflow | M | story-05 |
| 9 | story-09-evidence-store | Payload/response retention | S | story-04 |

---

## Decision Completeness Checklist

- [x] All third-party libraries named with versions
- [x] HMRC API pattern defined (XML over HTTPS)
- [x] Polling strategy defined (30s intervals, 24h timeout)
- [x] Retry logic specified (3 attempts, transient only)
- [x] Error handling strategy defined
- [x] Evidence retention approach defined (S3 storage)
- [x] No "TBD", slash-notation, or placeholder text remaining
