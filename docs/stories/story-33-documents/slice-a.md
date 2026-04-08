# Slice a: Document API with Secure Access

**Story:** story-33-documents
**Epic:** epic-07-employer-portal
**Effort:** S
**Dependencies:** story-26-client-auth

## Goal

Implement the backend API for secure employer document access including P30 notices, monthly summaries, year-end documents, and P45s with proper authorization and audit logging.

## Decision Checklist

- [x] All libraries/packages named: `drizzle-orm@latest`, `zod@3.x`
- [x] SDK methods identified: `db.query`, file stream APIs
- [x] External service endpoints: N/A (internal document storage)
- [x] Data contracts defined: DocumentType, DocumentMetadata, DocumentAccess
- [x] Configuration variables: `DOCUMENT_STORAGE_PATH`, `DOCUMENT_RETENTION_DAYS`
- [x] Error scenarios identified with handling
- [x] No TBD or placeholders remaining

## Spec References
- 02-06-employer-portal-spec.md:§ User Journeys → Document access
- 02-11-reporting-documents-spec.md:§ Document types and retention

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/server/api/routers/documents.ts` | create | Documents tRPC router |
| `src/lib/documents/storage.ts` | create | Document storage access |
| `src/lib/documents/types.ts` | create | Document type definitions |
| `src/lib/documents/audit.ts` | create | Access audit logging |

## Responsibilities
1. List documents accessible to the employer
2. Filter and categorize documents by type
3. Provide secure download URLs (time-limited)
4. Log all document access for audit
5. Enforce document retention policies
6. Handle expired/archived document indicators
7. Support search by date range and type

## Contracts

### DocumentType Enum
```typescript
// src/lib/documents/types.ts
export const documentTypeSchema = z.enum([
  'p30_notice',
  'monthly_summary',
  'year_end_p35',
  'p60',
  'p45',
  'fps_summary',
  'eps_summary',
  'custom_report',
]);

export type DocumentType = z.infer<typeof documentTypeSchema>;
```

### DocumentMetadata Type
```typescript
// src/lib/documents/types.ts
export interface DocumentMetadata {
  id: string;
  documentType: DocumentType;
  title: string;
  description?: string;
  period?: {
    startDate: Date;
    endDate: Date;
  };
  taxYear?: string;  // e.g., "2025-26"
  fileSize: number;
  fileFormat: 'pdf' | 'csv' | 'xlsx';
  createdAt: Date;
  availableUntil: Date;
  isArchived: boolean;
  downloadUrl?: string;  // Time-limited signed URL
}

export interface DocumentListFilters {
  documentType?: DocumentType;
  taxYear?: string;
  startDate?: Date;
  endDate?: Date;
  includeArchived?: boolean;
}
```

### DocumentAccessLog Type
```typescript
// src/lib/documents/types.ts
export interface DocumentAccessLog {
  id: string;
  documentId: string;
  userId: string;
  employerId: string;
  accessedAt: Date;
  accessType: 'view' | 'download';
  ipAddress: string;
  userAgent: string;
}
```

### Documents Router
```typescript
// src/server/api/routers/documents.ts
export const documentsRouter = router({
  // List documents for employer
  listDocuments: protectedProcedure
    .input(
      z.object({
        filters: z.object({
          documentType: documentTypeSchema.optional(),
          taxYear: z.string().optional(),
          startDate: z.date().optional(),
          endDate: z.date().optional(),
          includeArchived: z.boolean().default(false),
        }).optional(),
        limit: z.number().min(1).max(100).default(50),
        offset: z.number().min(0).default(0),
      })
    )
    .query(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      
      const documents = await db.query.documents.findMany({
        where: and(
          eq(documents.employerId, employerId),
          input.filters?.documentType 
            ? eq(documents.documentType, input.filters.documentType)
            : undefined,
          input.filters?.taxYear
            ? eq(documents.taxYear, input.filters.taxYear)
            : undefined,
          input.filters?.startDate
            ? gte(documents.createdAt, input.filters.startDate)
            : undefined,
          input.filters?.endDate
            ? lte(documents.createdAt, input.filters.endDate)
            : undefined,
          input.filters?.includeArchived
            ? undefined
            : eq(documents.isArchived, false)
        ),
        orderBy: [desc(documents.createdAt)],
        limit: input.limit,
        offset: input.offset,
      });
      
      // Get total count for pagination
      const totalCount = await db.select({ count: count() })
        .from(documents)
        .where(and(
          eq(documents.employerId, employerId),
          input.filters?.includeArchived ? undefined : eq(documents.isArchived, false)
        ));
      
      return {
        documents: documents.map(doc => ({
          id: doc.id,
          documentType: doc.documentType,
          title: doc.title,
          description: doc.description,
          period: doc.periodStart && doc.periodEnd ? {
            startDate: doc.periodStart,
            endDate: doc.periodEnd,
          } : undefined,
          taxYear: doc.taxYear,
          fileSize: doc.fileSize,
          fileFormat: doc.fileFormat,
          createdAt: doc.createdAt,
          availableUntil: doc.availableUntil,
          isArchived: doc.isArchived,
        })),
        pagination: {
          total: totalCount[0].count,
          limit: input.limit,
          offset: input.offset,
          hasMore: totalCount[0].count > input.offset + input.limit,
        },
      };
    }),

  // Get document download URL
  getDownloadUrl: protectedProcedure
    .input(z.object({ documentId: z.string().uuid() }))
    .mutation(async ({ input, ctx }) => {
      const employerId = ctx.session.user.employerId;
      const userId = ctx.session.user.id;
      
      // Verify document belongs to employer
      const document = await db.query.documents.findFirst({
        where: and(
          eq(documents.id, input.documentId),
          eq(documents.employerId, employerId)
        ),
      });
      
      if (!document) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: 'Document not found',
        });
      }
      
      if (document.isArchived) {
        throw new TRPCError({
          code: 'GONE',
          message: 'Document has been archived and is no longer available',
        });
      }
      
      // Log access
      await logDocumentAccess({
        documentId: input.documentId,
        userId,
        employerId,
        accessType: 'download',
        ipAddress: ctx.req.ip || 'unknown',
        userAgent: ctx.req.headers['user-agent'] || 'unknown',
      });
      
      // Generate time-limited signed URL (15 minutes)
      const downloadUrl = await generateSignedDownloadUrl(
        document.storagePath,
        document.fileName,
        15 * 60
      );
      
      return {
        documentId: document.id,
        fileName: document.fileName,
        fileFormat: document.fileFormat,
        downloadUrl,
        expiresAt: new Date(Date.now() + 15 * 60 * 1000),
      };
    }),

  // Get available tax years for filtering
  getTaxYears: protectedProcedure
    .query(async ({ ctx }) => {
      const employerId = ctx.session.user.employerId;
      
      const taxYears = await db.select({ taxYear: documents.taxYear })
        .from(documents)
        .where(and(
          eq(documents.employerId, employerId),
          isNotNull(documents.taxYear)
        ))
        .groupBy(documents.taxYear)
        .orderBy(desc(documents.taxYear));
      
      return taxYears.map(ty => ty.taxYear).filter(Boolean);
    }),
});
```

### Document Access Logging
```typescript
// src/lib/documents/audit.ts
export async function logDocumentAccess(
  access: Omit<DocumentAccessLog, 'id' | 'accessedAt'>
): Promise<void> {
  await db.insert(documentAccessLogs).values({
    id: crypto.randomUUID(),
    documentId: access.documentId,
    userId: access.userId,
    employerId: access.employerId,
    accessedAt: new Date(),
    accessType: access.accessType,
    ipAddress: access.ipAddress,
    userAgent: access.userAgent,
  });
}

export async function getDocumentAccessHistory(
  documentId: string,
  employerId: string,
  limit: number = 50
): Promise<DocumentAccessLog[]> {
  return db.query.documentAccessLogs.findMany({
    where: and(
      eq(documentAccessLogs.documentId, documentId),
      eq(documentAccessLogs.employerId, employerId)
    ),
    orderBy: [desc(documentAccessLogs.accessedAt)],
    limit,
    with: {
      user: {
        columns: {
          firstName: true,
          lastName: true,
        },
      },
    },
  });
}
```

### Signed URL Generation
```typescript
// src/lib/documents/storage.ts
export async function generateSignedDownloadUrl(
  storagePath: string,
  fileName: string,
  expirySeconds: number
): Promise<string> {
  // Implementation depends on storage backend:
  // - Local filesystem: Generate internal API token
  // - S3: Generate presigned URL
  // - Azure Blob: Generate SAS token
  
  const token = await createSecureToken({
    path: storagePath,
    filename: fileName,
    exp: Math.floor(Date.now() / 1000) + expirySeconds,
  });
  
  return `/api/documents/download?token=${token}`;
}
```

## Business Rules & Invariants
1. Documents strictly scoped to employer (no cross-access)
2. Archived documents cannot be downloaded
3. All downloads logged with user, timestamp, and IP
4. Download URLs expire after 15 minutes
5. Document retention follows HMRC guidelines (3+ years)
6. P45s available for leavers only
7. P60s available after tax year end

## Edge Cases
1. **Document not found** — Return 404 (don't reveal existence)
2. **Document archived** — Return 410 Gone with explanation
3. **URL expired** — Redirect to document list with message
4. **Concurrent downloads** — Each gets unique signed URL
5. **Large file download** — Stream response; don't buffer
6. **Access log write failure** — Log error but don't block download

## Tests

### src/lib/documents/audit.test.ts
- `should log document access`: Logging
- `should include all metadata`: Completeness
- `should retrieve access history`: Query
- `should handle log write failure`: Resilience

### src/server/api/routers/documents.test.ts
- `should list documents scoped to employer`: Security
- `should filter by document type`: Filtering
- `should generate time-limited download URL`: URL generation
- `should log download access`: Audit
- `should reject archived document access`: Validation
- `should require authentication`: Auth

## Verification
```bash
npm run test:unit src/lib/documents/audit.test.ts
npm run test:unit src/server/api/routers/documents.test.ts
npm run lint
npm run typecheck
```

## Source Sections
- 02-06-employer-portal-spec.md § User Journeys → Document access
- 02-11-reporting-documents-spec.md § Document retention
- 02-10-audit-compliance-spec.md:§ Audit logging requirements
