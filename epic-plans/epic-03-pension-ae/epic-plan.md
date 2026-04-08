# Epic: Pension & Auto-Enrolment

**Date:** 2026-04-08
**Sprint(s):** Months 3-4 (parallel with HMRC)
**Dependencies:** epic-01-core-payroll (earnings data for assessment)

---

## Scope & Deliverables

Auto-enrolment assessment engine, enrolment workflow, NEST integration, statutory communications, and compliance evidence retention.

### In Scope
- Worker assessment per pay period
- Eligible jobholder auto-enrolment
- Postponement handling
- Opt-in/opt-out workflows
- Contribution calculations
- NEST API integration
- Statutory letter generation
- Evidence retention (6 years)

### Out of Scope
- Multiple pension providers (NEST first only)
- Re-enrolment (phase 2)
- Pension provider file fallback (MVP uses API only)

---

## Decisions

### Libraries & Packages

| Package | Version | Rationale | License |
|---------|---------|-----------|---------|
| httpx | 0.27.x | Async HTTP for NEST API | BSD |
| jinja2 | 3.1.x | Letter template rendering | BSD |
| weasyprint | 62.x | PDF letter generation | BSD-3 |

### NEST Integration

| Aspect | Decision | Details |
|--------|----------|---------|
| Auth | API Key | NEST web services API key |
| Protocol | REST + XML | NEST supports SOAP/REST hybrid |
| Endpoints | Contributions, Enrolment, Opt-out | Per NEST API docs |
| Testing | NEST Test Facility | Sandbox environment |

### Assessment Engine

| Rule | Implementation |
|------|----------------|
| Age check | 22-67 (state pension age) |
| Earnings check | ≥ £10,000/year (pro-rated per period) |
| Category | Eligible/Non-eligible/Entitled |
| Assessment date | Pay reference period end |

### Communication Letters

| Letter | Trigger | Timing |
|--------|---------|--------|
| Enrolment | First eligibility | Within 6 weeks |
| Opt-out confirmation | Opt-out received | Within 1 month |
| Postponement | Postponement applied | Within 6 weeks of duties |

---

## Build Order (Story Sequence)

| # | Story | Description | Effort | Dependencies |
|---|-------|-------------|--------|--------------|
| 1 | story-01-pension-schemes | Scheme setup, configuration | S | epic-01 |
| 2 | story-02-assessment-engine | Worker categorization logic | M | epic-01 |
| 3 | story-03-enrolment-workflow | Auto-enrol eligible workers | M | story-02 |
| 4 | story-04-contributions | Qualifying earnings calc | M | story-02 |
| 5 | story-05-nest-integration | NEST API submission | L | story-03, story-04 |
| 6 | story-06-opt-out | Opt-out workflow, refunds | M | story-03 |
| 7 | story-07-communications | Letter generation, dispatch | M | story-03, story-06 |
| 8 | story-08-evidence | Assessment/enrolment records | S | story-02, story-03 |

---

## Decision Completeness Checklist

- [x] All third-party libraries named with versions
- [x] NEST API pattern defined (REST with API key)
- [x] Assessment rules specified (age, earnings)
- [x] Communication timing defined (6 weeks)
- [x] Evidence retention period defined (6 years)
- [x] No "TBD", slash-notation, or placeholder text remaining
