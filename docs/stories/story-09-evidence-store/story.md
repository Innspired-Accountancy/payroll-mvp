# Story: Submission Evidence Store

**Epic:** epic-02-hmrc-submissions
**Priority:** 9 of 9
**Sprint:** Month 3
**Dependencies:** story-04-polling

---

## Goal

Maintain a complete audit trail of all HMRC submissions by storing request payloads, HMRC responses, and acknowledgement data for the required 6-year retention period. Support retrieval for compliance audits and dispute resolution.

---

## Acceptance Criteria

- [ ] Store FPS/EPS XML payload at time of submission
- [ ] Store SHA256 hash of payload for integrity verification
- [ ] Store HMRC response XML (acknowledgement or rejection)
- [ ] Store all correlation IDs (platform and HMRC)
- [ ] Immutable storage (no updates, only inserts)
- [ ] 6-year retention with automated purging after retention period
- [ ] Search and retrieval API by submission ID, employer, tax year
- [ ] Export evidence package for audit/compliance
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Storage volume growth | Medium | Medium | Compression, storage tiering, automated archival policy |
| Data integrity failures | Low | High | Hash verification, checksums on retrieval, backup strategy |
| Retention policy compliance | Medium | High | Automated deletion jobs, audit logs, legal hold capability |
| Retrieval performance | Medium | Low | Indexing, archive tier for old data, query optimization |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Evidence storage schema and immutability | S | None |
| slice-b | Payload and response capture | S | slice-a |
| slice-c | Evidence retrieval and export API | XS | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — Evidence data model and integrity controls
- [slice-b.md](./slice-b.md) — Submission/response capture hooks
- [slice-c.md](./slice-c.md) — Evidence retrieval API and export functionality
