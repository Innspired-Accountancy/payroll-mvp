# Slice a: Employee Data Model and API

**Story:** story-01-employee-mgmt
**Epic:** epic-01-core-payroll
**Effort:** M
**Dependencies:** None

---

## Goal

Create the employee data model and REST API endpoints for CRUD operations with proper validation and tenant isolation.

---

## Decision Checklist

- [x] All libraries/packages named: Drizzle ORM 0.30.x, Zod 3.22.x, Vitest
- [x] Data contracts defined: EmployeeCreate, EmployeeUpdate, EmployeeResponse Zod schemas
- [x] tRPC procedures: employees.create, employees.getById, employees.update, employees.list
- [x] Tenant isolation: employerId foreign key + tRPC context check
- [x] Error scenarios: TRPCError with codes BAD_REQUEST, NOT_FOUND, CONFLICT
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-01-core-payroll-spec.md — Employee and employment data section
- 08-architecture-and-patterns.md — Multi-tenant isolation

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/db/schema/employees.ts` | create | Drizzle table schema |
| `src/server/routers/employees.ts` | create | tRPC procedures |
| `src/lib/validation/employees.ts` | create | Zod schemas (shared) |
| `src/app/(bureau)/employees/page.tsx` | create | Employee list UI |
| `src/app/(bureau)/employees/[id]/page.tsx` | create | Employee detail UI |
| `src/tests/employees.test.ts` | create | Vitest tests |

---

## Responsibilities

1. Define Employee model with all payroll-relevant fields
2. Implement CRUD API endpoints
3. Validate NI number format
4. Enforce tenant isolation
5. Return proper error responses

---

## Contracts

### employees.create
- **Method:** tRPC mutation `employees.create`
- **Input:** EmployeeCreate Zod schema
- **Output:** Employee (inserted record)
- **Errors:** BAD_REQUEST (validation), CONFLICT (duplicate NI)
- **Auth:** Protected procedure with employee:create permission

### EmployeeCreate Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| first_name | str | Yes | First name |
| last_name | str | Yes | Last name |
| date_of_birth | date | Yes | DOB |
| ni_number | str | Yes | National Insurance |
| address | Address | Yes | Full address |
| start_date | date | Yes | Employment start |

---

## Business Rules & Invariants

1. NI number must match HMRC format: AB123456C
2. Email must be unique within employer
3. employer_id is mandatory (tenant isolation)
4. All dates must be valid (not future for DOB)

---

## Edge Cases

1. **Duplicate NI number** — Return 409 with existing employee reference
2. **Invalid NI format** — Return 400 with validation error
3. **Future start date** — Allow (future-dated employment)
4. **Empty optional fields** — Store as null, not empty string

---

## Tests

### test_employees.py
- Create employee with valid data
- Create employee with invalid NI (expect 400)
- Create employee with duplicate NI (expect 409)
- Get employee by ID (expect 200)
- Get non-existent employee (expect 404)
- Update employee (expect 200)
- Delete/terminate employee (expect 200)

---

## Verification

```bash
cd backend
pytest tests/test_employees.py -v
mypy app/models/employee.py app/schemas/employee.py
ruff check app/api/v1/employees.py
```

---

## Source Sections

- epic-01-core-payroll/epic-plan.md § Data Contracts → EmployeeCreate schema
- 02-01-core-payroll-spec.md § Employee payroll data → Field requirements
