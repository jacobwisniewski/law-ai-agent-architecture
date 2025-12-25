# Implementation Phases

## Overview

Breaking down the MVP into manageable phases. Each phase delivers usable functionality.

## Phase 0: Foundation (Week 1-2)

### Goals
- Project setup
- Core infrastructure
- Basic auth

### Deliverables

```
□ Project scaffolding
  ├── Next.js app with TypeScript
  ├── Database schema (Drizzle/Prisma)
  ├── Redis connection
  └── Environment configuration

□ Database setup
  ├── PostgreSQL with pgvector extension
  ├── Core tables: tenants, users, sessions
  └── Migration system

□ Authentication
  ├── Email/password auth
  ├── Session management
  ├── Basic middleware (auth, tenant context)
  └── Login/logout UI

□ Tenant basics
  ├── Tenant CRUD
  ├── User CRUD within tenant
  └── Tenant-scoped queries
```

### Tech Decisions
- **ORM:** Drizzle (lighter) or Prisma (more features)
- **Auth:** Better Auth (email/password + multi-tenant)
- **UI:** Tailwind + shadcn/ui

---

## Phase 1: M365 Connector (Week 3-4)

### Goals
- Connect to M365
- Sync documents and emails
- Store in database

### Deliverables

```
□ M365 OAuth flow
  ├── App registration guide
  ├── OAuth consent flow
  ├── Token storage (encrypted)
  └── Token refresh logic

□ Document sync
  ├── List SharePoint/OneDrive files
  ├── Delta sync with tokens
  ├── Download document content
  └── Store metadata in DB

□ Email sync
  ├── List Outlook messages
  ├── Delta sync
  ├── Download email content + attachments
  └── Store in DB with thread linking

□ Background jobs
  ├── BullMQ setup
  ├── Sync job workers
  ├── Retry logic
  └── Progress tracking

□ Admin UI
  ├── Connect M365 button
  ├── Sync status display
  └── Error visibility
```

### Key Risks
- M365 Graph API rate limits
- Large mailbox initial sync time
- Token expiry handling

---

## Phase 2: Ingestion Pipeline (Week 5-6)

### Goals
- Extract text from documents
- Chunk and embed content
- Build searchable index

### Deliverables

```
□ Document extraction
  ├── PDF extraction (pdfjs-dist)
  ├── DOCX extraction (mammoth)
  ├── Email HTML → text
  └── Attachment extraction

□ Chunking
  ├── LangChain.js text splitters
  ├── Overlap handling
  ├── Metadata preservation (page, section)
  └── Chunk storage schema

□ Embedding
  ├── LangChain.js OpenAIEmbeddings
  ├── Batch embedding for efficiency
  ├── Store vectors in pgvector
  └── Handle API errors/retries

□ ACL sync
  ├── Fetch M365 permissions
  ├── Group expansion
  ├── Store ACL mappings
  └── Link ACLs to documents

□ Indexing
  ├── pgvector index creation
  ├── Full-text search index
  └── Index maintenance jobs
```

### Key Risks
- Embedding API costs (estimate: ~$10-50 for initial large sync)
- Complex document layouts (tables, images)
- ACL expansion for large groups

---

## Phase 3: Search (Week 7-8)

### Goals
- Hybrid search working
- ACL filtering
- Search UI

### Deliverables

```
□ Search API
  ├── Query endpoint
  ├── Keyword search (PostgreSQL FTS)
  ├── Vector search (pgvector)
  ├── Reciprocal Rank Fusion
  └── Result formatting

□ ACL filtering
  ├── User permission lookup
  ├── Redis ACL cache
  ├── Filter search results
  └── Cache invalidation

□ Search UI
  ├── Search bar component
  ├── Results list
  ├── Document preview cards
  ├── Filters (date, type, source)
  └── Click-to-open source link

□ Performance
  ├── Query latency monitoring
  ├── Caching layer
  └── Index tuning
```

### Key Metrics
- Search latency < 500ms p95
- ACL accuracy 100% (no unauthorized results)

---

## Phase 4: RAG & Chat (Week 9-10)

### Goals
- AI-powered Q&A
- Citations working
- Chat UI

### Deliverables

```
□ RAG pipeline
  ├── LangChain.js RetrievalQAChain
  ├── Retrieval (reuse search)
  ├── Context building
  ├── Prompt construction
  └── GPT-4.1 integration

□ Citation handling
  ├── Citation extraction from response
  ├── Deep link generation
  ├── Citation metadata storage
  └── Snippet extraction

□ Streaming
  ├── SSE/WebSocket setup
  ├── Token streaming from OpenAI
  ├── Progressive UI update
  └── Source pre-loading

□ Chat UI
  ├── Chat interface component
  ├── Message history display
  ├── Citation display (inline + list)
  ├── Source preview on hover
  └── Copy/share functionality

□ Chat history
  ├── Conversation storage
  ├── History sidebar
  └── Resume conversation
```

### Key Metrics
- Time to first token < 1s
- Citation accuracy (manual QA)

---

## Phase 5: Billing & Onboarding (Week 11-12)

### Goals
- Self-serve sign-up
- Payment working
- Usage metering

### Deliverables

```
□ Stripe integration
  ├── Product/price setup in Stripe
  ├── Checkout flow
  ├── Webhook handling
  ├── Customer portal link
  └── Invoice display

□ Usage metering
  ├── Event tracking middleware
  ├── Credit counting
  ├── Quota enforcement
  ├── Overage reporting
  └── Usage dashboard

□ Onboarding flow
  ├── Sign-up page
  ├── Create org wizard
  ├── Plan selection
  ├── M365 connection (reuse Phase 1)
  ├── Sync progress page
  └── Welcome tour

□ Admin features
  ├── User invitation
  ├── User management
  ├── Billing settings
  └── Connector management
```

---

## Phase 6: Polish & Launch Prep (Week 13-14)

### Goals
- Production readiness
- Documentation
- Soft launch

### Deliverables

```
□ Production infrastructure
  ├── AWS setup (ECS/RDS/ElastiCache)
  ├── CI/CD pipeline
  ├── Monitoring (CloudWatch/Datadog)
  ├── Alerting
  └── Backup strategy

□ Security hardening
  ├── Security audit
  ├── Penetration testing (basic)
  ├── Rate limiting
  ├── Input validation review
  └── Secrets management

□ Documentation
  ├── User guide
  ├── Admin guide
  ├── API documentation
  └── M365 setup guide

□ Testing
  ├── E2E test suite
  ├── Load testing
  ├── ACL testing scenarios
  └── Cross-browser testing

□ Launch prep
  ├── Landing page
  ├── Pricing page
  ├── Terms of service
  ├── Privacy policy
  └── Support channel setup
```

---

## MVP Feature Summary

### In Scope (MVP)

| Feature | Phase |
|---------|-------|
| Email/password auth | 0 |
| Multi-tenant data isolation | 0 |
| M365 document sync | 1 |
| M365 email sync | 1 |
| Document extraction (PDF, DOCX) | 2 |
| Chunking + embedding | 2 |
| ACL sync from M365 | 2 |
| Hybrid search | 3 |
| ACL-filtered results | 3 |
| RAG with citations | 4 |
| Streaming chat UI | 4 |
| Stripe billing | 5 |
| Self-serve onboarding | 5 |
| Usage metering | 5 |

### Out of Scope (Post-MVP)

| Feature | Reason |
|---------|--------|
| SSO (Okta, Azure AD) | Adds complexity, not needed for pilot |
| Google Workspace connector | M365 first |
| iManage/NetDocuments | Complex, niche |
| Document editing/write-back | Risk, complexity |
| Matter management | Nice-to-have |
| Mobile app | Web-first |
| On-premise deployment | SaaS only initially |
| Advanced analytics | Post-launch iteration |

---

## Timeline Summary

| Phase | Duration | Cumulative |
|-------|----------|------------|
| 0: Foundation | 2 weeks | Week 2 |
| 1: M365 Connector | 2 weeks | Week 4 |
| 2: Ingestion | 2 weeks | Week 6 |
| 3: Search | 2 weeks | Week 8 |
| 4: RAG & Chat | 2 weeks | Week 10 |
| 5: Billing & Onboarding | 2 weeks | Week 12 |
| 6: Polish & Launch | 2 weeks | Week 14 |

**Total: ~14 weeks to MVP**

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| M365 API complexity | Start early, have fallback to simpler sync |
| Embedding costs spike | Monitor closely, add budget alerts |
| ACL edge cases | Extensive testing, conservative defaults (deny if uncertain) |
| Single developer bandwidth | Prioritize ruthlessly, cut scope if needed |
| Pilot firm delays | Have backup pilot candidates |

## Success Criteria for MVP

1. **Functional:** User can search and chat across M365 docs/emails
2. **Secure:** ACL filtering works correctly (verified by testing)
3. **Usable:** < 10 min from sign-up to first search
4. **Scalable:** Handles 10k documents, 50k emails per tenant
5. **Monetizable:** Billing works, can charge pilot customers
