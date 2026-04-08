# UK Bureau Payroll Platform — Architecture & Patterns

**Date:** 2026-04-08 (Updated)  
**Version:** 1.1  
**Research Path:** Greenfield  
**Integration Context:** Practice Hub Beta (Next.js/tRPC/Drizzle stack)

---

## Stack Overview

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Full-Stack Framework | Next.js 14 (App Router) | React + API routes in one codebase, SSR for portals |
| Language | TypeScript | Shared types frontend/backend, HMRC XML still manageable |
| API Layer | tRPC | End-to-end type safety, integrates with existing Practice Hub |
| ORM | Drizzle | Type-safe SQL, PostgreSQL native, lightweight |
| Database | PostgreSQL 16 | ACID compliance, JSON support, familiar from Practice Hub |
| Auth | BetterAuth | Matches Practice Hub, supports MFA, RBAC ready |
| Queue | In-memory (BullMQ or manual) | Start simple, add Redis if needed |
| Hosting | VPS (Digital Ocean/Linode) | Simple, cost-effective, single server for MVP |
| Deployment | Docker Compose | Single container deployment, easy migration later |

**Integration Note:** This stack aligns with Practice Hub Beta (`/Users/josephstephenson-mouzo/Projects/03 - development/01 - practice hub/practice-hub-beta`) for eventual integration.

---

## Project Structure

```
payroll-platform/
├── src/
│   ├── app/                      # Next.js App Router
│   │   ├── (bureau)/             # Bureau portal routes
│   │   ├── (employer)/           # Employer portal routes  
│   │   ├── (employee)/           # Employee portal routes
│   │   ├── api/                  # API routes (webhooks, etc)
│   │   └── trpc/                 # tRPC router
│   ├── components/               # React components
│   │   ├── payroll/
│   │   ├── employees/
│   │   └── ui/
│   ├── lib/                      # Utilities
│   │   ├── db/                   # Drizzle schema & client
│   │   ├── auth/                 # BetterAuth config
│   │   ├── calculations/         # Tax/NIC engines
│   │   └── hmrc/                 # HMRC XML handling
│   ├── server/                   # tRPC procedures
│   │   ├── routers/
│   │   │   ├── payroll.ts
│   │   │   ├── employees.ts
│   │   │   ├── hmrc.ts
│   │   │   └── auth.ts
│   │   └── trpc.ts               # tRPC setup
│   └── types/                    # Shared TypeScript types
├── drizzle/                      # Database migrations
├── docker-compose.yml            # Local & VPS deployment
└── docs/                         # Documentation
```

---

## Code Patterns

### Naming Conventions
- **Files:** kebab-case.ts, PascalCase.tsx
- **Components:** PascalCase (e.g., `PayrollCalculator.tsx`)
- **Functions:** camelCase
- **Types/Interfaces:** PascalCase with descriptive names
- **Database tables:** snake_case, plural
- **tRPC procedures:** camelCase (e.g., `payroll.calculate()`)

### Error Handling
- **tRPC:** `TRPCError` with structured codes
  ```typescript
  throw new TRPCError({
    code: 'BAD_REQUEST',
    message: 'Invalid tax code format',
    cause: { field: 'taxCode', value: input.taxCode }
  });
  ```
- **Frontend:** React Error Boundaries + toast notifications
- **All errors logged with:** userId, traceId, timestamp

### Validation
- **Input:** Zod schemas (shared between frontend/backend via tRPC)
- **API layer:** tRPC context validation
- **HMRC XML:** XSD validation via `fast-xml-parser`
- **Database:** Drizzle schema constraints

### State Management
- **Server state:** tRPC React Query integration (caching, refetch)
- **Client state:** Zustand for auth, UI preferences
- **Form state:** React Hook Form + Zod resolver

---

## Data Architecture

### Schema Design Principles
- Multi-tenant with `employerId` or `bureauId` on all tenant-scoped tables
- Immutable pay run records (versioning for corrections)
- Drizzle schema with strict TypeScript types
- Soft deletes for audit compliance

### Migration Strategy
- Drizzle Kit for schema migrations
- Migrations run in CI/CD before deployment
- Backward-compatible migrations (no breaking changes)

### Tenant Isolation
- **Application layer:** All tRPC procedures filter by tenant context
- **Row-Level Security:** Optional PostgreSQL RLS for extra safety
- **tRPC middleware:** Validates user has access to requested resources

### Key Tables
```typescript
// Tenancy
bureaus, employers, payeSchemes

// People
employees, employments, subcontractors

// Payroll
payPeriods, payRuns, payslips, payElements

// Compliance
hmrcSubmissions, cisReturns, pensionEnrolments

// Operations
tasks, exceptions, auditLogs

// Security
users, roles, permissions
```

---

## API Design (tRPC)

### Router Structure
```typescript
// server/routers/payroll.ts
export const payrollRouter = router({
  // Queries
  getById: protectedProcedure
    .input(z.object({ id: z.string().uuid() }))
    .query(({ input, ctx }) => { ... }),
    
  // Mutations  
  calculate: protectedProcedure
    .input(CalculatePayrollInput)
    .mutation(({ input, ctx }) => { ... }),
    
  approve: protectedProcedure
    .input(z.object({ payRunId: z.string().uuid() }))
    .mutation(({ input, ctx }) => { ... }),
});
```

### Auth Middleware
```typescript
// server/trpc.ts
const protectedProcedure = t.procedure
  .use(isAuthed)  // BetterAuth session check
  .use(hasPermission('payroll:view'));  // RBAC check
```

### Error Shapes
```json
{
  "error": {
    "json": {
      "code": "BAD_REQUEST",
      "message": "Failed to calculate payroll",
      "data": {
        "employeeId": "uuid",
        "field": "taxCode",
        "issue": "Invalid format"
      }
    }
  }
}
```

---

## Testing Strategy

| Level | What to Test | Tools | Coverage Target |
|-------|-------------|-------|-----------------|
| Unit | Calculation functions, validators | Vitest | 90% |
| Integration | tRPC procedures, DB queries | Vitest + test DB | 80% |
| E2E | Critical user journeys | Playwright | Key flows |
| Compliance | HMRC reference calculations | Custom test packs | 100% |

### HMRC Compliance Testing
- HMRC provides test data packs for each tax year
- Automated test suite validates calculations
- Must pass before any production deployment

---

## CI/CD Pipeline

| Gate | Tool | Blocking? |
|------|------|-----------|
| Lint | ESLint + Prettier | Yes |
| Type-check | TypeScript | Yes |
| Unit tests | Vitest | Yes (90% pass) |
| Build | Next.js | Yes |
| Deploy | GitHub Actions → VPS | Manual approval for prod |

### Deployment Strategy
- **Development:** `docker-compose up` locally
- **Staging:** VPS with staging branch auto-deploy
- **Production:** VPS with manual promotion
- **Database:** PostgreSQL on same VPS (simpler for MVP)

---

## Security Posture

### Authentication
- BetterAuth sessions (cookie-based)
- MFA required for bureau staff (TOTP)
- Password policy: 12+ chars, complexity
- Session timeout: 30 minutes

### Authorization
- RBAC with granular permissions
- Resource-level access control (employer-scoped)
- tRPC middleware enforces permissions

### Input Validation
- Zod schemas at API boundary
- SQL injection prevention (Drizzle parameterized queries)
- XSS prevention (Next.js escapes by default)

### Secrets Management
- Environment variables (`.env` on VPS)
- Never commit secrets
- Database credentials via VPS environment

### Logging & Monitoring
- Structured logging (Winston/Pino)
- Sensitive fields masked
- Audit log append-only table
- Security events alerted

---

## Integration Architecture

### HMRC Gateway
- XML submissions via HTTPS
- Node.js `https` module or `axios`
- Polling for acknowledgements
- Evidence retention in filesystem (migrate to S3 later)

### NEST API
- REST API via `fetch`/`axios`
- OAuth2 or API key auth
- Async submission handling

### Modulr
- REST API for payments
- Webhook endpoint for status updates
- Idempotency keys for duplicate prevention

---

## Scalability Considerations (MVP Phase)

**Intentionally Simple for MVP:**
- Single VPS (2-4GB RAM, 2 vCPUs)
- PostgreSQL on same server
- No Redis (use in-memory queue or setTimeout)
- No CDN (serve from VPS)
- File storage on VPS disk (migrate to S3 later)

**When to Scale:**
- Move PostgreSQL to managed service when >100 concurrent users
- Add Redis for sessions/queue when background jobs grow
- Add S3 for document storage when disk fills
- Add CDN when global users join

**Performance Targets (MVP):**
- Payroll calc (100 employees): <5 seconds
- API response (p99): <500ms
- Page load: <2 seconds

---

## Compliance Architecture

### GDPR
- UK data residency (VPS in UK/EU region)
- Data minimization
- DSAR export capability
- Retention policies with automated deletion

### HMRC Compliance
- Immutable audit logs
- Submission evidence retention (6 years)
- Calculation snapshots
- Annual uprating via configuration

---

## Revised Timeline (Simpler Stack)

| Phase | Target | Deliverables |
|-------|--------|--------------|
| **Month 1** | Foundation | Next.js setup, Drizzle schema, auth, employee CRUD |
| **Month 2** | Payroll Core | Tax/NIC calculations, pay runs, payslips |
| **Month 3** | Compliance | HMRC FPS generation, NEST integration |
| **Month 4** | Portals | Bureau dashboard, employer portal, employee portal |
| **Month 5** | Payments & Polish | Modulr integration, bug fixes, testing |
| **Month 6** | Pilot | Deploy to your accountancy firm, iterate |

**Benefits of simpler stack:**
- Faster development (shared types, single codebase)
- Lower hosting costs (~$20-40/month vs $200+ for AWS)
- Easier debugging (single server)
- Easier migration path to Practice Hub integration
