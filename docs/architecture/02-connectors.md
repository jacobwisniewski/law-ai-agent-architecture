# Connector Framework

## Overview

The connector framework provides a generic interface for ingesting documents and emails from external systems. Designed for extensibility - add new data sources without modifying core platform code.

## Connector Interface

```typescript
interface Connector {
  id: string;
  name: string;
  
  // Authentication
  getAuthUrl(tenantId: string): string;
  handleAuthCallback(code: string, tenantId: string): Promise<Credentials>;
  refreshCredentials(credentials: Credentials): Promise<Credentials>;
  
  // Discovery
  listDocuments(options: ListOptions): AsyncIterable<DocumentMeta>;
  listEmails(options: ListOptions): AsyncIterable<EmailMeta>;
  
  // Content retrieval
  getDocument(id: string): Promise<DocumentContent>;
  getEmail(id: string): Promise<EmailContent>;
  
  // Permissions
  getDocumentPermissions(id: string): Promise<ACL>;
  getEmailPermissions(id: string): Promise<ACL>;
  
  // Change detection
  getDeltaToken(): Promise<string>;
  getChanges(deltaToken: string): AsyncIterable<ChangeEvent>;
  
  // Webhooks (optional)
  registerWebhook?(callback: WebhookConfig): Promise<WebhookRegistration>;
  handleWebhook?(payload: unknown): Promise<ChangeEvent[]>;
}

interface ListOptions {
  since?: Date;
  deltaToken?: string;
  pageSize?: number;
  filter?: {
    folders?: string[];
    mimeTypes?: string[];
  };
}

interface DocumentMeta {
  id: string;
  name: string;
  mimeType: string;
  size: number;
  createdAt: Date;
  modifiedAt: Date;
  parentPath: string;
  webUrl: string;
  driveId?: string;
}

interface EmailMeta {
  id: string;
  subject: string;
  from: EmailAddress;
  to: EmailAddress[];
  cc: EmailAddress[];
  receivedAt: Date;
  threadId: string;
  hasAttachments: boolean;
  webUrl: string;
}

interface ChangeEvent {
  type: 'created' | 'updated' | 'deleted';
  resourceType: 'document' | 'email';
  resourceId: string;
  timestamp: Date;
}
```

## Document Data Model

Documents stored in the platform after ingestion:

```typescript
interface Document {
  id: string;
  tenantId: string;
  connectorId: string;
  
  // External reference
  externalId: string;            // ID in source system
  externalUrl: string;           // URL to open in source system
  
  // Metadata
  title: string;
  mimeType: string;
  sizeBytes: number;
  
  // Content
  extractedText: string;         // Full extracted text
  contentHash: string;           // SHA-256 for deduplication
  
  // Hierarchy
  parentPath: string;            // Folder path in source
  parentId?: string;             // Parent folder external ID
  
  // Timestamps
  sourceCreatedAt: Date;         // Created in source system
  sourceModifiedAt: Date;        // Modified in source system
  ingestedAt: Date;              // When we ingested it
  lastSyncedAt: Date;            // Last sync check
  
  // Processing status
  status: 'pending' | 'processing' | 'indexed' | 'error';
  errorMessage?: string;
  chunkCount: number;            // Number of chunks created
  
  // Source tracking
  sourceType: 'sharepoint' | 'onedrive' | 'email_attachment' | 'netdocuments' | 'imanage';
  sourceEmailId?: string;        // If extracted from email attachment
}

interface DocumentContent {
  documentId: string;
  content: Buffer;               // Raw file content
  mimeType: string;
}
```

## M365 Connector Implementation

### Authentication Flow

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Tenant  │────▶│ Platform │────▶│  Azure   │────▶│  M365    │
│  Admin   │     │   API    │     │   AD     │     │  Graph   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
     │                │                │                │
     │  1. Connect    │                │                │
     │───────────────▶│                │                │
     │                │  2. Redirect   │                │
     │◀───────────────│───────────────▶│                │
     │                │                │                │
     │  3. Consent    │                │                │
     │───────────────────────────────▶│                │
     │                │                │                │
     │  4. Auth code  │                │                │
     │◀───────────────────────────────│                │
     │                │                │                │
     │                │  5. Exchange   │                │
     │                │───────────────▶│                │
     │                │                │                │
     │                │  6. Tokens     │                │
     │                │◀───────────────│                │
     │                │                │                │
     │                │  7. API calls  │                │
     │                │───────────────────────────────▶│
```

### Required Graph API Permissions

| Permission | Type | Purpose |
|------------|------|---------|
| `Files.Read.All` | Application | Read all files in SharePoint/OneDrive |
| `Mail.Read` | Delegated or App | Read emails |
| `Sites.Read.All` | Application | Read SharePoint sites |
| `User.Read.All` | Application | Read user profiles for ACL mapping |
| `Group.Read.All` | Application | Read groups for ACL expansion |

### Delta Sync Strategy

M365 Graph API supports delta queries for efficient syncing:

```typescript
// Initial sync - get all documents
GET /drives/{driveId}/root/delta

// Response includes @odata.deltaLink
{
  "value": [...documents...],
  "@odata.deltaLink": "https://graph.microsoft.com/v1.0/drives/{id}/root/delta?token=..."
}

// Subsequent syncs - only changes
GET {deltaLink}
```

### Webhook Registration

```typescript
// Register for change notifications
POST /subscriptions
{
  "changeType": "created,updated,deleted",
  "notificationUrl": "https://platform.example.com/webhooks/m365",
  "resource": "/drives/{driveId}/root",
  "expirationDateTime": "2024-01-01T00:00:00Z",
  "clientState": "{tenantId}:{secret}"
}
```

**Webhook limitations:**
- Max 3 days expiration (must renew)
- Notifications don't include full content (must fetch)
- Rate limits apply

## Ingestion Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INGESTION PIPELINE                                   │
│                                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│  │  Fetch  │───▶│ Extract │───▶│  Chunk  │───▶│  Embed  │───▶│  Index  │   │
│  │         │    │         │    │         │    │         │    │         │   │
│  │ • Delta │    │ • PDF   │    │ • Split │    │ • OpenAI│    │ • Store │   │
│  │   query │    │ • DOCX  │    │ • Overlap│   │   API   │    │ • ACL   │   │
│  │ • Rate  │    │ • HTML  │    │ • Meta  │    │         │    │ • Link  │   │
│  │   limit │    │ • Email │    │         │    │         │    │         │   │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘   │
│                                                                             │
│                      ┌─────────────────────────┐                           │
│                      │      Job Queue          │                           │
│                      │   (BullMQ + Redis)      │                           │
│                      └─────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Job Queue Design

```typescript
// Job types
type IngestionJob = 
  | { type: 'sync_tenant'; tenantId: string; connectorId: string; full?: boolean }
  | { type: 'process_document'; tenantId: string; documentId: string; connectorId: string }
  | { type: 'process_email'; tenantId: string; emailId: string; connectorId: string }
  | { type: 'delete_document'; tenantId: string; documentId: string }
  | { type: 'refresh_acl'; tenantId: string; resourceId: string };

// Queue configuration
const ingestionQueue = new Queue('ingestion', {
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: 1000,
    removeOnFail: 5000,
  }
});
```

### Document Extraction

| File Type | Library | Notes |
|-----------|---------|-------|
| PDF | `pdfjs-dist` | Mozilla's PDF.js, industry standard |
| DOCX | `mammoth` | Preserves structure, loses formatting |
| XLSX | `xlsx` | Extract text from cells |
| PPTX | `officegen` or custom | Slide-by-slide extraction |
| HTML/Email | `html-to-text` | Strip tags, preserve structure |
| Plain text | - | Direct use |
| OCR | `tesseract.js` | For scanned PDFs/images |

See [Tech Stack & Packages](./11-tech-stack.md) for detailed package information.

### Chunking Strategy

We use LangChain.js for chunking (see [Tech Stack](./11-tech-stack.md)):

```typescript
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";

const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 500,           // ~500 tokens
  chunkOverlap: 50,         // ~50 tokens overlap
  separators: ["\n\n", "\n", ". ", " ", ""],
});

interface Chunk {
  id: string;
  documentId: string;
  tenantId: string;
  content: string;
  embedding: number[];      // 1536 dimensions (text-embedding-3-small)
  metadata: {
    page?: number;
    section?: string;
    heading?: string;
    charStart: number;
    charEnd: number;
  };
}
```

## Future Connectors

| Connector | Complexity | Notes |
|-----------|------------|-------|
| Google Workspace | Medium | Similar to M365, Drive + Gmail APIs |
| iManage | High | SOAP/REST APIs, complex security model |
| NetDocuments | High | REST API, strict profiling requirements |
| Dropbox | Low | Simple REST API |
| Box | Medium | Good enterprise features |
| Local file upload | Low | Direct upload, manual ACL |

## Error Handling

```typescript
// Connector errors should be typed
class ConnectorError extends Error {
  constructor(
    message: string,
    public code: ConnectorErrorCode,
    public retryable: boolean,
    public retryAfter?: number
  ) {
    super(message);
  }
}

type ConnectorErrorCode =
  | 'AUTH_EXPIRED'
  | 'AUTH_REVOKED'
  | 'RATE_LIMITED'
  | 'NOT_FOUND'
  | 'PERMISSION_DENIED'
  | 'SERVICE_UNAVAILABLE'
  | 'UNKNOWN';
```

## Monitoring & Metrics

Track per-connector:
- Documents synced (count, size)
- Sync latency (time since last change)
- Error rates by type
- API quota usage
- Webhook delivery success
