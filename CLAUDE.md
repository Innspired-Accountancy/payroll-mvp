# Project: payroll-mvp

## Critical Behavior (Project-Level)
- Inherits the organizational Critical Behavior rules from `~/.claude/claude.md`; project rules may add, but must not relax, those hard rules without an explicit, approved exception.
- Autocompact is enabled and handles context management automatically. Support it by keeping responses focused and ensuring critical decisions, file paths, and open questions survive compaction.
- Stop-the-line triggers (project additions):
  - [ ] Domain-specific quality gates (e.g., accessibility/i18n/performance budgets)
  - [ ] Data integrity or migration risks unique to this project
  - [ ] Compliance or audit requirements
- Always ask before (project additions):
  - [ ] New dependencies specific to this project's architecture
  - [ ] Schema/API changes for shared services
  - [ ] CI/CD, infra, or security policy changes
- Never do:
  - [ ] Placeholders, stubs, or TODOs in place of working code
  - [ ] Bypassing failing checks or disabling lint/tests without approved ticket + timebox
  - [ ] Destructive operations without explicit approval, backups, and rollback plan

## Scope
Payroll MVP - A minimum viable product for payroll processing.

## Stack
### Languages/Runtimes + Versions
- [ ] Primary language:
- [ ] Runtime version:
- [ ] Secondary languages:

### Frameworks/Libraries
- [ ] Web framework:
- [ ] Testing framework:
- [ ] UI library:
- [ ] State management:
- [ ] Database/ORM:

### Package Manager/Build Tools
- [ ] Package manager:
- [ ] Build tool:
- [ ] Bundler:
- [ ] Task runner:

## Commands
| Command | Description |
|---------|-------------|
| `git status` | Check git status |
| `git branch` | List branches |

## Code Style
### Formatter + Config
- [ ] Formatter tool:
- [ ] Config file location:
- [ ] Pre-commit hooks:

### Linter + Rulesets
- [ ] Linter tool:
- [ ] Ruleset/config:
- [ ] Custom rules:

### Naming/Conventions
- [ ] File naming:
- [ ] Component structure:
- [ ] Variable conventions:
- [ ] Project-specific patterns:

## Testing
### Frameworks
- [ ] Unit test framework:
- [ ] Integration test tools:
- [ ] E2E test framework:

### Coverage Targets
- [ ] Line coverage target: _%
- [ ] Branch coverage target: _%
- [ ] Required coverage areas:
- [ ] Excluded from coverage:

### Integration/E2E Scope
- [ ] Test environment setup:
- [ ] Critical user paths:
- [ ] Performance benchmarks:

## CI/CD
### Required PR Checks
- [ ] Lint/format check
- [ ] Type check
- [ ] Unit tests
- [ ] Integration tests
- [ ] Build verification
- [ ] Security scan
- [ ] Custom checks:

### Branch Protection Rules
- [ ] Protected branches: `main`, `dev`
- [ ] Review requirements:
- [ ] Status checks:
- [ ] Merge restrictions:

### Deployment Strategy
- [ ] Environments:
- [ ] Deploy process:
- [ ] Rollback procedure:
- [ ] Feature flags:

## Dependencies
### Allowed/Blocked Dependencies
- [ ] Approved list:
- [ ] Blocked packages:
- [ ] Review process:
- [ ] Version pinning strategy:

## Security/Compliance
### Secrets Handling
- [ ] Secret management tool:
- [ ] Environment variables:
- [ ] Rotation policy:

### License Policy
- [ ] Allowed licenses:
- [ ] Prohibited licenses:
- [ ] Attribution requirements:

### Special Checks
- [ ] SAST tool:
- [ ] DAST tool:
- [ ] SBOM generation:
- [ ] Compliance scans:

## Project-Specific Instructions
Additional instructions or context specific to this project that Claude Code should know.

### Key Entry Points
- [ ] Main application:
- [ ] API endpoints:
- [ ] Background jobs:
- [ ] CLI tools:

### Critical Business Logic
- [ ] Core domains:
- [ ] Key algorithms:
- [ ] Data flows:
- [ ] Integration points:

### Known Issues/Tech Debt
- [ ] Areas to avoid:
- [ ] Planned refactors:
- [ ] Performance bottlenecks:
- [ ] Security considerations:

---
*Initialized via `/wf-init` on 2026-04-08*
