# Email Ingestion

## Overview

Email is a critical data source for law firms - often containing the most current context about matters, client communications, and negotiations. This doc covers email-specific ingestion challenges.

## Email Data Model

```typescript
interface Email {
  id: string;
  tenantId: string;
  connectorId: string;
  externalId: string;           // ID in source system (M365 message ID)
  
  // Core fields
  subject: string;
  from: EmailAddress;
  to: EmailAddress[];
  cc: EmailAddress[];
  bcc: EmailAddress[];          // Usually not available
  receivedAt: Date;
  sentAt: Date;
  
  // Threading
  threadId: string;             // Conversation ID
  inReplyTo?: string;           // Parent message ID
  references: string[];         // Full thread chain
  
  // Content
  bodyText: string;             // Plain text version
  bodyHtml?: string;            // Original HTML (stored for reference)
  
  // Attachments
  attachments: AttachmentMeta[];
  
  // Metadata
  importance: 'low' | 'normal' | 'high';
  categories: string[];         // User-applied labels
  webUrl: string;               // Link to open in Outlook
  
  // ACL
  mailboxOwner: string;         // Primary permission holder
  sharedWith: string[];         // Delegates, shared mailbox members
}

interface AttachmentMeta {
  id: string;
  name: string;
  mimeType: string;
  size: number;
  isInline: boolean;            // Embedded image vs actual attachment
  documentId?: string;          // Link to extracted document record
}

interface EmailAddress {
  email: string;
  name?: string;
}
```

## M365 Email Sync

### Graph API Endpoints

```typescript
// List messages with delta
GET /users/{userId}/mailFolders/inbox/messages/delta

// Get specific message
GET /users/{userId}/messages/{messageId}

// Get attachment content
GET /users/{userId}/messages/{messageId}/attachments/{attachmentId}/$value

// Subscribe to changes
POST /subscriptions
{
  "changeType": "created,updated,deleted",
  "resource": "/users/{userId}/mailFolders/inbox/messages",
  "notificationUrl": "https://...",
  "expirationDateTime": "..."
}
```

### Sync Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    EMAIL SYNC FLOW                           │
│                                                             │
│  ┌─────────────────┐                                        │
│  │  Initial Sync   │                                        │
│  │                 │                                        │
│  │  • Fetch all    │    ┌─────────────────┐                │
│  │    messages     │───▶│  Process Each   │                │
│  │  • Store delta  │    │                 │                │
│  │    token        │    │ • Parse body    │                │
│  └─────────────────┘    │ • Extract attach│                │
│          │              │ • Store ACL     │                │
│          ▼              │ • Chunk + embed │                │
│  ┌─────────────────┐    └─────────────────┘                │
│  │  Delta Sync     │              │                        │
│  │  (periodic)     │              ▼                        │
│  │                 │    ┌─────────────────┐                │
│  │  • Use delta    │    │    pgvector     │                │
│  │    token        │    │                 │                │
│  │  • Get changes  │    │ • Email chunks  │                │
│  │  • Update token │    │ • Attachments   │                │
│  └─────────────────┘    └─────────────────┘                │
│          │                                                  │
│          ▼                                                  │
│  ┌─────────────────┐                                       │
│  │  Webhook Events │                                       │
│  │  (real-time)    │                                       │
│  │                 │                                       │
│  │  • New email    │                                       │
│  │  • Trigger sync │                                       │
│  └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Email Threading

### Why Threading Matters

When a user asks "What did John say about the contract?", the answer might span multiple emails in a thread. RAG should:
1. Retrieve relevant emails
2. Include thread context in the prompt
3. Cite specific messages

### Thread Resolution

```typescript
interface EmailThread {
  threadId: string;
  subject: string;              // Normalized (without Re:, Fwd:)
  participants: EmailAddress[];
  messageCount: number;
  firstMessageAt: Date;
  lastMessageAt: Date;
  messages: Email[];            // Ordered chronologically
}

// Build thread from individual emails
function buildThread(emails: Email[]): EmailThread {
  const sorted = emails.sort((a, b) => 
    a.receivedAt.getTime() - b.receivedAt.getTime()
  );
  
  return {
    threadId: emails[0].threadId,
    subject: normalizeSubject(emails[0].subject),
    participants: uniqueParticipants(emails),
    messageCount: emails.length,
    firstMessageAt: sorted[0].receivedAt,
    lastMessageAt: sorted[sorted.length - 1].receivedAt,
    messages: sorted,
  };
}
```

### Thread-Aware Chunking

Options for chunking email threads:

| Strategy | Pros | Cons |
|----------|------|------|
| Individual emails | Simple, granular citations | Loses thread context |
| Full thread as one chunk | Full context | May exceed token limits |
| Sliding window over thread | Balance of both | More complex |

**Recommendation:** Chunk individual emails but store `threadId`. At retrieval time, optionally fetch sibling messages for context.

## Attachment Handling

```
Email with Attachments
        │
        ▼
┌───────────────────────────────────────┐
│         Attachment Router              │
│                                        │
│  Is inline image? ──────▶ Skip/OCR    │
│          │                             │
│          ▼                             │
│  Is document? ──────────▶ Full extract │
│  (PDF, DOCX, etc.)         + chunk    │
│          │                  + embed    │
│          ▼                     │       │
│  Is other? ─────────────▶ Metadata    │
│  (ZIP, EXE, etc.)          only       │
│                                        │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│         Link to Parent Email           │
│                                        │
│  Document.sourceEmailId = email.id     │
│  Document.context = email.subject      │
│                                        │
└───────────────────────────────────────┘
```

### Attachment Deduplication

Same file often attached to multiple emails. Handle via content hashing:

```typescript
interface Document {
  // ...
  contentHash: string;          // SHA-256 of content
  sourceEmails: string[];       // Multiple emails can reference same doc
}

// On ingestion
const hash = computeHash(attachmentContent);
const existing = await db.documents.findByHash(tenantId, hash);

if (existing) {
  // Link existing doc to this email
  await db.documents.addSourceEmail(existing.id, emailId);
} else {
  // Create new document
  await ingestDocument(attachmentContent, { sourceEmailId: emailId });
}
```

## Email ACL

### Permission Model

Email permissions are simpler than documents:

| Access Type | Who Can See |
|-------------|-------------|
| Personal mailbox | Owner + delegates |
| Shared mailbox | All members |
| Sent email | Sender (in Sent folder) |
| Distribution list | Each recipient sees their copy |

### M365 Specifics

```typescript
// Get mailbox delegates
GET /users/{userId}/mailboxSettings

// Check if shared mailbox
GET /users/{mailboxId}?$select=mailboxSettings

// For shared mailboxes, get members
GET /groups/{groupId}/members  // If mailbox is group-linked
```

### ACL Storage

```typescript
interface EmailACL {
  emailId: string;
  tenantId: string;
  
  // Primary access
  mailboxOwner: string;         // User ID who owns the mailbox
  
  // Delegated access
  delegates: string[];          // Users with delegate access
  
  // Shared mailbox members (if applicable)
  sharedMailboxMembers?: string[];
  
  // Computed at query time
  // Can user X see this email?
  // = user is owner OR user in delegates OR user in sharedMailboxMembers
}
```

## Email-Specific Search Considerations

### Searchable Fields

| Field | Search Type | Notes |
|-------|-------------|-------|
| Subject | Keyword + vector | High signal |
| Body | Vector | Main content |
| From/To/CC | Keyword | Exact match filter |
| Attachments | Vector | Via linked docs |
| Date | Range filter | Common filter |
| Thread | Retrieval expansion | Fetch related |

### Query Examples

```typescript
// "Emails from John about the merger"
{
  filters: {
    from: { contains: "john" }
  },
  query: "merger",
  type: "email"
}

// "What did the client say last week?"
{
  filters: {
    receivedAt: { gte: lastWeek }
  },
  query: "client communication",
  type: "email"
}
```

## Volume Considerations

Email volume can be very high:

| Metric | Typical Law Firm |
|--------|------------------|
| Emails per user per day | 50-200 |
| Users per firm | 10-500 |
| Emails per month | 15,000 - 3,000,000 |
| Average email size | 50KB (without attachments) |

### Optimization Strategies

1. **Skip large attachments initially** - Defer huge files to background processing
2. **Batch embedding calls** - Group chunks to reduce API calls
3. **Incremental indexing** - Only re-embed on content change, not metadata
4. **TTL on old emails** - Consider not indexing emails older than X years (configurable)
5. **Folder filtering** - Skip Spam, Trash, maybe Sent (configurable)
