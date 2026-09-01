# Payroll MVP

Minimum viable product for UK payroll processing, in planning. No application code
exists yet; `docs/` is the whole repo and the source of truth.

Precedence: user rules, then this file, then the nearest descendant `CLAUDE.md` or path
rule, which wins for its own folder.

## Principles

Follow KISS, DRY, YAGNI, SOLID, and Tell-Don't-Ask as written in
`~/.claude/rules/code-structure.md`: the simplest structure that passes the tests, reuse
before writing, nothing built for a need the task does not name.

## Project

- Purpose: payroll processing MVP.
- Source-of-truth docs: `docs/00-discovery.md`, `docs/01-strategic-plan.md`, the
  `docs/02-*` specs (core payroll, HMRC submissions, pension auto-enrolment, bureau
  operations, employee and employer portals, payments, CIS, identity/access,
  audit/compliance, reporting documents, migration/onboarding),
  `docs/08-architecture-and-patterns.md`, `docs/requirements/`, `docs/stories/`.
- Critical domains: payroll calculation, HMRC submissions, pensions, CIS, payments,
  identity and access, audit and compliance.

## Stack

Not established. Ask before adding dependencies, schema, infrastructure, CI, or external
integrations.

## Commands

No build, lint, typecheck, or test commands exist yet. Baseline checks:

```bash
git status --short
git diff --check
```

## Architecture

```
docs/00-discovery.md                 discovery
docs/01-strategic-plan.md            strategic plan
docs/02-*.md                         per-domain specs
docs/08-architecture-and-patterns.md architecture and patterns
docs/requirements/                   requirements
docs/stories/                        stories and slices
```

## Domain gates

- Payroll, tax, pension, and CIS semantics are statutory: confirm against the spec and
  the governing HMRC rule before implementing, and ask when the spec is silent.

## Git and PR

- Protected branches: `main`, `dev`. Base branch for work: `dev`.
