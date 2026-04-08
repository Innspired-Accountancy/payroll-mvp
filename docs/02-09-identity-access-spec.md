# Module: Identity & Access Management

**Parent:** UK Bureau Payroll Platform Strategic Plan
**Owner:** Backend/Security Team
**Priority:** P0 (Critical Path)

---

## Purpose

Provide comprehensive role-based access control (RBAC) with multi-tenant scoping, multi-factor authentication, segregation of duties enforcement, and secure user lifecycle management across bureau staff, client users, and employees.

---

## User Journeys

### Journey 1: Bureau User Login
1. **User** navigates to bureau login page
2. User enters email and password
3. System validates credentials
4. System requires MFA (TOTP/SMS)
5. User enters MFA code
6. System validates session, sets JWT tokens
7. System redirects to dashboard with role-appropriate view

### Journey 2: Assign Role to User
1. **Bureau Owner** navigates to User Management
2. Owner clicks "Invite User"
3. Owner enters email and selects role (Payroll Manager/Processor/Reviewer)
4. Owner assigns employer access scope (all/specific clients)
5. System sends invitation email
6. User accepts invitation and sets password
7. System activates user with assigned permissions

### Journey 3: Client Portal User Setup
1. **Bureau Admin** navigates to Client Users
2. Admin selects employer
3. Admin clicks "Add Portal User"
4. Admin enters email, name, selects role (Admin/Manager)
5. System sends invitation to client user
6. Client user sets up account with MFA
7. Client user can only access their employer data

### Journey 4: Segregation Check
1. **System** evaluates payment approval request
2. System checks approver is not same as preparer
3. System checks approver has payment_approve permission
4. System logs segregation compliance
5. If violation detected, system blocks and alerts

---

## Data Models

### Entity: User
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| email | String | Yes | Unique login email |
| first_name | String | Yes | First name |
| last_name | String | Yes | Last name |
| user_type | Enum | Yes | bureau/client/employee |
| bureau_id | UUID | No | FK to Bureau (bureau users) |
| employer_id | UUID | No | FK to Employer (client/employee users) |
| employee_id | UUID | No | FK to Employee (employee users) |
| status | Enum | Yes | active/inactive/pending |
| mfa_enabled | Boolean | Yes | MFA status |
| mfa_secret | String | No | Encrypted TOTP secret |
| last_login | DateTime | No | Last successful login |
| password_changed_at | DateTime | No | Password last changed |
| failed_login_attempts | Int | Yes | For lockout logic |
| locked_until | DateTime | No | Account lockout expiry |

### Entity: Role
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| bureau_id | UUID | No | FK to Bureau (null for system roles) |
| name | String | Yes | Role name |
| description | String | Yes | Role description |
| permissions | JSONB | Yes | List of permissions |
| is_system | Boolean | Yes | System-defined vs custom |

### Entity: UserRoleAssignment
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | UUID | Yes | Primary key |
| user_id | UUID | Yes | FK to User |
| role_id | UUID | Yes | FK to Role |
| scope_type | Enum | Yes | bureau/employer/employee |
| scope_id | UUID | Yes | ID of scoped entity |
| assigned_by | UUID | Yes | FK to User who assigned |
| assigned_at | DateTime | Yes | Assignment timestamp |

### Entity: Permission
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | String | Yes | Permission code (e.g., payroll:view) |
| name | String | Yes | Human-readable name |
| description | String | Yes | Description |
| resource | String | Yes | Resource type |
| action | String | Yes | Action (view/create/edit/approve/delete) |

---

## Standard Roles

### Bureau Roles
| Role | Permissions | Scope |
|------|-------------|-------|
| Platform Admin | All permissions | System |
| Bureau Owner | All bureau permissions | Bureau |
| Payroll Manager | payroll:*, employee:*, reports:* | Assigned employers |
| Payroll Processor | payroll:view,payroll:create,payroll:edit | Assigned employers |
| Reviewer | payroll:view, payroll:approve | Assigned employers |
| Approver | payroll:approve, payments:approve | Assigned employers |
| CIS Operator | cis:* | Assigned employers |

### Client Portal Roles
| Role | Permissions |
|------|-------------|
| Client Admin | employee:view,employee:create,variable_pay:submit,payroll:approve |
| Client Manager | employee:view,reports:view,payroll:approve |
| Client Viewer | employee:view,reports:view |

### Employee Role
| Permission | Description |
|------------|-------------|
| payslip:view_own | View own payslips only |
| leave:request | Request leave |
| profile:edit_own | Edit own personal details |

---

## API Contracts

### POST /api/v1/auth/login
Authenticate user.

**Input:**
```json
{
  "email": "user@example.com",
  "password": "********"
}
```

**Output:**
```json
{
  "access_token": "jwt-token",
  "refresh_token": "refresh-token",
  "expires_in": 3600,
  "mfa_required": true
}
```

### POST /api/v1/auth/mfa/verify
Verify MFA code.

**Input:**
```json
{
  "code": "123456"
}
```

**Output:**
```json
{
  "access_token": "jwt-token",
  "user": {
    "id": "uuid",
    "name": "John Smith",
    "roles": ["payroll_manager"]
  }
}
```

### GET /api/v1/users/{id}/permissions
Get user permissions.

**Output:**
```json
{
  "user_id": "uuid",
  "permissions": [
    {
      "permission": "payroll:view",
      "scope": {"type": "employer", "id": "uuid"}
    },
    {
      "permission": "payroll:approve",
      "scope": {"type": "employer", "id": "uuid"}
    }
  ]
}
```

---

## Integration Points

| External System | Purpose | Integration Method |
|-----------------|---------|-------------------|
| All Modules | Permission checks | Shared library |
| Audit Service | Login/access logging | Events |
| Notification Service | 2FA/magic links | Events |

---

## Non-Functional Requirements

### Security
- Password policy: min 12 chars, complexity required
- Account lockout after 5 failed attempts
- MFA required for all bureau staff
- Session timeout: 30 minutes idle
- Secure cookie settings
- CSRF protection

### Compliance
- Segregation of duties enforcement
- Regular access reviews
- Audit trail of all permission changes
- Least privilege principle
