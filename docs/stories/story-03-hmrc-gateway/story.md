# Story: HMRC Gateway Integration

**Epic:** epic-02-hmrc-submissions
**Priority:** 3 of 9
**Sprint:** Month 2
**Dependencies:** story-02-fps-validation

---

## Goal

Submit validated FPS/EPS XML to HMRC via the PAYE Online (RTI) gateway over HTTPS. Handle authentication using HMRC-issued credentials, implement secure communication with proper SSL/TLS configuration, and manage submission lifecycle with idempotency protection.

---

## Acceptance Criteria

- [ ] Submit FPS XML to HMRC test gateway endpoint
- [ ] Submit FPS XML to HMRC live gateway endpoint
- [ ] Authenticate using HMRC credentials (User ID, Password, Tax Office Number)
- [ ] Send proper HTTP headers including fraud prevention headers
- [ ] Receive and store HMRC correlation ID from response
- [ ] Handle HTTPS with TLS 1.2 minimum
- [ ] Implement idempotency to prevent duplicate submissions
- [ ] Circuit breaker for gateway unavailability
- [ ] Retry logic for transient failures (max 3 attempts)
- [ ] All tests pass (unit + integration)
- [ ] No lint/type-check errors

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| HMRC gateway downtime | Medium | High | Circuit breaker, queue submissions, exponential backoff retry |
| Authentication failures | Medium | High | Secure credential storage, automatic retry with fresh auth |
| TLS/SSL compatibility | Low | High | Test against HMRC endpoints early, enforce TLS 1.2+ |
| Idempotency key collisions | Low | Medium | Use UUID v4 with timestamp component, include employer identifier |

---

## Slices

| Slice | Description | Effort | Dependencies |
|-------|-------------|--------|--------------|
| slice-a | HTTP client configuration and security | M | None |
| slice-b | HMRC authentication handling | M | slice-a |
| slice-c | Submission endpoint and queuing | M | slice-b |

---

## Plan

{To be populated by /wf-plan}

---

## Slices (detail)

- [slice-a.md](./slice-a.md) — HTTPS client with TLS 1.2, fraud prevention headers
- [slice-b.md](./slice-b.md) — HMRC credential management and authentication
- [slice-c.md](./slice-c.md) — Submission service with idempotency and queuing
