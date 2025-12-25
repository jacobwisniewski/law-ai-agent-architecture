# Tech Stack & Package Decisions

## Overview

This document outlines the specific packages and libraries chosen for implementation. All packages are actively maintained, well-documented, and have strong TypeScript support.

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MONOREPO STRUCTURE                                 │
│                                                                             │
│  /apps                                                                      │
│    /web          Vite + React + tRPC client + TanStack Query               │
│    /api          Fastify + tRPC + BullMQ workers                           │
│                                                                             │
│  /packages                                                                  │
│    /db           Kysely schema + migrations                                 │
│    /shared       Zod schemas, shared types                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Stack

| Layer | Technology | Why |
|-------|------------|-----|
| **Frontend** | Vite + React | Fast dev, no SSR needed for auth-gated SPA |
| **UI Components** | Radix UI + styled-components | Accessible primitives, full styling control |
| **API** | Fastify + tRPC | Type-safe client/server, fast, good for streaming |
| **Database** | PostgreSQL + pgvector | Relational + vector search in one |
| **ORM** | Kysely | Type-safe query builder, close to SQL, flexible |
| **Migrations** | kysely-ctl | Simple, TypeScript-native |
| **Cache/Queue** | Redis | Sessions, ACL cache, BullMQ backend |
| **Job Queue** | BullMQ | Background jobs, retries, scheduling |
| **Auth** | Better Auth | Multi-tenant, SSO-ready, framework-agnostic |
| **Validation** | Zod | Shared schemas between client/server |
| **Monorepo** | pnpm workspaces | Simple, fast, no extra tooling |
| **Runtime** | Node.js | Stability, ecosystem compatibility |
| **Logging** | Pino | Fast structured logging, Fastify native |

## Infrastructure (AWS ap-southeast-2)

| Service | AWS Resource | Notes |
|---------|--------------|-------|
| **API + Workers** | ECS Fargate | Serverless containers, no timeout limits |
| **Database** | RDS PostgreSQL | pgvector extension enabled |
| **Cache/Queue** | ElastiCache Redis | Sessions, ACL cache, BullMQ |
| **Logs** | CloudWatch Logs | Pino JSON → CloudWatch |
| **Metrics** | CloudWatch Metrics | Built-in, no extra infra |
| **Alerts** | CloudWatch Alarms | SNS for notifications |
| **Secrets** | AWS Secrets Manager | API keys, tokens |
| **CDN** | CloudFront | Static frontend assets |
| **Storage** | S3 | Document cache (optional) |

## Core Dependencies

```json
{
  "dependencies": {
    "@trpc/server": "^11.0.0",
    "@trpc/client": "^11.0.0",
    "@trpc/react-query": "^11.0.0",
    "@tanstack/react-query": "^5.0.0",
    "fastify": "^5.0.0",
    "kysely": "^0.27.0",
    "pg": "^8.0.0",
    "better-auth": "^1.4.0",
    "bullmq": "^5.0.0",
    "ioredis": "^5.0.0",
    "zod": "^3.23.0",
    "pino": "^9.0.0",
    "langchain": "^0.3.0",
    "@langchain/openai": "^0.3.0",
    "@langchain/community": "^0.3.0",
    "@microsoft/microsoft-graph-client": "^3.0.0",
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

---

## Type-Safe Client/Server: tRPC

| Aspect | Details |
|--------|---------|
| Package | `@trpc/server`, `@trpc/client`, `@trpc/react-query` |
| Why | End-to-end type safety without codegen |

**Why tRPC over REST + OpenAPI:**

1. **Zero codegen** - Change backend, frontend types update instantly
2. **Full inference** - Input/output types flow automatically
3. **React Query integration** - Built-in caching, mutations, optimistic updates
4. **Streaming support** - SSE subscriptions for chat
5. **Fastify adapter** - Works with our backend choice

**Example:**

```typescript
// packages/shared/src/schemas.ts
import { z } from "zod";

export const searchSchema = z.object({
  query: z.string().min(1),
  filters: z.object({
    dateFrom: z.date().optional(),
    documentTypes: z.array(z.string()).optional(),
  }).optional(),
  limit: z.number().min(1).max(100).default(20),
});

// apps/api/src/routers/search.ts
import { router, protectedProcedure } from "../trpc";
import { searchSchema } from "@law-ai/shared";

export const searchRouter = router({
  search: protectedProcedure
    .input(searchSchema)
    .query(async ({ input, ctx }) => {
      const results = await searchService.search(input, ctx.user, ctx.tenant);
      return results; // Type flows to client automatically
    }),
});

// apps/web/src/pages/Search.tsx
import { trpc } from "../utils/trpc";

function SearchPage() {
  const { data, isLoading } = trpc.search.search.useQuery({
    query: "contract terms",
    limit: 10,
  }); // Full type inference on `data`
}
```

---

## Database: Kysely

| Aspect | Details |
|--------|---------|
| Package | `kysely`, `pg` |
| Why | Type-safe SQL, lightweight, great for pgvector |

**Why Kysely over Prisma/Drizzle:**

1. **Close to SQL** - Query builder, not ORM abstraction
2. **Type-safe** - Full TypeScript inference
3. **pgvector friendly** - Easy raw SQL for vector operations
4. **Lightweight** - No heavy runtime, no codegen step
5. **Flexible** - Can drop to raw SQL when needed

**Example:**

```typescript
// packages/db/src/schema.ts
import { Generated, ColumnType } from "kysely";

export interface Database {
  tenants: TenantsTable;
  users: UsersTable;
  documents: DocumentsTable;
  document_chunks: DocumentChunksTable;
}

export interface TenantsTable {
  id: Generated<string>;
  slug: string;
  name: string;
  status: "active" | "suspended" | "cancelled";
  created_at: Generated<Date>;
}

export interface DocumentChunksTable {
  id: Generated<string>;
  document_id: string;
  tenant_id: string;
  content: string;
  embedding: number[] | null; // pgvector
  chunk_index: number;
}

// packages/db/src/db.ts
import { Kysely, PostgresDialect } from "kysely";
import { Pool } from "pg";
import { Database } from "./schema";

export const db = new Kysely<Database>({
  dialect: new PostgresDialect({
    pool: new Pool({ connectionString: process.env.DATABASE_URL }),
  }),
});

// Type-safe queries
const docs = await db
  .selectFrom("documents")
  .where("tenant_id", "=", tenantId)
  .where("status", "=", "indexed")
  .selectAll()
  .execute();

// Raw SQL for pgvector
const similar = await db
  .selectFrom("document_chunks")
  .where("tenant_id", "=", tenantId)
  .select(["id", "content", "document_id"])
  .select(sql<number>`embedding <=> ${embedding}`.as("distance"))
  .orderBy("distance")
  .limit(20)
  .execute();
```

---

## Frontend: Vite + React + Radix UI + styled-components

| Aspect | Details |
|--------|---------|
| Bundler | Vite |
| Framework | React 18 |
| UI Primitives | Radix UI (unstyled, accessible) |
| Styling | styled-components |
| State | TanStack Query (via tRPC) |

**Why Radix UI + styled-components:**

1. **Radix primitives** - Accessible, keyboard nav, screen reader support, unstyled
2. **styled-components** - Familiar CSS-in-JS, full styling control
3. **No utility classes** - Write actual CSS
4. **Theming** - styled-components ThemeProvider for design tokens

**Theming:**

```typescript
// apps/web/src/styles/theme.ts
export const theme = {
  colors: {
    background: "#ffffff",
    foreground: "#0a0a0a",
    primary: "#3b82f6",
    primaryForeground: "#ffffff",
    muted: "#f5f5f5",
    border: "#e5e5e5",
  },
  radii: {
    sm: "4px",
    md: "8px",
    lg: "12px",
  },
  fonts: {
    body: "Inter, system-ui, sans-serif",
    mono: "JetBrains Mono, monospace",
  },
};

// apps/web/src/App.tsx
import { ThemeProvider } from "styled-components";
import { theme } from "./styles/theme";

function App() {
  return (
    <ThemeProvider theme={theme}>
      {/* ... */}
    </ThemeProvider>
  );
}
```

**Radix + styled-components example:**

```typescript
import * as Dialog from "@radix-ui/react-dialog";
import styled from "styled-components";

const Overlay = styled(Dialog.Overlay)`
  background: rgba(0, 0, 0, 0.5);
  position: fixed;
  inset: 0;
`;

const Content = styled(Dialog.Content)`
  background: ${({ theme }) => theme.colors.background};
  border-radius: ${({ theme }) => theme.radii.lg};
  padding: 24px;
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.15);
`;
```

---

## Job Queue: BullMQ

| Aspect | Details |
|--------|---------|
| Package | `bullmq` |
| Backend | Redis |
| Why | Background processing for sync, embeddings |

**What needs background processing:**

- Document sync from M365 (can take hours for initial)
- Embedding generation (batched API calls)
- ACL expansion (group membership resolution)
- Webhook processing
- Email notifications

**Embedding Batching:**

```typescript
import { Queue, Worker } from "bullmq";

const embeddingQueue = new Queue("embeddings", {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
  },
});

// Batch processor - accumulate chunks, flush in batches
const worker = new Worker("embeddings", async (job) => {
  const { chunks } = job.data; // Array of chunks to embed
  
  // OpenAI supports up to 2048 texts per call
  // We batch at 100-500 for balance of efficiency and memory
  const BATCH_SIZE = 100;
  
  for (let i = 0; i < chunks.length; i += BATCH_SIZE) {
    const batch = chunks.slice(i, i + BATCH_SIZE);
    const embeddings = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: batch.map(c => c.content),
    });
    
    // Store embeddings
    await db.transaction().execute(async (trx) => {
      for (let j = 0; j < batch.length; j++) {
        await trx
          .updateTable("document_chunks")
          .set({ embedding: embeddings.data[j].embedding })
          .where("id", "=", batch[j].id)
          .execute();
      }
    });
  }
}, { connection: redis, concurrency: 2 });
```

---

## Logging: Pino → CloudWatch

| Aspect | Details |
|--------|---------|
| Package | `pino`, `pino-pretty` (dev) |
| Production | JSON logs → CloudWatch Logs |

**Setup:**

```typescript
// apps/api/src/logger.ts
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  ...(process.env.NODE_ENV === "development" && {
    transport: {
      target: "pino-pretty",
      options: { colorize: true },
    },
  }),
});

// Fastify uses Pino natively
const app = fastify({ logger });

// Structured logging
logger.info({ tenantId, userId, query }, "Search executed");
logger.error({ err, documentId }, "Document processing failed");
```

**CloudWatch:**

ECS Fargate automatically sends stdout/stderr to CloudWatch Logs. Pino's JSON format makes logs searchable via CloudWatch Logs Insights:

```sql
-- CloudWatch Logs Insights query
fields @timestamp, tenantId, userId, message
| filter level = "error"
| sort @timestamp desc
| limit 100
```

---

## Authentication: Better Auth

| Aspect | Details |
|--------|---------|
| Package | `better-auth` |
| Why | Multi-tenant, framework-agnostic, SSO-ready |

**Setup with Fastify:**

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

## RAG & AI: LangChain.js

| Aspect | Details |
|--------|---------|
| Package | `langchain`, `@langchain/openai`, `@langchain/community` |
| Why | Comprehensive RAG pipeline, streaming, pgvector support |

**What LangChain handles:**

| Task | LangChain Component |
|------|---------------------|
| Chunking | `RecursiveCharacterTextSplitter` |
| Embeddings | `OpenAIEmbeddings` |
| Vector store | `PGVectorStore` |
| Hybrid search | `EnsembleRetriever` |
| RAG chain | `RetrievalQAChain` |
| Streaming | `streamEvents` |

---

## Document Extraction

| Package | Purpose | Notes |
|---------|---------|-------|
| `pdfjs-dist` | PDF text extraction | Mozilla, industry standard |
| `tesseract.js` | OCR for scanned docs | WebAssembly Tesseract |
| `mammoth` | DOCX to HTML/text | Simple, preserves structure |
| `mailparser` | Email parsing | MIME, attachments, streaming |
| `html-to-text` | HTML stripping | Clean text from emails |

---

## Embedding Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| Model | `text-embedding-3-small` | Cost-effective, good quality |
| Dimensions | 1536 | Default for this model |
| Batch size | 100-500 | Balance efficiency and memory |

**Cost Estimate:**
- ~$0.02 per 1M tokens
- Average document: ~2000 tokens
- 10,000 documents: ~$0.40

---

## What We're NOT Using

### Next.js

**Why not:** No SSR needed for auth-gated SPA. Vite + separate Fastify API gives cleaner separation, better for long-running workers.

### Prisma/Drizzle

**Why not:** Kysely is closer to SQL, better for pgvector raw queries, lighter weight.

### Bun

**Why not:** Node.js has better ecosystem compatibility for BullMQ, Better Auth, etc. Stability over speed.

### CASL

**Why not:** Our ACL model is simple (user in allowed_user_ids array). CASL adds overhead we don't need.

---

## Project Structure

```
law-ai-platform/
├── apps/
│   ├── web/                    # Vite + React frontend
│   │   ├── src/
│   │   │   ├── components/     # shadcn/ui components
│   │   │   ├── pages/
│   │   │   ├── utils/
│   │   │   │   └── trpc.ts     # tRPC client setup
│   │   │   └── main.tsx
│   │   ├── index.html
│   │   └── vite.config.ts
│   │
│   └── api/                    # Fastify + tRPC backend
│       ├── src/
│       │   ├── routers/        # tRPC routers
│       │   ├── services/       # Business logic
│       │   ├── workers/        # BullMQ workers
│       │   ├── connectors/     # M365, etc.
│       │   ├── trpc.ts         # tRPC setup
│       │   └── index.ts        # Fastify entry
│       └── tsconfig.json
│
├── packages/
│   ├── db/                     # Kysely schema + migrations
│   │   ├── src/
│   │   │   ├── schema.ts       # Type definitions
│   │   │   ├── db.ts           # Kysely instance
│   │   │   └── migrations/
│   │   └── kysely.config.ts
│   │
│   └── shared/                 # Shared types + schemas
│       └── src/
│           ├── schemas/        # Zod schemas
│           └── types/          # Shared TypeScript types
│
├── pnpm-workspace.yaml
├── package.json
└── tsconfig.base.json
```

---

## Development Setup

```bash
# Install dependencies
pnpm install

# Environment variables
cp .env.example .env.local

# Start services (Docker)
docker-compose up -d  # PostgreSQL + Redis

# Run migrations
pnpm --filter @law-ai/db migrate:latest

# Start dev servers
pnpm dev  # Runs both web and api in parallel
```

### Environment Variables

```bash
# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/law_ai

# Redis
REDIS_URL=redis://localhost:6379

# Auth
BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=http://localhost:3001

# OpenAI
OPENAI_API_KEY=sk-...

# Stripe
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# M365 Platform Auth (for "Login with Microsoft")
MS_CLIENT_ID=...
MS_CLIENT_SECRET=...

# M365 Connector (separate app for data access)
MS_CONNECTOR_CLIENT_ID=...
MS_CONNECTOR_CLIENT_SECRET=...

# Token encryption
TOKEN_ENCRYPTION_KEY=...  # 32-byte hex

# AWS (production)
AWS_REGION=ap-southeast-2
```

---

## Package Version Policy

- **Pin major versions** in package.json
- **Renovate/Dependabot** for automated updates
- **Weekly update review** for security patches
