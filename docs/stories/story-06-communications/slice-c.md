# Slice c: Email Open/Response Tracking

**Story:** story-06-communications
**Epic:** epic-04-bureau-operations
**Effort:** S
**Dependencies:** slice-b

---

## Goal

Implement email open tracking via tracking pixels and response detection. Track when recipients open emails and when they reply to enable engagement metrics and automated follow-up logic.

---

## Decision Checklist

- [x] All libraries/packages named: Next.js 14 App Router, base64-url 2.x, crypto (native)
- [x] All SDK methods/API calls identified: Image pixel endpoint, email parsing
- [x] All external service endpoints specified: GET /api/track/open?id={token}
- [x] All data contracts defined: TrackingToken payload, OpenEvent
- [x] All configuration/environment variables listed: TRACKING_PIXEL_ENABLED (default: true)
- [x] All error scenarios identified with handling strategy: Token validation failures
- [x] No "TBD", slash-notation, or placeholder text remaining

---

## Spec References

- 02-04-bureau-operations-spec.md § Data Models — ClientCommunication opened_at, response_received
- 02-04-bureau-operations-spec.md § Non-Functional Requirements — Tracking

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/app/api/track/open/route.ts` | create | Tracking pixel endpoint |
| `src/lib/services/tracking-service.ts` | create | Token generation and validation |
| `src/lib/email/template-helpers.ts` | create | Email template tracking helpers |
| `src/lib/email/response-parser.ts` | create | Email response detection |
| `src/tests/api/tracking.test.ts` | create | Tracking endpoint tests |

---

## Responsibilities

1. Generate unique tracking tokens for each email sent
2. Embed 1x1 transparent tracking pixel in HTML emails
3. Create endpoint to receive pixel requests and log opens
4. Prevent duplicate open tracking
5. Detect email responses via reply-to parsing
6. Update communication record with open/response timestamps

---

## Contracts

### TrackingService

```typescript
// src/lib/services/tracking-service.ts
export class TrackingService {
  private readonly secret: string;
  
  generateTrackingToken(communicationId: string): string {
    // Create signed JWT or HMAC token
    // Payload: { communicationId, sentAt }
    // Return base64url encoded token
  }
  
  validateTrackingToken(token: string): { communicationId: string; sentAt: Date } | null {
    // Verify signature
    // Check not expired (30 days)
    // Return payload or null if invalid
  }
  
  async recordOpen(
    token: string,
    metadata: { ip: string; userAgent: string; timestamp: Date }
  ): Promise<boolean>;
}
```

### Tracking Pixel Endpoint

- **Path:** `/api/track/open`
- **Method:** GET
- **Query Parameters:** `id` (tracking token)
- **Response:** 1x1 transparent GIF (image/gif)
- **Headers:** Cache-Control: no-store, no-cache

```typescript
// src/app/api/track/open/route.ts
export async function GET(request: Request): Promise<Response> {
  const { searchParams } = new URL(request.url);
  const token = searchParams.get('id');
  
  if (token) {
    await trackingService.recordOpen(token, {
      ip: request.headers.get('x-forwarded-for') || 'unknown',
      userAgent: request.headers.get('user-agent') || 'unknown',
      timestamp: new Date(),
    });
  }
  
  // Return 1x1 transparent GIF
  const pixel = Buffer.from('R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7', 'base64');
  return new Response(pixel, {
    headers: {
      'Content-Type': 'image/gif',
      'Cache-Control': 'no-store, no-cache, must-revalidate',
    },
  });
}
```

### Email Template Helpers

```typescript
// src/lib/email/template-helpers.ts
export function addTrackingPixel(
  htmlContent: string,
  communicationId: string
): string {
  const token = trackingService.generateTrackingToken(communicationId);
  const pixelUrl = `${process.env.APP_URL}/api/track/open?id=${token}`;
  const pixel = `<img src="${pixelUrl}" width="1" height="1" alt="" style="display:block;" />`;
  
  // Append before closing body tag, or append to end
  if (htmlContent.includes('</body>')) {
    return htmlContent.replace('</body>', `${pixel}</body>`);
  }
  return htmlContent + pixel;
}

export function addTrackingToLinks(
  htmlContent: string,
  communicationId: string
): string {
  // Add UTM parameters or tracking IDs to links
  const token = trackingService.generateTrackingToken(communicationId);
  // Parse HTML, add tracking params to all <a> tags
  return modifiedHtml;
}
```

### Response Detection

```typescript
// src/lib/email/response-parser.ts
export class ResponseParser {
  async detectResponse(email: IncomingEmail): Promise<ResponseDetectionResult> {
    // Parse email headers for In-Reply-To or References
    // Match against sent message IDs
    // Find original communication by messageId
    // Return result with original communication ID
  }
  
  async processIncomingEmail(email: IncomingEmail): Promise<void> {
    const result = await this.detectResponse(email);
    if (result.isResponse && result.originalCommunicationId) {
      await communicationService.markAsResponded(result.originalCommunicationId);
      await chaseService.recordResponse(result.employerId);
    }
  }
}

interface ResponseDetectionResult {
  isResponse: boolean;
  originalCommunicationId?: string;
  employerId?: string;
  confidence: number;
}
```

---

## Business Rules & Invariants

1. Tracking pixel is 1x1 transparent GIF
2. Only first open is recorded (subsequent ignored)
3. Tokens expire after 30 days
4. IP and user agent captured for analytics
5. Response detection uses email threading headers
6. Manual marking as responded also accepted

---

## Edge Cases

1. **Images blocked** — No open tracked (accept limitation)
2. **Preview pane** — May trigger false open (acceptable)
3. **Forwarded email** — Original recipient tracked
4. **Reply to wrong email** — Manual matching or ignore

---

## Tests

### tracking.test.ts

- Tracking token generates correctly
- Pixel endpoint records open
- Duplicate opens ignored
- Invalid token handled gracefully
- Response detection matches correctly

---

## Verification

```bash
npm run test:unit -- tracking.test.ts
npm run typecheck
npm run lint
```

---

## Source Sections

- 02-04-bureau-operations-spec.md § Data Models → ClientCommunication tracking fields
