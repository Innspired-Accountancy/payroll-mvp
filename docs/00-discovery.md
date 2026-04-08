# Discovery: Payroll MVP

**Date:** 2026-04-08
**Verdict:** GO

## Problem Statement

Small businesses (5-50 employees) struggle with payroll processing due to expensive enterprise solutions and complex manual calculations. Current options are either too costly (ADP, Gusto at $40+/month + per-employee fees) or require significant accounting expertise to execute correctly.

## Target Users

**Primary: Small Business Owners / Office Managers**
- 5-50 employees
- Limited HR/accounting staff
- Need simple, accurate payroll without enterprise complexity
- Price-sensitive but compliance-conscious

**Secondary: Accountants serving small business clients**
- Need efficient tools for multiple small clients
- Require accurate tax calculations and reporting

**Pain Points:**
1. Tax calculation errors lead to penalties
2. Time spent on manual calculations
3. Compliance anxiety (federal, state, local taxes)
4. Employee self-service for pay stubs/history
5. Year-end reporting (W-2s, 1099s)

## Existing Alternatives

| Solution | Price | Complexity | Limitations |
|----------|-------|------------|-------------|
| ADP/Gusto | $40-150+/mo | Medium | Expensive for small teams |
| QuickBooks Payroll | $45+/mo | High | Requires QB ecosystem |
| Manual (Excel) | Free | Very High | Error-prone, no compliance guardrails |
| Wave Payroll | $20+/mo | Low | Limited tax support in many states |

**Why insufficient:** Existing affordable options lack robust tax calculation. Manual methods create compliance risk. Enterprise tools are overbuilt and overpriced for small businesses.

## Feasibility Assessment

**Technical:** HIGH — Payroll calculations are well-documented, deterministic algorithms. No novel technology required. Risk lies in accuracy and keeping tax tables current.

**Resource:** MEDIUM — Single developer can build MVP in 4-6 weeks. Tax rate maintenance is ongoing operational cost.

**Timeline:** MEDIUM — 6-8 weeks for functional MVP with basic federal/state tax support for a limited set of states.

## Scope

### In Scope (MVP)
- Employee management (add, edit, terminate)
- Basic payroll calculation (gross pay, federal taxes, Social Security, Medicare)
- Support for salaried and hourly employees
- Pay period management (weekly, biweekly, semimonthly, monthly)
- Pay stub generation (PDF)
- Simple deductions (fixed amount, percentage)
- Basic reporting (payroll register, employee summary)
- Web-based UI for administration

### Out of Scope (MVP)
- State/local tax calculations beyond 5 pilot states
- Direct deposit integration
- Benefits administration (health insurance, 401k)
- Time tracking / attendance
- Multi-company support
- Accountant portal
- Mobile apps
- W-2/1099 generation (post-MVP feature)
- International payroll
- Contractor payments (1099)

## Key Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Tax calculation errors | Medium | Critical | Extensive unit tests, reference against IRS publications, quarterly reconciliation reports |
| State tax complexity | High | High | Start with 5 most populous states only, clear roadmap for expansion |
| Data security (PII/SSN) | Medium | Critical | Encryption at rest/in transit, secure hosting, audit logging |
| Regulatory changes | High | Medium | Architecture allows tax table updates without code changes |
| User adoption | Medium | Medium | Focus on UX simplicity, clear value prop vs alternatives |

## Success Metrics

1. **Accuracy:** 100% match with reference payroll calculations for test scenarios
2. **Usability:** New user can process first payroll in <30 minutes without documentation
3. **Performance:** Payroll calculation for 50 employees completes in <5 seconds
4. **Adoption:** (Post-launch) 10+ active companies processing payroll within 3 months

## Verdict Rationale

**GO** — This is a well-understood domain with clear user needs and feasible technical scope. The MVP can deliver genuine value with limited complexity. Key risks (tax accuracy, security) are manageable with proper testing and architecture. The market need is validated by existing solutions' pricing gaps.

**Success factors:**
- Strict scope discipline (resist feature creep)
- Comprehensive test coverage for calculations
- Security-first architecture from day one
- Clear path to revenue (freemium or affordable SaaS model)

**Next steps:** Proceed to strategic vision and architecture planning.
