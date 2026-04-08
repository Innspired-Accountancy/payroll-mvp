# Story: Compliance Evidence Store

**Epic:** epic-03-pension-ae
**Priority:** 8 of 8
**Sprint:** Month 5
**Dependencies:** story-02-assessment-engine, story-03-enrolment-workflow

---

## Goal

Implement an immutable compliance evidence store for pension assessments, enrolments, opt-outs, and communications per The Pensions Regulator retention requirements (6 years for assessments, 4 years for opt-outs).

---

## Acceptance Criteria

- [ ] Store all pension assessments immutably with timestamp
- [ ] Store all enrolment decisions with reason
- [ ] Store all opt-out records with evidence
- [ ] Store all communication dispatches with delivery proof
- [ ] Support 6-year retention for assessment records
- [ ] Support 4-year retention for opt-out records
- [ ] Provide audit trail query interface
- [ ] Generate compliance reports for TPR inspection
- [ ] Export evidence packages by employee or employer
- [ ] Tamper-evident record storage (hash/checksum)
- [ ] All tests pass
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Data loss | Low | Critical | Regular backups, immutable storage |
| Retention policy errors | Medium | High | Automated retention management, alerts |
| Large data volume | Medium | Medium | Archival strategy, compression |
| Audit query performance | Medium | Medium | Proper indexing, read replicas |
| Evidence tampering | Low | Critical | Checksums, audit logging, access controls |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Evidence record storage | S | story-02-assessment-engine, story-03-enrolment-workflow |
| slice-b | Audit query interface | S | slice-a |
| slice-c | Compliance reporting | S | slice-a |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Evidence record storage and retention
- [slice-b.md](./slice-b.md) — Audit query interface
- [slice-c.md](./slice-c.md) — Compliance reporting for TPR
