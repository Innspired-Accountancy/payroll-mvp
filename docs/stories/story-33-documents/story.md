# Story: Document Access (P30, P45, Summaries)

**Epic:** epic-07-employer-portal
**Priority:** 8 of 8
**Sprint:** Sprint 5
**Dependencies:** story-26-client-auth, story-10-payslip-generation

## Goal

Provide secure access to employer-facing documents including P30 payment notices, monthly summaries, year-end documentation, and P45s for leavers. Documents must be employer-scoped with appropriate retention and access controls.

## Acceptance Criteria

- [ ] Document list view with categories
- [ ] P30 payment notice access
- [ ] Monthly payroll summary documents
- [ ] Year-end P35/P60 summaries (when available)
- [ ] P45 documents for leavers
- [ ] Document search and filtering
- [ ] Date-based document organization
- [ ] PDF viewer with download option
- [ ] Document retention compliance (GDPR)
- [ ] Access logging for sensitive documents
- [ ] Expired document archival indication
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors introduced

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Document access by wrong employer | Low | Critical | Strict employer_id filtering; document ownership |
| PII exposure in document metadata | Low | High | Metadata scrubbing; secure generation |
| Document retention non-compliance | Medium | High | Retention policies; automated archival |
| Large document storage costs | Medium | Low | Compression; tiered storage; cleanup policies |
| Expired document access attempts | Low | Medium | Clear labeling; access denied handling |
| Audit trail gaps | Low | High | Comprehensive access logging; tamper-proof storage |

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | Document API and storage access | S | story-26-client-auth |
| slice-b | Document list and viewer UI | S | slice-a |

## Plan

{Will be populated by /wf-plan — do not fill during wf-research}

## Slices (detail)

- [slice-a.md](./slice-a.md) — Document API with secure access
- [slice-b.md](./slice-b.md) — Document UI components
