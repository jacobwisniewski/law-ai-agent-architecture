# Tech Stack & Package Decisions

## Overview

This document outlines the specific packages and libraries chosen for implementation. All packages are actively maintained, well-documented, and have strong TypeScript support.

## Core Dependencies

```json
{
  "dependencies": {
    "better-auth": "^1.4.0",
    "langchain": "^0.3.0",
    "@langchain/openai": "^0.3.0",
    "@langchain/community": "^0.3.0",
    "@microsoft/microsoft-graph-client": "^3.0.0",
    "bullmq": "^5.0.0",
    "stripe": "^17.0.0",
    "pdfjs-dist": "^4.0.0",
    "tesseract.js": "^5.0.0",
    "mammoth": "^1.8.0",
    "mailparser": "^3.7.0",
    "html-to-text": "^9.0.0",
    "rate-limiter-flexible": "^5.0.0",
    "pgvector": "^0.2.0"
  }
}
```

## Package Decisions by Category

### Authentication: Better Auth

| Aspect | Details |
|--------|---------|
| Package | `better-auth` |
| Stars | 24.4k |
| License | MIT |
| Website | https://better-auth.com |

**Why Better Auth over Auth.js/NextAuth:**

1. **TypeScript-first** - Better type inference and DX
2. **Framework-agnostic** - Works with Next.js, Fastify, Express, etc.
3. **Built-in multi-tenancy** - Plugin for organization/tenant support
4. **Enterprise features** - SSO, SAML, OIDC built-in as plugins
5. **2FA/Passkeys** - First-class support via plugins
6. **Active development** - Very active, 695+ contributors

**Key Features We'll Use:**

- Email/password authentication (MVP)
- Organization plugin (multi-tenancy)
- Session management
- Microsoft OAuth (for M365 connection)
- SSO plugin (post-MVP: Okta, Azure AD)

**Example Setup:**

```typescript
import { betterAuth } from "better-auth";
import { organization } from "better-auth/plugins";

export const auth = betterAuth({
  database: {
    provider: "pg",
    url: process.env.DATABASE_URL,
  },
  emailAndPassword: {
    enabled: true,
  },
  plugins: [
    organization({
      allowUserToCreateOrganization: true,
    }),
  ],
  socialProviders: {
    microsoft: {
      clientId: process.env.MS_CLIENT_ID!,
      clientSecret: process.env.MS_CLIENT_SECRET!,
    },
  },
});
```

---

### RAG & AI: LangChain.js

| Aspect | Details |
|--------|---------|
| Package | `langchain`, `@langchain/openai`, `@langchain/community` |
| Stars | 16.6k |
| License | MIT |
| Docs | https://js.langchain.com |

**Why LangChain:**

1. **Comprehensive RAG pipeline** - Handles retrieval, context building, generation
2. **Text splitters** - Built-in semantic chunking with overlap
3. **Vector store integrations** - PGVector support out of the box
4. **Embeddings abstraction** - Easy to swap embedding providers
5. **Streaming support** - Built-in streaming for chat responses
6. **Document loaders** - PDF, DOCX, HTML loaders included

**What LangChain Replaces:**

| Our Custom Code | LangChain Alternative |
|-----------------|----------------------|
| Chunking logic | `RecursiveCharacterTextSplitter` |
| Embedding calls | `OpenAIEmbeddings` |
| Vector queries | `PGVectorStore` |
| Hybrid search | `EnsembleRetriever` |
| RAG chain | `RetrievalQAChain` |
| Streaming | Built-in `streamEvents` |

**Example Setup:**

```typescript
import { ChatOpenAI, OpenAIEmbeddings } from "@langchain/openai";
import { PGVectorStore } from "@langchain/community/vectorstores/pgvector";
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";

// Embeddings
const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",
  dimensions: 1536,
});

// Vector store
const vectorStore = await PGVectorStore.initialize(embeddings, {
  postgresConnectionOptions: {
    connectionString: process.env.DATABASE_URL,
  },
  tableName: "document_chunks",
  columns: {
    idColumnName: "id",
    vectorColumnName: "embedding",
    contentColumnName: "content",
    metadataColumnName: "metadata",
  },
});

// Text splitting
const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 500,
  chunkOverlap: 50,
  separators: ["\n\n", "\n", ". ", " ", ""],
});

// LLM
const llm = new ChatOpenAI({
  model: "gpt-4.1",
  temperature: 0.1,
  streaming: true,
});
```

---

### Document Extraction

| Package | Purpose | Stars | Notes |
|---------|---------|-------|-------|
| `pdfjs-dist` | PDF text extraction | 52.5k | Mozilla, industry standard |
| `tesseract.js` | OCR for scanned docs | 37.6k | WebAssembly Tesseract |
| `mammoth` | DOCX to HTML/text | 6k | Simple, preserves structure |
| `mailparser` | Email parsing | 1.7k | MIME, attachments, streaming |
| `html-to-text` | HTML stripping | 1.7k | Clean text from emails |

**LangChain also provides document loaders**, but these packages give more control:

```typescript
// PDF extraction
import { getDocument } from "pdfjs-dist";

async function extractPdfText(buffer: Buffer): Promise<string> {
  const pdf = await getDocument({ data: buffer }).promise;
  const pages: string[] = [];
  
  for (let i = 1; i <= pdf.numPages; i++) {
    const page = await pdf.getPage(i);
    const content = await page.getTextContent();
    const text = content.items.map((item: any) => item.str).join(" ");
    pages.push(text);
  }
  
  return pages.join("\n\n");
}

// DOCX extraction
import mammoth from "mammoth";

async function extractDocxText(buffer: Buffer): Promise<string> {
  const result = await mammoth.extractRawText({ buffer });
  return result.value;
}

// Email parsing
import { simpleParser } from "mailparser";
import { convert } from "html-to-text";

async function extractEmailText(raw: string): Promise<string> {
  const parsed = await simpleParser(raw);
  const body = parsed.html 
    ? convert(parsed.html, { wordwrap: false })
    : parsed.text || "";
  return body;
}
```

---

### Microsoft 365 Integration

| Aspect | Details |
|--------|---------|
| Package | `@microsoft/microsoft-graph-client` |
| Stars | 812 |
| License | MIT |
| Docs | https://learn.microsoft.com/graph |

**Official SDK** - Best support for Graph API features:

```typescript
import { Client } from "@microsoft/microsoft-graph-client";

const client = Client.initWithMiddleware({
  authProvider: {
    getAccessToken: async () => accessToken,
  },
});

// Delta sync for documents
const delta = await client
  .api("/drives/{driveId}/root/delta")
  .get();

// Get file content
const content = await client
  .api("/drives/{driveId}/items/{itemId}/content")
  .get();

// Get permissions
const permissions = await client
  .api("/drives/{driveId}/items/{itemId}/permissions")
  .get();
```

---

### Job Queue: BullMQ

| Aspect | Details |
|--------|---------|
| Package | `bullmq` |
| Stars | 8.1k |
| License | MIT |
| Backend | Redis |

**Already in architecture docs** - Best Redis-based queue for Node.js:

```typescript
import { Queue, Worker } from "bullmq";

const ingestionQueue = new Queue("ingestion", {
  connection: { host: "localhost", port: 6379 },
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
  },
});

const worker = new Worker("ingestion", async (job) => {
  switch (job.name) {
    case "process_document":
      await processDocument(job.data);
      break;
    case "sync_tenant":
      await syncTenant(job.data);
      break;
  }
}, {
  connection: { host: "localhost", port: 6379 },
  concurrency: 5,
});
```

---

### Rate Limiting

| Aspect | Details |
|--------|---------|
| Package | `rate-limiter-flexible` |
| Stars | 3.4k |
| License | ISC |
| Backends | Redis, PostgreSQL, Memory |

**Better than rolling our own:**

```typescript
import { RateLimiterRedis } from "rate-limiter-flexible";
import Redis from "ioredis";

const redis = new Redis();

const searchLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: "rl:search",
  points: 60,      // requests
  duration: 60,    // per minute
});

const chatLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: "rl:chat",
  points: 20,      // requests
  duration: 60,    // per minute
});

// Usage in API route
async function searchHandler(req, res) {
  try {
    await searchLimiter.consume(req.user.id);
    // ... handle search
  } catch (err) {
    res.status(429).json({ error: "Too many requests" });
  }
}
```

---

### Billing: Stripe

| Aspect | Details |
|--------|---------|
| Package | `stripe` |
| Stars | 4.3k |
| License | MIT |

**Official SDK** - Already in architecture docs, no change needed.

---

## What We're NOT Using

### CASL (Authorization Library)

**What it is:** CASL is an isomorphic authorization library for checking "can user X do action Y on resource Z".

**Why we're skipping it:**

1. **Overkill for our ACL model** - Our permissions are simple: user has access to document (yes/no) based on M365 sync
2. **Performance overhead** - CASL's MongoDB-style conditions add abstraction we don't need
3. **Our ACL is simpler** - Just check if `userId` is in `allowed_user_ids` array

**Our approach instead:**

```typescript
// Simple ACL check - no CASL needed
async function canAccessDocument(
  userId: string, 
  documentId: string,
  tenantId: string
): Promise<boolean> {
  const acl = await db.expandedAcls.findFirst({
    where: { 
      tenantId,
      resourceId: documentId,
    },
  });
  
  return acl?.allowedUserIds.includes(userId) ?? false;
}
```

If we need complex role-based permissions later (e.g., "partners can see all matters, associates only their assigned"), we can add CASL then.

---

## Embedding Model Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| Model | `text-embedding-3-small` | Cost-effective, good quality |
| Dimensions | 1536 | Default for this model |
| Batch size | 100 | Max texts per API call |

**Cost Estimate:**
- ~$0.02 per 1M tokens
- Average document: ~2000 tokens
- 10,000 documents: ~$0.40

---

## Package Version Policy

- **Pin major versions** in package.json
- **Renovate/Dependabot** for automated updates
- **Weekly update review** for security patches

---

## Development Setup

```bash
# Install dependencies
pnpm install

# Environment variables
cp .env.example .env.local

# Required env vars:
# DATABASE_URL=postgresql://...
# REDIS_URL=redis://...
# OPENAI_API_KEY=sk-...
# BETTER_AUTH_SECRET=...
# STRIPE_SECRET_KEY=sk_...
# STRIPE_WEBHOOK_SECRET=whsec_...

# App URLs
# APP_URL=http://localhost:3000

# M365 Platform Auth (for "Login with Microsoft")
# MS_CLIENT_ID=...
# MS_CLIENT_SECRET=...

# M365 Connector (separate app for data access - see 12-auth-architecture.md)
# MS_CONNECTOR_CLIENT_ID=...
# MS_CONNECTOR_CLIENT_SECRET=...

# Token encryption (32-byte hex key for AES-256-GCM)
# TOKEN_ENCRYPTION_KEY=...
```
