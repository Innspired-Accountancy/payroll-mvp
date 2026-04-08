# Slice a: Payroll Schedule Model and API

**Story:** story-02-payroll-setup
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** None

---

## Goal

Create the payroll schedule data model and tRPC API for CRUD operations with proper validation and tenant isolation.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x, date-fns 3.x
- [x] Data contracts defined: PayrollScheduleCreate, PayrollScheduleUpdate, PayrollScheduleResponse Zod schemas
- [x] tRPC procedures: payrollSchedules.create, payrollSchedules.getById, payrollSchedules.update, payrollSchedules.list, payrollSchedules.delete
- [x] Tenant isolation: employerId foreign key + tRPC context check
- [x] Error scenarios: TRPCError with codes BAD_REQUEST, NOT_FOUND, CONFLICT
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — PayPeriod entity definition
- 08-architecture-and-patterns.md — Multi-tenant isolation

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/payroll-schedules.ts` | create | Drizzle table schema |
| `src/server/routers/payroll-schedules.ts` | create | tRPC procedures |
| `src/lib/validation/payroll-schedules.ts` | create | Zod schemas (shared) |
| `src/tests/payroll-schedules.test.ts` | create | Vitest tests |

---

## Responsibilities

1. Define PayrollSchedule model with frequency, anchor date, pay day
2. Implement CRUD API endpoints
3. Validate schedule does not overlap with existing
4. Enforce tenant isolation
5. Return proper error responses

---

## Contracts

### payrollSchedules.create
- **Method:** tRPC mutation `payrollSchedules.create`
- **Input:** PayrollScheduleCreate Zod schema
- **Output:** PayrollSchedule (inserted record)
- **Errors:** BAD_REQUEST (validation), CONFLICT (duplicate name)
- **Auth:** Protected procedure with payroll:write permission

### PayrollScheduleCreate Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| name | string | Yes | Schedule name (e.g., "Monthly Payroll") |
| frequency | enum | Yes | weekly, monthly, lunar, quarterly |
| anchor_date | date | Yes | First period start date |
| pay_day_offset | int | Yes | Days after period end for pay date |
| employer_id | uuid | Yes | FK to Employer |

---

## Business Rules & Invariants

1. Frequency must be one of: weekly, monthly, lunar, quarterly
2. Anchor date must be valid date in YYYY-MM-DD format
3. pay_day_offset must be >= 0 and <= 31
4. employer_id is mandatory (tenant isolation)
5. Schedule name must be unique within employer

---

## Edge Cases

1. **Duplicate schedule name** — Return 409 with existing schedule reference
2. **Invalid frequency value** — Return 400 with allowed values list
3. **Future anchor date** — Allow (future-dated schedule)
4. **Zero pay_day_offset** — Allow (pay date same as period end)

---

## Tests

### payroll-schedules.test.ts
- Create schedule with valid data
- Create schedule with invalid frequency (expect 400)
- Create schedule with duplicate name (expect 409)
- Get schedule by ID (expect 200)
- Get non-existent schedule (expect 404)
- Update schedule (expect 200)
- Delete schedule with no periods (expect 200)
- Delete schedule with periods (expect 409)

---

## Verification

```bash
npm run test:unit
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-01-core-payroll-spec.md § Data Models → PayPeriod entity
