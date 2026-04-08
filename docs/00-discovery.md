# Discovery: UK Bureau Payroll Platform

**Date:** 2026-04-08
**Verdict:** GO (with HMRC conformance as critical path dependency)

## Problem Statement

UK payroll bureaux and accountancy firms currently operate across fragmented systems: payroll software, email chasing, spreadsheets, banking portals, pension portals, manual approvals, and disconnected document distribution. This fragmentation creates deadline risk, weak auditability, duplicated effort, and poor client experience. Incumbent tools like BrightPay (desktop-rooted), FreshPay (smaller footprint), and Xero Payroll (not bureau-first) leave critical gaps in multi-client workflow orchestration, exception management, and integrated compliance automation.

## Target Users

**Primary Users:**
1. **UK Payroll Bureaux** - Dedicated payroll service providers managing 50-500+ employer clients
2. **Accountancy Firms** - Practices offering payroll services to SME clients (10-200+ clients)

**Secondary Users:**
- Employers (clients of bureaux) needing visibility, approvals, and data submission
- Employees needing payslip access, leave requests, and personal data management

**Tertiary Stakeholders:**
- HMRC (submission recipient and compliance authority)
- The Pensions Regulator (auto-enrolment enforcement)
- Pension providers (NEST and others)
- Payment providers (Modulr, Telleroo)

### Pain Points
1. Managing dozens/hundreds of clients across disconnected tools
2. Email-led client data collection with missed cut-offs
3. Weak maker/checker controls and approval governance
4. Separate logins for payroll, pensions, and payment execution
5. Poor visibility of failed HMRC submissions or payment failures
6. Cumbersome mid-year migrations from incumbent systems
7. Manual document distribution and weak audit trails

## Existing Alternatives

| Solution | Strengths | Weaknesses/Gaps |
|----------|-----------|-----------------|
| **BrightPay** | Mature payroll engine, bureau licensing, Connect portals, Modulr integration | Desktop-rooted architecture, limited multi-user concurrency, workflow/tasking less central |
| **FreshPay** | Cloud-native, bureau dashboard, Telleroo integration, PensionSync | Smaller market footprint, less evidenced audit-governance depth, third-party pension reliance |
| **Xero Payroll** | Strong cloud UX, accounting adjacency, direct pension connections | Not bureau-first, weak multi-client task orchestration, limited CIS/P11D depth |

**Why insufficient:** No incumbent delivers a true cloud-native bureau *operating system* with integrated workflow orchestration, exception queues, migration tooling, and payment governance as foundational architecture.

## Feasibility Assessment

**Technical:** HIGH — UK payroll calculations are well-documented deterministic algorithms. HMRC XML submission patterns are established. Cloud-native multi-tenant architectures are proven. Risk lies in accuracy validation and HMRC conformance process.

**Resource:** MEDIUM — Requires team with UK payroll domain expertise, HMRC integration experience, and compliance-sensitive engineering practices. 6-9 month timeline is aggressive but achievable with parallel workstreams.

**Timeline:** MEDIUM — 6-9 months to Minimum Credible Replacement (MVP) with pilot bureau usage. Critical path is HMRC conformance testing and payment provider integration.

## Scope

### In Scope (V1 - Minimum Credible Replacement)

**Core Payroll:**
- Weekly, fortnightly, four-weekly, monthly payroll calculations
- PAYE tax (cumulative, Week 1/Month 1, Scottish/Welsh rates, K-codes)
- NIC (all categories, directors annual/alternative methods)
- Starters/leavers, P45/P60 generation
- Statutory payments (SSP, SMP, SPP, SAP, ShPP)
- Student/postgraduate loans
- Salary sacrifice, basic AEO support

**Compliance:**
- HMRC RTI FPS/EPS submissions (XML)
- Year-end final submission handling
- Submission validation, status polling, error handling
- CIS subcontractor verification and returns

**Pensions:**
- Auto-enrolment assessment engine
- NEST integration (primary provider)
- Worker categorization, postponement, opt-in/opt-out
- Statutory communications tracking

**Portals:**
- Employee portal (payslips, P60, leave requests, personal details)
- Employer/client portal (variable data input, approvals, documents)

**Operations:**
- Multi-client bureau dashboard
- Task/deadline tracking
- Basic exception queues
- User management and RBAC
- Audit logging

**Payments:**
- Payment batch generation
- Modulr integration (primary)
- BACS/manual fallback

**Migration:**
- Employee/YTD import
- Validation and reconciliation tools

### Out of Scope (V1)
- Multiple pension providers beyond NEST
- Full P11D/benefits module (basic architecture only)
- Telleroo payment integration (architected for future)
- Mobile-native apps (responsive web only)
- Advanced analytics/BI
- Accounting system integrations
- Public API

### Later Phase (Post-V1)
- Additional pension providers (People's Pension, NOW:Pensions)
- Full P11D and payrolled benefits (April 2027 readiness)
- Telleroo integration
- Custom report builder
- Advanced leave (irregular hours, TOIL)
- SSO (SAML/OIDC)
- Public API

## Key Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| HMRC conformance delays | Medium | Critical | Early engagement with HMRC, flexible architecture, sandbox testing from day one |
| Payroll calculation errors | Low | Critical | Extensive unit tests, HMRC reference data, parallel run validation |
| Payment provider integration issues | Medium | High | Multiple provider architecture, manual fallback, early API testing |
| Multi-tenant data isolation breach | Low | Critical | Security-first architecture, penetration testing, audit logging |
| Market/competitive response | Medium | Medium | Focus on bureau workflow differentiation, speed to market |
| Regulatory changes mid-build | Medium | Medium | Configurable tax tables, annual uprating architecture |

## Success Metrics

1. **Compliance:** Pass HMRC software developer conformance testing
2. **Accuracy:** 100% match with reference payroll calculations for test scenarios
3. **Pilot Validation:** Accountancy firm successfully processes live payrolls for 5+ clients
4. **Performance:** Payroll calculation for 100 employees completes in <5 seconds
5. **Adoption:** 3+ pilot bureaux actively using platform within 6 months of launch

## Verdict Rationale

**GO** — This is a well-understood domain with clear market gaps and proven technical feasibility. The accountancy firm pilot commitment provides immediate commercial validation and de-risks adoption. The phased MVP approach balances time-to-market with credible replacement capability.

**Critical success factors:**
- HMRC conformance process must start early and track closely
- Strict scope discipline on V1 (resist feature creep)
- Comprehensive test coverage for calculations
- Security-first multi-tenant architecture
- Pilot bureau feedback loop from month 4 onwards

**Next steps:** Proceed to strategic vision and module specification.
