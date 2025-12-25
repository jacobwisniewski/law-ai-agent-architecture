# Law Firm AI Platform - Architecture Documentation

A knowledge and productivity platform for law firms. Unified search and AI chat across firm documents and email with strong permission controls.

## Product Vision

- **Unified Search**: Single search across documents, emails, and matter workspaces
- **AI Chat**: RAG-powered Q&A with citations linking to source documents
- **Permission-Aware**: Users only see content they have access to (ACL trimming)
- **Multi-Tenant SaaS**: Self-serve onboarding, per-seat pricing

## Key Decisions

| Aspect | Decision |
|--------|----------|
| Cloud | AWS |
| Language | TypeScript |
| LLM | OpenAI GPT-4.1 |
| Vector DB | pgvector (PostgreSQL) |
| First Connector | M365 (SharePoint, OneDrive, Outlook) |
| Auth | Better Auth (email/password MVP, SSO post-MVP) |
| RAG Framework | LangChain.js |
| Deployment | SaaS only (initially) |
| Pricing | Per-seat + usage credits |

## Documentation Index

### Core Architecture
- [Database Schema](./architecture/00-database-schema.md) - Consolidated SQL schema for all tables
- [Architecture Overview](./architecture/01-overview.md) - System components and data flow
- [Connector Framework](./architecture/02-connectors.md) - Generic connector interface, M365 implementation
- [Email Ingestion](./architecture/03-email-ingestion.md) - Email-specific handling and threading

### Security & Access
- [ACL and Security](./architecture/04-acl-security.md) - Permission sync, caching, query-time filtering
- [Auth Architecture](./architecture/12-auth-architecture.md) - Platform auth, connector OAuth, identity linking

### Search & AI
- [Search and RAG](./architecture/05-search-rag.md) - Hybrid search, retrieval, chat pipeline
- [Citations](./architecture/06-citations.md) - Deep linking to source documents

### Platform
- [Multi-Tenancy](./architecture/07-multi-tenancy.md) - Tenant isolation, data model
- [Billing and Pricing](./architecture/08-billing.md) - Metering, Stripe integration
- [Onboarding Flow](./architecture/09-onboarding.md) - Self-serve setup wizard

### Implementation
- [Implementation Phases](./architecture/10-phases.md) - MVP breakdown and milestones
- [Tech Stack & Packages](./architecture/11-tech-stack.md) - Package decisions and setup

## Tech Stack Summary

```
Frontend:       Next.js (React)
API:            Next.js API routes or Fastify
Database:       PostgreSQL + pgvector
Cache:          Redis
Queue:          BullMQ
Auth:           Better Auth (multi-tenant, SSO-ready)
RAG:            LangChain.js (@langchain/openai, @langchain/community)
Billing:        Stripe
M365:           @microsoft/microsoft-graph-client
Doc Extraction: pdfjs-dist, mammoth, mailparser
Infrastructure: AWS (ECS/Lambda, RDS, ElastiCache)
```

See [Tech Stack & Packages](./architecture/11-tech-stack.md) for detailed package decisions.
