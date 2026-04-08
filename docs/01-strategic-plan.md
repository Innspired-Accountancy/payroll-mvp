# UK Bureau Payroll Platform — Strategic Plan

**Date:** 2026-04-08
**Version:** 1.0
**Research Path:** Greenfield

---

## Vision

To become the UK's most operationally effective bureau payroll platform — the operating system through which UK payroll bureaux and accountancy firms onboard, run, review, submit, pay, evidence, and support payroll across their client portfolios.

Unlike existing solutions that treat payroll as a calculation problem, this platform treats it as a **governed multi-party operating workflow** with hard deadlines, external dependencies (HMRC, pension providers, payment rails), and near-zero tolerance for silent failure.

The platform collapses fragmented workflows (payroll software, email chasing, banking portals, pension portals, manual approvals) into one controlled system of record with integrated compliance, payment execution, and audit-grade evidence retention.

---

## Business Model

**Primary Revenue Model:** SaaS subscription with tiered bureau pricing

| Tier | Target | Price Range | Features |
|------|--------|-------------|----------|
| **Starter** | Sole practitioners (5-30 clients) | £50-100/month | Core payroll, RTI, basic portal |
| **Professional** | Small bureaux (30-150 clients) | £150-400/month | Full compliance, CIS, pensions, payments |
| **Bureau** | Mid-size bureaux (150-500 clients) | £500-1500/month | Advanced workflow, API access, priority support |
| **Enterprise** | Large/multi-entity | Custom pricing | White-label, dedicated infra, custom dev |

**Per-employee pricing:** £1-3/employee/month (bureau passes to clients)

**Additional Revenue Streams:**
- Payment processing markup (Modulr/Telleroo)
- Migration services (one-time)
- Training and certification
- Premium support tiers

---

## Success Criteria

### Technical Success
- Pass HMRC software developer conformance testing
- 100% accuracy on reference payroll calculation test packs
- 99.9% uptime for core payroll operations
- <5s payroll calculation for 100 employees

### Commercial Success
- Pilot accountancy firm successfully processes live payrolls for 10+ clients by month 6
- 3+ pilot bureaux actively using platform by month 9
- First paying non-pilot customer by month 10
- £10K MRR by end of month 12

### Operational Success
- <0.5% late FPS incidence (vs industry average 2-3%)
- 95%+ pension submission first-time success rate
- 50%+ reduction in client data chase time (measured vs incumbent workflow)

---

## Timeline

| Phase | Target Date | Deliverables |
|-------|-------------|--------------|
| **Month 1-2: Foundation** | Jun 2026 | Core payroll engine, tax/NIC calculations, basic employee management |
| **Month 3-4: Compliance Core** | Aug 2026 | HMRC FPS/EPS generation, submission framework, NEST integration |
| **Month 5-6: Portals & Workflow** | Oct 2026 | Employee portal, employer portal, bureau dashboard, basic workflow |
| **Month 7-8: Payments & CIS** | Dec 2026 | Modulr integration, CIS module, payment approval workflows |
| **Month 9: Pilot Launch** | Jan 2027 | Pilot with accountancy firm, feedback iteration, bug fixes |
| **Month 10-12: Scale** | Mar 2027 | Additional pilot bureaux, migration tooling, performance optimization |

**Critical Path:** HMRC conformance process must start Month 3, target completion Month 8.

---

## Team

| Role | Count | Skills Needed |
|------|-------|---------------|
| **Product Manager/Domain Expert** | 1 | UK payroll expertise, HMRC processes, compliance requirements |
| **Tech Lead/Architect** | 1 | Distributed systems, security, compliance-sensitive architecture |
| **Backend Engineers** | 3-4 | Python/Node/Go, API design, database design, XML/API integrations |
| **Frontend Engineers** | 2 | React/Vue/Angular, responsive design, accessibility |
| **DevOps/Platform** | 1 | AWS/GCP/Azure, Kubernetes, CI/CD, security hardening |
| **QA Engineer** | 1 | Automated testing, payroll calculation validation, HMRC test packs |
| **UX Designer** | 1 | Complex workflow design, bureau operations research |

**Total Core Team:** 9-11 people

**Key Hiring Priority:** UK payroll domain expert with HMRC conformance experience.

---

## Technology Constraints

### Languages/Frameworks
- **Backend:** Python (Django/FastAPI) or Node.js (NestJS) — preference for strong typing and HMRC XML handling
- **Frontend:** React or Svelte with TypeScript
- **Database:** PostgreSQL (primary), Redis (caching/sessions)
- **Queue/Events:** RabbitMQ or AWS SQS

### Hosting/Deployment
- Cloud-native (AWS preferred for UK region/data residency)
- Kubernetes for container orchestration
- Multi-tenant architecture with logical isolation
- UK/EU data residency for GDPR compliance

### Integration Requirements
- **HMRC:** PAYE Online XML API, RTI technical specifications
- **NEST:** Web services API for pension submissions
- **Modulr:** Payment initiation and reconciliation APIs
- **Email/SMS:** SendGrid, Twilio for notifications

### Compliance/Security
- SOC 2 Type II preparation from day one
- GDPR compliance with UK data residency
- Encryption at rest (AES-256) and in transit (TLS 1.3)
- MFA for all bureau users

---

## Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| HMRC conformance rejection | Low | Critical | Early engagement, hire advisor, sandbox testing, flexible architecture |
| Payroll calculation errors in production | Low | Critical | Extensive unit tests, HMRC reference packs, parallel runs, pilot validation |
| Payment provider integration delays | Medium | High | Multiple provider architecture, manual fallback, early API access |
| Multi-tenant data breach | Low | Critical | Security-first design, pen testing, audit logging, least-privilege access |
| Key person dependency (payroll domain) | Medium | High | Document all domain knowledge, cross-train team |
| Pilot bureau churn before validation | Low | Medium | Strong relationship management, rapid iteration, clear success metrics |
| Competitor launches similar platform | Medium | Medium | Focus on bureau workflow differentiation, speed to market |
| Annual HMRC schema changes | High | Medium | Versioned schema handling, automated regression testing |

---

## Module Inventory

The platform is decomposed into 12 modules:

1. **Core Payroll Engine** — Calculations, pay runs, payslips
2. **HMRC Submissions** — RTI FPS/EPS, year-end, compliance
3. **CIS Module** — Subcontractor verification, returns, statements
4. **Pension & Auto-Enrolment** — Assessment, NEST integration, communications
5. **Employee Portal** — Payslips, P60, leave, self-service
6. **Employer/Client Portal** — Data input, approvals, visibility
7. **Payments Module** — Modulr integration, BACS, approval workflows
8. **Bureau Operations Centre** — Multi-client dashboard, tasks, exceptions
9. **Reporting & Documents** — Standard reports, exports, document archive
10. **Identity & Access Management** — RBAC, MFA, user management
11. **Audit & Compliance** — Immutable logs, evidence retention, compliance reports
12. **Migration & Onboarding** — Import tools, validation, parallel runs

Each module has a detailed specification in `docs/02-{NN}-{module}-spec.md`.
