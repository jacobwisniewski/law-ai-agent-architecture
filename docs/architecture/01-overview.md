# Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MULTI-TENANT PLATFORM                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │   Tenant A   │     │   Tenant B   │     │   Tenant C   │                │
│  │   (Firm 1)   │     │   (Firm 2)   │     │   (Firm 3)   │                │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘                │
│         │                    │                    │                         │
├─────────┴────────────────────┴────────────────────┴─────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    AUTH LAYER (Better Auth)                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │   │
│  │  │ Email/  │  │Microsoft│  │  SSO    │  │  2FA    │                 │   │
│  │  │ Password│  │  OAuth  │  │(post-MVP│  │(post-MVP│                 │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         API LAYER (Next.js / Fastify)                │   │
│  │                                                                      │   │
│  │   /search    /chat    /documents    /connectors    /admin           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │          │           │              │                             │
│         ▼          ▼           ▼              ▼                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      CORE SERVICES                                   │   │
│  │                                                                      │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐         │   │
│  │  │ Search Service │  │  Chat/RAG      │  │  ACL Service   │         │   │
│  │  │                │  │  Service       │  │                │         │   │
│  │  │ • Query parse  │  │ • Retrieval    │  │ • Permission   │         │   │
│  │  │ • Hybrid search│  │ • Context build│  │   resolution   │         │   │
│  │  │ • ACL filter   │  │ • LLM call     │  │ • ACL sync     │         │   │
│  │  │ • Ranking      │  │ • Citations    │  │ • Cache        │         │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      DATA LAYER                                      │   │
│  │                                                                      │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐         │   │
│  │  │   PostgreSQL   │  │   pgvector     │  │     Redis      │         │   │
│  │  │                │  │                │  │                │         │   │
│  │  │ • Tenants      │  │ • Embeddings   │  │ • ACL cache    │         │   │
│  │  │ • Users        │  │ • Hybrid search│  │ • Session      │         │   │
│  │  │ • Documents    │  │                │  │ • Rate limit   │         │   │
│  │  │ • ACL mappings │  │                │  │                │         │   │
│  │  │ • Audit logs   │  │                │  │                │         │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow

### 1. Ingestion Flow

```
External Source (M365)
        │
        ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│   Connector   │────▶│  Job Queue    │────▶│   Workers     │
│               │     │  (BullMQ)     │     │               │
│ • OAuth auth  │     │               │     │ • Extract     │
│ • Delta sync  │     │ • Retry logic │     │ • Chunk       │
│ • Webhooks    │     │ • Rate limit  │     │ • Embed       │
└───────────────┘     └───────────────┘     └───────────────┘
                                                    │
                                                    ▼
                                            ┌───────────────┐
                                            │   pgvector    │
                                            │               │
                                            │ • Store chunk │
                                            │ • Store ACL   │
                                            │ • Index       │
                                            └───────────────┘
```

### 2. Query Flow

```
User Query
    │
    ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  Auth Check   │────▶│  ACL Service  │────▶│ Search/RAG    │
│               │     │               │     │               │
│ • Verify JWT  │     │ • Get user    │     │ • Hybrid      │
│ • Get tenant  │     │   permissions │     │   search      │
│ • Get user    │     │ • Cache check │     │ • Filter ACL  │
└───────────────┘     └───────────────┘     └───────────────┘
                                                    │
                                                    ▼
                                            ┌───────────────┐
                                            │   Response    │
                                            │               │
                                            │ • Results     │
                                            │ • Citations   │
                                            │ • AI answer   │
                                            └───────────────┘
```

## Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| **Auth Layer** | Authenticate users via Better Auth (email/password MVP, SSO post-MVP) |
| **API Layer** | REST/GraphQL endpoints, request validation, rate limiting |
| **Search Service** | Hybrid keyword + vector search with ACL filtering |
| **Chat/RAG Service** | Retrieval, context building, LLM calls, citation generation |
| **ACL Service** | Permission resolution, caching, sync from source systems |
| **Connector Framework** | Abstract interface for data sources, M365 implementation |
| **Ingestion Pipeline** | Document extraction, chunking, embedding, indexing |
| **PostgreSQL** | Relational data (tenants, users, documents, audit logs) |
| **pgvector** | Vector embeddings and similarity search |
| **Redis** | Caching (ACLs, sessions), rate limiting, job queue backend |

## Key Design Principles

1. **Tenant Isolation**: All data queries include `tenant_id` filter. No cross-tenant data leakage.

2. **ACL-First**: Permissions checked before any data access. Cached for performance, invalidated on changes.

3. **Connector Abstraction**: Generic interface allows adding new data sources without core changes.

4. **Async Ingestion**: Heavy processing (extraction, embedding) happens in background workers.

5. **Audit Everything**: All data access logged for compliance and debugging.

## Infrastructure (AWS)

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS                                  │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   ECS /     │  │    RDS      │  │ ElastiCache │         │
│  │   Lambda    │  │ (Postgres)  │  │  (Redis)    │         │
│  │             │  │             │  │             │         │
│  │ • API       │  │ • Data      │  │ • Cache     │         │
│  │ • Workers   │  │ • pgvector  │  │ • Queue     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│         │                │                │                 │
│         └────────────────┼────────────────┘                 │
│                          │                                  │
│                    ┌─────────────┐                         │
│                    │     VPC     │                         │
│                    │   Private   │                         │
│                    └─────────────┘                         │
│                          │                                  │
│                    ┌─────────────┐                         │
│                    │     ALB     │                         │
│                    │  (Public)   │                         │
│                    └─────────────┘                         │
│                          │                                  │
│                    ┌─────────────┐                         │
│                    │ CloudFront  │                         │
│                    │    CDN      │                         │
│                    └─────────────┘                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
