# UK Bureau Payroll Platform — Architecture & Patterns

**Date:** 2026-04-08
**Version:** 1.0
**Research Path:** Greenfield

---

## Stack Overview

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | React 18 + TypeScript + Vite | Modern, typed, fast build, large ecosystem |
| UI Library | Tailwind CSS + Headless UI | Utility-first, accessible, customizable |
| State Management | React Query (server), Zustand (client) | Proven patterns for server state and local state |
| Backend | Python 3.12 + FastAPI | Type hints, async, auto-generated OpenAPI, HMRC XML handling |
| Database | PostgreSQL 16 | ACID compliance, JSON support, row-level security |
| Cache | Redis | Sessions, rate limiting, calculation caching |
| Queue | Celery + Redis | Background jobs, HMRC polling, report generation |
| Auth | JWT + python-jose + passlib | Stateless, industry standard, MFA support |
| Hosting | AWS (London region) | UK data residency, compliance, managed services |
| Container | Docker + ECS Fargate | Scalable, managed, infrastructure as code |
| Storage | S3 | Document storage, backups, exports |
| CDN | CloudFront | Static assets, edge caching |
| CI/CD | GitHub Actions | Integrated with repo, familiar toolchain |

---

## Project Structure

```
payroll-platform/
├── frontend/                          # React SPA
│   ├── src/
│   │   ├── components/               # Reusable UI components
│   │   ├── features/                 # Feature-based modules
│   │   │   ├── payroll/
│   │   │   ├── employees/
│   │   │   ├── reports/
│   │   │   └── auth/
│   │   ├── hooks/                    # Custom React hooks
│   │   ├── lib/                      # Utilities, API clients
│   │   ├── stores/                   # Zustand stores
│   │   └── types/                    # TypeScript types
│   ├── public/
│   └── package.json
├── backend/                           # FastAPI application
│   ├── app/
│   │   ├── api/                      # API routes
│   │   │   ├── v1/
│   │   │   │   ├── payroll.py
│   │   │   │   ├── employees.py
│   │   │   │   ├── hmrc.py
│   │   │   │   └── auth.py
│   │   ├── core/                     # Config, security, logging
│   │   ├── models/                   # SQLAlchemy models
│   │   ├── schemas/                  # Pydantic schemas
│   │   ├── services/                 # Business logic
│   │   │   ├── payroll_calculator/
│   │   │   ├── hmrc_submitter/
│   │   │   └── pension_assessor/
│   │   ├── integrations/             # External APIs
│   │   │   ├── hmrc/
│   │   │   ├── nest/
│   │   │   └── modulr/
│   │   └── tasks/                    # Celery background tasks
│   ├── alembic/                      # Database migrations
│   ├── tests/
│   └── pyproject.toml
├── infrastructure/                    # Terraform/IaC
│   ├── terraform/
│   └── scripts/
└── docs/                             # Documentation
```

---

## Code Patterns

### Naming Conventions
- **Files:** snake_case.py, PascalCase.tsx
- **Components:** PascalCase (e.g., `PayrollCalculator.tsx`)
- **Functions:** snake_case (Python), camelCase (TypeScript)
- **Types:** PascalCase with descriptive names (e.g., `PayRunStatus`, `TaxCalculation`)
- **Database tables:** snake_case, plural (e.g., `pay_runs`, `employees`)

### Error Handling
- **Backend:** Structured HTTP exceptions with error codes
  ```python
  class PayrollError(HTTPException):
      def __init__(self, code: str, message: str, details: dict = None):
          super().__init__(status_code=400, detail={"code": code, "message": message, "details": details})
  ```
- **Frontend:** React Error Boundaries + toast notifications
- **All errors logged with:** user_id, trace_id, timestamp, stack trace

### Validation
- **Input:** Pydantic schemas (backend), Zod (frontend)
- **Business rules:** Domain validators in services
- **HMRC schemas:** XML validation against XSD
- **Boundary validation:** API middleware

### State Management
- **Server state:** React Query with caching, background refetch
- **Client state:** Zustand for auth, UI preferences
- **Form state:** React Hook Form with validation
- **URL state:** React Router for filters, pagination

---

## Data Architecture

### Schema Design Principles
- Multi-tenant with `employer_id` or `bureau_id` on all tenant-scoped tables
- Immutable pay run records (versioning for corrections)
- Soft deletes for audit compliance
- JSONB for flexible metadata, strict schemas for core data

### Migration Strategy
- Alembic for schema migrations
- Migrations run in CI/CD before deployment
- Backward-compatible migrations (no breaking changes in single deploy)
- Data migrations separate from schema migrations

### Tenant Isolation
- **Row-Level Security (RLS):** PostgreSQL RLS policies enforce tenant boundaries
- **Application layer:** All queries filtered by tenant context
- **API layer:** JWT token includes tenant scope, middleware validates

### Key Tables
```sql
-- Tenancy
bureaus, employers, paye_schemes

-- People
employees, employments, subcontractors

-- Payroll
pay_periods, pay_runs, payslips, pay_elements

-- Compliance
hmrc_submissions, cis_returns, pension_enrolments

-- Operations
tasks, exceptions, audit_logs

-- Security
users, roles, permissions, sessions
```

---

## API Design

### Endpoint Conventions
- **Base:** `/api/v1/`
- **Resources:** Plural nouns (e.g., `/employees`, `/pay-runs`)
- **Actions:** POST for create, PATCH for update, POST for actions
- **Versioning:** URL versioning (v1, v2)

### Auth Middleware
```python
async def require_auth(request: Request) -> User:
    token = extract_bearer_token(request)
    payload = jwt.decode(token, SECRET_KEY)
    user = await get_user(payload['sub'])
    if not user or user.status != 'active':
        raise Unauthorized()
    return user

async def require_permission(permission: str):
    def checker(user: User = Depends(require_auth)):
        if not user.has_permission(permission):
            raise Forbidden()
    return checker
```

### Error Shapes
```json
{
  "error": {
    "code": "PAYROLL_CALCULATION_ERROR",
    "message": "Failed to calculate payroll",
    "details": {
      "employee_id": "uuid",
      "field": "tax_code",
      "issue": "Invalid format"
    },
    "trace_id": "abc123",
    "timestamp": "2026-04-28T10:30:00Z"
  }
}
```

---

## Testing Strategy

| Level | What to Test | Tools | Coverage Target |
|-------|-------------|-------|-----------------|
| Unit | Calculation functions, validators | pytest | 90% |
| Integration | API endpoints, database queries | pytest + TestClient | 80% |
| E2E | Critical user journeys | Playwright | Key flows |
| Compliance | HMRC reference calculations | Custom test packs | 100% |

### HMRC Compliance Testing
- HMRC provides test data packs for each tax year
- Automated test suite validates calculations against reference
- Must pass before any production deployment
- Annual regression testing for tax year changes

---

## CI/CD Pipeline

| Gate | Tool | Blocking? |
|------|------|-----------|
| Lint | Ruff (Python), ESLint (TS) | Yes |
| Type-check | mypy, tsc | Yes |
| Unit tests | pytest, vitest | Yes (90% pass) |
| Integration tests | pytest | Yes |
| Security scan | bandit, npm audit | Yes (no critical) |
| Build | Docker | Yes |
| Deploy staging | Terraform + ECS | Yes (manual approval for prod) |

### Deployment Strategy
- Blue-green deployment for zero-downtime
- Feature flags for gradual rollout
- Database migrations run before app deployment
- Rollback plan tested monthly

---

## Security Posture

### Authentication
- JWT access tokens (15 min expiry)
- Refresh tokens (7 days, rotating)
- MFA required for all bureau staff (TOTP)
- Password policy: 12+ chars, complexity, breach check

### Authorization
- RBAC with granular permissions
- Resource-level access control (employer-scoped)
- Field-level restrictions (bank details, NI numbers)
- Segregation of duties enforced

### Input Validation
- Schema validation at API boundary
- SQL injection prevention (parameterized queries, ORM)
- XSS prevention (output encoding, CSP headers)
- File upload restrictions (type, size, virus scan)

### Secrets Management
- AWS Secrets Manager for production
- .env files for local (never committed)
- Database credentials rotated quarterly
- API keys scoped and audited

### Logging & Monitoring
- Structured JSON logging
- Sensitive fields masked (PII redaction)
- Audit log append-only
- Security events alerted (failed logins, permission changes)
- CloudWatch + Datadog for monitoring

---

## Integration Architecture

### HMRC Gateway
- XML submissions via HTTPS
- Connection pooling with retry logic
- Polling for acknowledgements
- Evidence retention in S3

### NEST API
- REST API for contributions
- OAuth2 authentication
- Async submission with status polling
- File fallback for API outages

### Modulr
- REST API for payment initiation
- Webhook callbacks for status updates
- Idempotency keys for duplicate prevention
- Reconciliation polling

---

## Scalability Considerations

### Horizontal Scaling
- Stateless API servers (ECS Fargate)
- Database read replicas for reporting
- Redis cluster for session sharing
- CDN for static assets

### Performance Targets
- Payroll calc (100 employees): <5s
- API response time (p99): <200ms
- Dashboard load: <2s
- Report generation: <10s (async for large)

### Resource Limits
- Max 1000 employees per pay run (initial)
- Max 500 concurrent users per bureau
- Max 10MB payload for API requests
- Max 1000 records per page

---

## Compliance Architecture

### GDPR
- UK/EU data residency (AWS London)
- Data minimization (collect only required)
- DSAR export capability
- Retention policies with automated deletion
- Privacy by design

### HMRC Compliance
- Immutable audit logs
- Submission evidence retention (6 years)
- Calculation snapshots
- Annual uprating via configuration
- HMRC conformance testing

### Security Standards
- SOC 2 Type II roadmap
- ISO 27001 alignment
- Regular penetration testing
- Vulnerability management program
