# Slice a: JWT Token Generation and Claims

**Story:** story-03-jwt-implementation
**Epic:** epic-05-identity-access
**Effort:** M
**Dependencies:** story-02-password-auth

## Goal

Implement JWT access token and refresh token generation with proper claims structure, cryptographic signing using RS256, and secure key management.

## Decision Checklist

- [x] All libraries/packages named: `jose@5.x`, `crypto` (Node.js built-in)
- [x] SDK methods identified: `new SignJWT()`, `importPKCS8()`, `generateKeyPair()`
- [x] External service endpoints: N/A (internal signing)
- [x] Data contracts defined: JWT claims structure, token pair response
- [x] Configuration variables: `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`, `JWT_ACCESS_EXPIRY=3600`, `JWT_REFRESH_EXPIRY=604800`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-09-identity-access-spec.md:§ API Contracts → POST /api/v1/auth/login (token output)
- 02-09-identity-access-spec.md:§ API Contracts → POST /api/v1/auth/mfa/verify (token output)

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/jwt/index.ts` | create | JWT signing and verification functions |
| `src/lib/jwt/claims.ts` | create | Claims type definitions |
| `src/lib/jwt/keys.ts` | create | Key management utilities |
| `src/lib/jwt/tokens.ts` | create | Token generation functions |
| `scripts/generate-jwt-keys.ts` | create | Key generation script |

## Responsibilities
1. Generate RS256 key pair for JWT signing
2. Create access tokens with claims (sub, exp, iat, roles, tenant_id, scope)
3. Create refresh tokens with minimal claims (sub, jti, exp)
4. Secure key storage (private key in env, public key distributable)
5. Token ID (jti) generation for tracking and revocation

## Contracts

### JWT Claims Types
```typescript
// src/lib/jwt/claims.ts
export interface AccessTokenClaims {
  // Standard claims
  sub: string;        // User ID
  jti: string;        // Unique token ID
  iat: number;        // Issued at (unix timestamp)
  exp: number;        // Expiration (unix timestamp)
  iss: string;        // Issuer (app URL)
  aud: string;        // Audience (app URL)
  
  // Custom claims
  type: 'access';
  email: string;
  userType: 'bureau' | 'client' | 'employee';
  roles: string[];    // Role IDs
  permissions: string[]; // Permission IDs (cached)
  bureauId?: string;
  employerId?: string;
  employeeId?: string;
  sessionId: string;  // Link to session record
}

export interface RefreshTokenClaims {
  // Standard claims
  sub: string;        // User ID
  jti: string;        // Unique token ID
  iat: number;
  exp: number;
  iss: string;
  aud: string;
  
  // Custom claims
  type: 'refresh';
  sessionId: string;
  version: number;    // For token rotation
}

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
  accessTokenExpiresAt: Date;
  refreshTokenExpiresAt: Date;
}
```

### Key Management
```typescript
// src/lib/jwt/keys.ts
import { importPKCS8, importSPKI, generateKeyPair } from 'jose';

export async function loadPrivateKey(): Promise<CryptoKey> {
  const privateKeyPem = process.env.JWT_PRIVATE_KEY!;
  
  return importPKCS8(privateKeyPem, 'RS256');
}

export async function loadPublicKey(): Promise<CryptoKey> {
  const publicKeyPem = process.env.JWT_PUBLIC_KEY!;
  
  return importSPKI(publicKeyPem, 'RS256');
}

export async function generateKeyPair(): Promise<{ privateKey: string; publicKey: string }> {
  const { privateKey, publicKey } = await generateKeyPair('RS256', {
    modulusLength: 2048,
  });
  
  // Export to PEM format
  const privateKeyPem = await crypto.subtle.exportKey('pkcs8', privateKey)
    .then(buf => `-----BEGIN PRIVATE KEY-----\n${Buffer.from(buf).toString('base64')}\n-----END PRIVATE KEY-----`);
  
  const publicKeyPem = await crypto.subtle.exportKey('spki', publicKey)
    .then(buf => `-----BEGIN PUBLIC KEY-----\n${Buffer.from(buf).toString('base64')}\n-----END PUBLIC KEY-----`);
  
  return { privateKey: privateKeyPem, publicKey: publicKeyPem };
}
```

### Token Generation
```typescript
// src/lib/jwt/tokens.ts
import { SignJWT } from 'jose';
import { randomUUID } from 'crypto';
import { loadPrivateKey } from './keys';
import type { AccessTokenClaims, RefreshTokenClaims, TokenPair } from './claims';
import type { User, UserRoleAssignment } from '@/lib/db/schema';

const ACCESS_TOKEN_EXPIRY = 60 * 60; // 1 hour in seconds
const REFRESH_TOKEN_EXPIRY = 60 * 60 * 24 * 7; // 7 days in seconds
const ISSUER = process.env.BETTER_AUTH_URL!;
const AUDIENCE = process.env.BETTER_AUTH_URL!;

export async function generateTokenPair(
  user: User,
  roles: UserRoleAssignment[],
  sessionId: string
): Promise<TokenPair> {
  const privateKey = await loadPrivateKey();
  const now = Math.floor(Date.now() / 1000);
  
  // Extract permission IDs from roles
  const permissionIds = await getPermissionsForRoles(roles.map(r => r.roleId));
  
  // Generate access token
  const accessTokenJti = randomUUID();
  const accessTokenClaims: Omit<AccessTokenClaims, 'iat' | 'exp'> = {
    sub: user.id,
    jti: accessTokenJti,
    iss: ISSUER,
    aud: AUDIENCE,
    type: 'access',
    email: user.email,
    userType: user.userType as 'bureau' | 'client' | 'employee',
    roles: roles.map(r => r.roleId),
    permissions: permissionIds,
    bureauId: user.bureauId ?? undefined,
    employerId: user.employerId ?? undefined,
    employeeId: user.employeeId ?? undefined,
    sessionId,
  };
  
  const accessToken = await new SignJWT(accessTokenClaims)
    .setProtectedHeader({ alg: 'RS256', typ: 'JWT' })
    .setIssuedAt(now)
    .setExpirationTime(now + ACCESS_TOKEN_EXPIRY)
    .sign(privateKey);
  
  // Generate refresh token
  const refreshTokenJti = randomUUID();
  const refreshTokenClaims: Omit<RefreshTokenClaims, 'iat' | 'exp'> = {
    sub: user.id,
    jti: refreshTokenJti,
    iss: ISSUER,
    aud: AUDIENCE,
    type: 'refresh',
    sessionId,
    version: 1, // Incremented on rotation
  };
  
  const refreshToken = await new SignJWT(refreshTokenClaims)
    .setProtectedHeader({ alg: 'RS256', typ: 'JWT' })
    .setIssuedAt(now)
    .setExpirationTime(now + REFRESH_TOKEN_EXPIRY)
    .sign(privateKey);
  
  return {
    accessToken,
    refreshToken,
    accessTokenExpiresAt: new Date((now + ACCESS_TOKEN_EXPIRY) * 1000),
    refreshTokenExpiresAt: new Date((now + REFRESH_TOKEN_EXPIRY) * 1000),
  };
}

async function getPermissionsForRoles(roleIds: string[]): Promise<string[]> {
  // Query database for permissions associated with these roles
  // Return flattened unique permission IDs
  const roles = await db.query.roles.findMany({
    where: inArray(roles.id, roleIds),
    columns: { permissions: true },
  });
  
  const permissionSet = new Set<string>();
  for (const role of roles) {
    for (const perm of role.permissions) {
      permissionSet.add(perm);
    }
  }
  
  return Array.from(permissionSet);
}
```

### Environment Configuration
```bash
# .env.example additions
JWT_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC...
-----END PRIVATE KEY-----"

JWT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----"

JWT_ACCESS_EXPIRY="3600"
JWT_REFRESH_EXPIRY="604800"
```

## Business Rules & Invariants
1. Access tokens expire after 1 hour (3600 seconds)
2. Refresh tokens expire after 7 days (604800 seconds)
3. RS256 algorithm required (asymmetric for distributed verification)
4. All tokens have unique JTI for tracking and revocation
5. Session ID links tokens to session record for cleanup

## Edge Cases
1. **Key loading failure** — Throw at startup, prevent app from running
2. **Clock skew** — Add 60-second leeway in verification (handled in slice-b)
3. **Token size limit** — Monitor claims size, compress if needed
4. **Concurrent token generation** — Each gets unique JTI, no conflict
5. **Private key compromise** — Document key rotation procedure

## Tests

### src/lib/jwt/tokens.test.ts
- `should generate valid access token`: Verifies JWT structure and signing
- `should generate valid refresh token`: Verifies refresh token format
- `should have correct expiration times`: Verifies exp claims
- `should include all required claims`: Verifies claim completeness
- `should generate unique JTIs`: Verifies uniqueness
- `should load private key correctly`: Verifies key loading

## Verification
```bash
# Generate keys for testing
npm run generate:jwt-keys

# Run tests
npm run test:unit src/lib/jwt/tokens.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-09-identity-access-spec.md § API Contracts → Token output structures
