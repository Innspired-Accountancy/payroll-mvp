# Slice a: HTTP Client Configuration and Security

**Story:** story-03-hmrc-gateway
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** None

---

## Goal

Configure secure HTTPS client for HMRC Gateway communication with TLS 1.2 minimum, proper certificate validation, fraud prevention headers, and connection pooling for performance.

---

## Decision Checklist

- [x] All libraries/packages named: Node.js https module, undici 6.x (HTTP client)
- [x] SDK methods identified: undici.request(), Agent for connection pooling
- [x] External service endpoints: HMRC PAYE Online Gateway (test/live)
- [x] Data contracts defined: HMRCHeaders, GatewayConfig interfaces
- [x] Configuration: HMRC_GATEWAY_URL, TLS_CERT_PATH, FRAUD_PREVENTION_HEADERS
- [x] Error scenarios: TLS handshake failure, certificate validation, timeout
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § Integration Points → HMRC Gateway
- HMRC RTI Security Guidelines

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/gateway/client.ts` | create | HTTPS client configuration |
| `src/lib/hmrc/gateway/config.ts` | create | Gateway configuration |
| `src/lib/hmrc/gateway/fraud-headers.ts` | create | Fraud prevention header generator |

---

## Responsibilities

1. Configure TLS 1.2+ with strong cipher suites
2. Load and validate client certificates for mutual TLS
3. Implement connection pooling for performance
4. Generate HMRC fraud prevention headers
5. Handle connection timeouts and retries

---

## Contracts

### GatewayClient
- **Class:** `GatewayClient`
- **Constructor:** `new GatewayClient(config: GatewayConfig)`
- **Config:**
  ```typescript
  interface GatewayConfig {
    baseUrl: string; // https://tpvs.hmrc.gov.uk/rti or test URL
    timeoutMs: number; // Default: 30000
    maxRetries: number; // Default: 3
    tls: {
      minVersion: 'TLSv1.2';
      certPath: string; // Client certificate PEM
      keyPath: string; // Private key PEM
      caPath?: string; // CA certificate for validation
    };
    fraudPrevention: {
      vendorId: string;
      productVersion: string;
    };
  }
  ```
- **Method:** `async request(options: RequestOptions): Promise<Response>`
  ```typescript
  interface RequestOptions {
    method: 'POST' | 'GET';
    path: string;
    body?: string; // XML payload
    headers?: Record<string, string>;
  }
  
  interface Response {
    statusCode: number;
    headers: Record<string, string>;
    body: string;
    durationMs: number;
  }
  ```
- **Errors:**
  - `TLSConnectionError` — TLS handshake failed
  - `CertificateError` — Certificate validation failed
  - `TimeoutError` — Request timeout
  - `NetworkError` — Connection failure

### Fraud Prevention Headers (per HMRC requirements)
```typescript
interface FraudPreventionHeaders {
  'Gov-Client-Connection-Method': 'WEB_APP_VIA_SERVER';
  'Gov-Client-Public-IP': string; // Client public IP
  'Gov-Client-Public-Port': string;
  'Gov-Client-Device-ID': string; // Persistent device ID
  'Gov-Client-User-IDs': string; // Hashed user ID
  'Gov-Client-Timezone': string; // e.g., 'UTC+00:00'
  'Gov-Client-Local-IPs': string; // Hashed local IPs
  'Gov-Client-Screens': string; // Screen resolution
  'Gov-Client-Window-Size': string;
  'Gov-Client-Browser-Plugins': string;
  'Gov-Client-Browser-JS-User-Agent': string;
  'Gov-Client-Browser-Do-Not-Track': string;
  'Gov-Client-Multi-Factor': string;
  'Gov-Vendor-Version': string; // Product version
  'Gov-Vendor-License-IDs': string;
  'Gov-Vendor-Product-Name': 'UKBureauPayroll';
}
```

### HMRC Endpoints
| Environment | Base URL | Purpose |
|-------------|----------|---------|
| Test | https://tpvs.hmrc.gov.uk/rti | Test submissions |
| Live | https://tpvs.hmrc.gov.uk/rti | Production submissions |

---

## Business Rules & Invariants

1. TLS 1.2 is minimum version; reject TLS 1.0/1.1
2. Client certificate must be valid and not expired
3. All requests must include fraud prevention headers
4. Connection pool size: 10 max sockets per origin
5. Request timeout: 30 seconds default
6. Retry only on idempotent operations with exponential backoff

---

## Edge Cases

1. **Certificate expiry** — Alert 30 days before, auto-renewal if possible
2. **Clock skew** — NTP sync required, reject if skew >5 minutes
3. **Connection pool exhaustion** — Queue requests, timeout if queued >60s
4. **TLS downgrade attack** — Strict TLS 1.2+, no fallback

---

## Tests

### gateway-client.test.ts
- Successful HTTPS request with valid certificate
- Reject TLS 1.1 connection
- Timeout handling (simulate slow response)
- Certificate validation failure
- Fraud prevention headers present
- Connection pooling (reuse connections)

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/gateway/client.test.ts
npm run lint src/lib/hmrc/gateway/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § Integration Points → HMRC Gateway
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Compliance
