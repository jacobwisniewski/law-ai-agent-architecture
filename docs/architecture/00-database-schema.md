# Database Schema

## Overview

Consolidated database schema. All tables use `tenant_id` for isolation. PostgreSQL with pgvector extension.

**Related Docs:**
- [Multi-Tenancy](./07-multi-tenancy.md) - Isolation strategy
- [Auth Architecture](./12-auth-architecture.md) - Identity linking tables
- [ACL & Security](./04-acl-security.md) - Permission tables

## Entity Relationship

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CORE ENTITIES                                      │
│                                                                             │
│  tenants ─────────┬──────────────────────────────────────────────────────┐  │
│     │             │                                                      │  │
│     │             ▼                                                      │  │
│     │        subscriptions ──► invoices                                  │  │
│     │             │                                                      │  │
│     │             ▼                                                      │  │
│     │        usage_events                                                │  │
│     │                                                                    │  │
│     ├──► users ─────┬──► external_identities                            │  │
│     │               │                                                    │  │
│     │               ├──► sessions                                        │  │
│     │               │                                                    │  │
│     │               └──► user_preferences                                │  │
│     │                                                                    │  │
│     ├──► connector_credentials                                           │  │
│     │                                                                    │  │
│     ├──► documents ──────┬──► document_chunks (pgvector)                 │  │
│     │                    │                                               │  │
│     │                    └──► document_source_emails (junction)          │  │
│     │                                                                    │  │
│     ├──► emails ─────────┬──► email_attachments                          │  │
│     │                    │                                               │  │
│     │                    └──► email_chunks (pgvector)                    │  │
│     │                                                                    │  │
│     ├──► expanded_acls                                                   │  │
│     │                                                                    │  │
│     ├──► external_groups                                                 │  │
│     │                                                                    │  │
│     ├──► sync_jobs                                                       │  │
│     │                                                                    │  │
│     ├──► conversations ──► messages                                      │  │
│     │                                                                    │  │
│     ├──► notifications                                                   │  │
│     │                                                                    │  │
│     └──► audit_logs                                                      │  │
│                                                                             │
│  tenant_sso_configs (1:1 with tenants, post-MVP)                           │
│                                                                             │
│  api_keys (per tenant)                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Tables

### tenants

```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug VARCHAR(100) NOT NULL UNIQUE,
  name VARCHAR(255) NOT NULL,
  
  status VARCHAR(50) NOT NULL DEFAULT 'active',
    -- 'active' | 'suspended' | 'cancelled'
  
  plan_id VARCHAR(50) NOT NULL DEFAULT 'starter',
  seats_included INTEGER NOT NULL DEFAULT 5,
  seats_used INTEGER NOT NULL DEFAULT 0,
  
  -- Stripe (nullable until payment setup)
  stripe_customer_id VARCHAR(255),
  stripe_subscription_id VARCHAR(255),
  
  -- Settings (JSONB for flexibility)
  settings JSONB NOT NULL DEFAULT '{
    "emailIngestionEnabled": true,
    "maxDocumentSizeMb": 50,
    "retentionDays": 365,
    "syncIntervalMinutes": 15,
    "webhooksEnabled": true
  }',
  
  -- Onboarding
  tour_completed BOOLEAN NOT NULL DEFAULT FALSE,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_stripe_customer ON tenants(stripe_customer_id);
```

### users

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  
  -- Auth (for email/password users)
  password_hash VARCHAR(255),
  email_verified BOOLEAN NOT NULL DEFAULT FALSE,
  
  -- Social auth IDs (for platform login)
  microsoft_id VARCHAR(255),
  google_id VARCHAR(255),
  
  -- SSO (post-MVP)
  sso_provider_id VARCHAR(255),
  
  -- Status
  status VARCHAR(50) NOT NULL DEFAULT 'invited',
    -- 'invited' | 'active' | 'suspended'
  role VARCHAR(50) NOT NULL DEFAULT 'member',
    -- 'admin' | 'member'
  
  -- Platform admin (cross-tenant)
  is_platform_admin BOOLEAN NOT NULL DEFAULT FALSE,
  
  -- Timestamps
  invited_at TIMESTAMPTZ,
  activated_at TIMESTAMPTZ,
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(tenant_id, email);
CREATE INDEX idx_users_microsoft_id ON users(microsoft_id) WHERE microsoft_id IS NOT NULL;
CREATE INDEX idx_users_google_id ON users(google_id) WHERE google_id IS NOT NULL;
```

### user_preferences

```sql
CREATE TABLE user_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE UNIQUE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  default_search_filters JSONB,
  email_notifications BOOLEAN NOT NULL DEFAULT TRUE,
  timezone VARCHAR(100) NOT NULL DEFAULT 'UTC',
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_preferences_user ON user_preferences(user_id);
```

### sessions

```sql
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  token_hash VARCHAR(255) NOT NULL UNIQUE,
  
  ip_address VARCHAR(45),
  user_agent TEXT,
  
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_active_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_user ON sessions(user_id);
CREATE INDEX idx_sessions_token ON sessions(token_hash);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);
```

---

## Auth & Identity Tables

### external_identities

Maps platform users to external system identities (for ACL resolution).

```sql
CREATE TABLE external_identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  provider VARCHAR(50) NOT NULL,
    -- 'm365' | 'netdocuments' | 'imanage' | 'google'
  external_id VARCHAR(255) NOT NULL,
  external_email VARCHAR(255),
  
  link_method VARCHAR(50) NOT NULL,
    -- 'email_match' | 'oauth' | 'admin_manual' | 'scim'
  linked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  linked_by UUID REFERENCES users(id),
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  
  UNIQUE(user_id, provider),
  UNIQUE(tenant_id, provider, external_id)
);

CREATE INDEX idx_external_identities_lookup 
  ON external_identities(tenant_id, provider, external_id);
```

### connector_credentials

OAuth tokens for accessing external systems.

```sql
CREATE TABLE connector_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  connector_type VARCHAR(50) NOT NULL,
    -- 'm365' | 'netdocuments' | 'imanage' | 'google'
  
  -- Tokens (encrypted with AES-256-GCM)
  access_token TEXT NOT NULL,
  refresh_token TEXT NOT NULL,
  token_expires_at TIMESTAMPTZ NOT NULL,
  
  -- Connector-specific config
  config JSONB NOT NULL DEFAULT '{}',
  
  -- Status
  status VARCHAR(50) NOT NULL DEFAULT 'active',
    -- 'active' | 'expired' | 'revoked' | 'error'
  last_used_at TIMESTAMPTZ,
  last_refresh_at TIMESTAMPTZ,
  last_error TEXT,
  
  -- Audit
  authorized_by UUID NOT NULL REFERENCES users(id),
  authorized_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  UNIQUE(tenant_id, connector_type)
);

CREATE INDEX idx_connector_credentials_tenant ON connector_credentials(tenant_id);
```

### external_groups

Cached group membership expansion for ACL resolution.

```sql
CREATE TABLE external_groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  provider VARCHAR(50) NOT NULL,
  external_group_id VARCHAR(255) NOT NULL,
  
  display_name VARCHAR(255),
  member_user_ids UUID[] NOT NULL DEFAULT '{}',
  
  last_synced_at TIMESTAMPTZ NOT NULL,
  member_count INTEGER NOT NULL DEFAULT 0,
  
  UNIQUE(tenant_id, provider, external_group_id)
);

CREATE INDEX idx_external_groups_members 
  ON external_groups USING GIN(member_user_ids);
```

### tenant_sso_configs

SSO configuration (post-MVP).

```sql
CREATE TABLE tenant_sso_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE UNIQUE,
  
  provider VARCHAR(50) NOT NULL,
    -- 'okta' | 'azure_ad' | 'onelogin' | 'custom_oidc'
  issuer_url VARCHAR(500) NOT NULL,
  client_id VARCHAR(255) NOT NULL,
  client_secret TEXT NOT NULL,  -- Encrypted
  
  -- SAML-specific
  saml_certificate TEXT,
  saml_entity_id VARCHAR(500),
  
  -- Behavior
  allow_password_login BOOLEAN NOT NULL DEFAULT TRUE,
  auto_provision BOOLEAN NOT NULL DEFAULT FALSE,
  allowed_domains TEXT[] NOT NULL DEFAULT '{}',
  
  -- Attribute mapping
  attribute_mapping JSONB NOT NULL DEFAULT '{}',
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Document Tables

### documents

```sql
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  connector_id UUID REFERENCES connector_credentials(id) ON DELETE SET NULL,
  
  -- External reference
  external_id VARCHAR(500) NOT NULL,
  external_url TEXT NOT NULL,
  
  -- Metadata
  title VARCHAR(1000) NOT NULL,
  mime_type VARCHAR(255) NOT NULL,
  size_bytes BIGINT NOT NULL DEFAULT 0,
  
  -- Content
  extracted_text TEXT,
  content_hash VARCHAR(64),  -- SHA-256
  
  -- Hierarchy
  parent_path TEXT,
  parent_id VARCHAR(500),
  
  -- Source type
  source_type VARCHAR(50) NOT NULL,
    -- 'sharepoint' | 'onedrive' | 'email_attachment' | 'netdocuments' | 'imanage'
  
  -- Timestamps
  source_created_at TIMESTAMPTZ,
  source_modified_at TIMESTAMPTZ,
  ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_synced_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  -- Processing
  status VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- 'pending' | 'processing' | 'indexed' | 'error'
  error_message TEXT,
  chunk_count INTEGER NOT NULL DEFAULT 0,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  UNIQUE(tenant_id, external_id)
);

CREATE INDEX idx_documents_tenant ON documents(tenant_id);
CREATE INDEX idx_documents_external ON documents(tenant_id, external_id);
CREATE INDEX idx_documents_hash ON documents(tenant_id, content_hash);
CREATE INDEX idx_documents_status ON documents(tenant_id, status);
```

### document_source_emails

Junction table for documents attached to multiple emails (deduplication).

```sql
CREATE TABLE document_source_emails (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  email_id UUID NOT NULL REFERENCES emails(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  UNIQUE(document_id, email_id)
);

CREATE INDEX idx_doc_source_emails_doc ON document_source_emails(document_id);
CREATE INDEX idx_doc_source_emails_email ON document_source_emails(email_id);
```

### document_chunks

Vector embeddings for document content.

```sql
CREATE TABLE document_chunks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  content TEXT NOT NULL,
  embedding vector(1536),  -- text-embedding-3-small
  
  -- Location metadata
  metadata JSONB NOT NULL DEFAULT '{}',
    -- { page?: number, section?: string, heading?: string, charStart: number, charEnd: number }
  
  -- Full-text search
  search_vector tsvector GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
  
  chunk_index INTEGER NOT NULL DEFAULT 0,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_document_chunks_tenant_doc ON document_chunks(tenant_id, document_id);
CREATE INDEX idx_document_chunks_search ON document_chunks USING GIN(search_vector);
CREATE INDEX idx_document_chunks_embedding ON document_chunks 
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

---

## Email Tables

### emails

```sql
CREATE TABLE emails (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  connector_id UUID REFERENCES connector_credentials(id) ON DELETE SET NULL,
  external_id VARCHAR(500) NOT NULL,
  
  -- Core fields
  subject TEXT NOT NULL,
  from_address JSONB NOT NULL,  -- { email: string, name?: string }
  to_addresses JSONB NOT NULL DEFAULT '[]',
  cc_addresses JSONB NOT NULL DEFAULT '[]',
  bcc_addresses JSONB NOT NULL DEFAULT '[]',
  
  received_at TIMESTAMPTZ NOT NULL,
  sent_at TIMESTAMPTZ,
  
  -- Threading
  thread_id VARCHAR(500),
  in_reply_to VARCHAR(500),
  references_ids JSONB NOT NULL DEFAULT '[]',
  
  -- Content
  body_text TEXT,
  body_html TEXT,
  
  -- Metadata
  importance VARCHAR(20) NOT NULL DEFAULT 'normal',
    -- 'low' | 'normal' | 'high'
  categories JSONB NOT NULL DEFAULT '[]',
  external_url TEXT NOT NULL,
  has_attachments BOOLEAN NOT NULL DEFAULT FALSE,
  
  -- ACL
  mailbox_owner_id UUID REFERENCES users(id),
  mailbox_owner_external_id VARCHAR(255),
  
  -- Processing
  status VARCHAR(50) NOT NULL DEFAULT 'pending',
  chunk_count INTEGER NOT NULL DEFAULT 0,
  
  ingested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_synced_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  UNIQUE(tenant_id, external_id)
);

CREATE INDEX idx_emails_tenant ON emails(tenant_id);
CREATE INDEX idx_emails_thread ON emails(tenant_id, thread_id);
CREATE INDEX idx_emails_received ON emails(tenant_id, received_at DESC);
CREATE INDEX idx_emails_mailbox ON emails(tenant_id, mailbox_owner_id);
```

### email_attachments

```sql
CREATE TABLE email_attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email_id UUID NOT NULL REFERENCES emails(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  name VARCHAR(500) NOT NULL,
  mime_type VARCHAR(255) NOT NULL,
  size_bytes BIGINT NOT NULL DEFAULT 0,
  is_inline BOOLEAN NOT NULL DEFAULT FALSE,
  
  -- Link to extracted document (if processed)
  document_id UUID REFERENCES documents(id) ON DELETE SET NULL,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_email_attachments_email ON email_attachments(email_id);
CREATE INDEX idx_email_attachments_doc ON email_attachments(document_id);
```

### email_chunks

Vector embeddings for email content.

```sql
CREATE TABLE email_chunks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email_id UUID NOT NULL REFERENCES emails(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  content TEXT NOT NULL,
  embedding vector(1536),
  
  metadata JSONB NOT NULL DEFAULT '{}',
  
  search_vector tsvector GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
  
  chunk_index INTEGER NOT NULL DEFAULT 0,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_email_chunks_tenant_email ON email_chunks(tenant_id, email_id);
CREATE INDEX idx_email_chunks_search ON email_chunks USING GIN(search_vector);
CREATE INDEX idx_email_chunks_embedding ON email_chunks 
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

---

## ACL Tables

### expanded_acls

Pre-computed ACLs with group expansion.

```sql
CREATE TABLE expanded_acls (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  resource_id UUID NOT NULL,  -- document.id or email.id
  resource_type VARCHAR(50) NOT NULL,
    -- 'document' | 'email'
  
  -- Flattened user IDs who can access
  allowed_user_ids UUID[] NOT NULL DEFAULT '{}',
  
  -- Original groups (for audit)
  source_groups TEXT[] NOT NULL DEFAULT '{}',
  
  -- Metadata
  expanded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expansion_version INTEGER NOT NULL DEFAULT 1,
  
  UNIQUE(tenant_id, resource_id, resource_type)
);

CREATE INDEX idx_expanded_acls_resource ON expanded_acls(tenant_id, resource_id);
CREATE INDEX idx_expanded_acls_users ON expanded_acls USING GIN(allowed_user_ids);
```

---

## Sync & Jobs

### sync_jobs

Track sync progress for connectors.

```sql
CREATE TABLE sync_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  connector_id UUID NOT NULL REFERENCES connector_credentials(id) ON DELETE CASCADE,
  
  status VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- 'pending' | 'in_progress' | 'completed' | 'failed'
  job_type VARCHAR(50) NOT NULL DEFAULT 'incremental',
    -- 'full' | 'incremental'
  
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  
  -- Progress
  documents_total INTEGER NOT NULL DEFAULT 0,
  documents_processed INTEGER NOT NULL DEFAULT 0,
  emails_total INTEGER NOT NULL DEFAULT 0,
  emails_processed INTEGER NOT NULL DEFAULT 0,
  
  -- Delta tokens (for incremental sync)
  delta_token TEXT,
  
  -- Errors
  error_count INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sync_jobs_tenant ON sync_jobs(tenant_id);
CREATE INDEX idx_sync_jobs_status ON sync_jobs(tenant_id, status);
CREATE INDEX idx_sync_jobs_latest ON sync_jobs(tenant_id, connector_id, created_at DESC);
```

---

## Chat & Conversations

### conversations

```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  title VARCHAR(255),  -- Auto-generated from first message
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_conversations_user ON conversations(tenant_id, user_id);
CREATE INDEX idx_conversations_recent ON conversations(tenant_id, user_id, updated_at DESC);
```

### messages

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  role VARCHAR(20) NOT NULL,
    -- 'user' | 'assistant'
  content TEXT NOT NULL,
  
  -- Citations (for assistant messages)
  citations JSONB,  -- Array of citation objects
  
  -- Metadata
  tokens_used INTEGER,
  model VARCHAR(100),
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at);
```

---

## Billing Tables

### subscriptions

```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE UNIQUE,
  
  stripe_customer_id VARCHAR(255) NOT NULL,
  stripe_subscription_id VARCHAR(255) NOT NULL,
  stripe_seat_item_id VARCHAR(255),
  
  plan_id VARCHAR(50) NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'active',
    -- 'active' | 'past_due' | 'cancelled'
  
  -- Billing cycle
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  
  -- Seats
  seats_quantity INTEGER NOT NULL DEFAULT 1,
  seats_price_id VARCHAR(255),
  
  -- Credits
  credits_included INTEGER NOT NULL DEFAULT 5000,
  credits_used INTEGER NOT NULL DEFAULT 0,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_stripe ON subscriptions(stripe_subscription_id);
```

### usage_events

```sql
CREATE TABLE usage_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  
  event_type VARCHAR(50) NOT NULL,
    -- 'search' | 'chat' | 'document_ingest' | 'email_ingest'
  credits INTEGER NOT NULL,
  
  metadata JSONB,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_usage_events_tenant ON usage_events(tenant_id, created_at DESC);
CREATE INDEX idx_usage_events_period ON usage_events(tenant_id, created_at);
```

### invoices

```sql
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  stripe_invoice_id VARCHAR(255) NOT NULL UNIQUE,
  
  period_start TIMESTAMPTZ NOT NULL,
  period_end TIMESTAMPTZ NOT NULL,
  
  seats_amount INTEGER NOT NULL DEFAULT 0,
  total_amount INTEGER NOT NULL DEFAULT 0,  -- In cents
  
  status VARCHAR(50) NOT NULL DEFAULT 'draft',
    -- 'draft' | 'open' | 'paid' | 'void'
  paid_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invoices_tenant ON invoices(tenant_id, created_at DESC);
```

---

## Notifications

### notifications

```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  type VARCHAR(50) NOT NULL,
    -- 'sync_complete' | 'connector_error' | 'payment_failed' | 'usage_warning' | etc.
  title VARCHAR(255) NOT NULL,
  message TEXT NOT NULL,
  action_url VARCHAR(500),
  
  read BOOLEAN NOT NULL DEFAULT FALSE,
  read_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(tenant_id, user_id, created_at DESC);
CREATE INDEX idx_notifications_unread ON notifications(tenant_id, user_id) WHERE NOT read;
```

---

## Audit Logs

### audit_logs

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  -- Actor
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  user_email VARCHAR(255),
  ip_address VARCHAR(45),
  user_agent TEXT,
  
  -- Action
  action VARCHAR(50) NOT NULL,
    -- 'search' | 'chat' | 'document_view' | 'login' | 'logout' | 'acl_check' | etc.
  resource_type VARCHAR(50),
    -- 'document' | 'email' | 'search' | 'chat'
  resource_id UUID,
  
  -- Details
  query TEXT,
  results_count INTEGER,
  resources_accessed UUID[],
  
  -- ACL context
  acl_decision VARCHAR(20),
    -- 'allowed' | 'denied'
  acl_reason TEXT,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Partitioned by month for retention management
CREATE INDEX idx_audit_logs_tenant ON audit_logs(tenant_id, created_at DESC);
CREATE INDEX idx_audit_logs_user ON audit_logs(tenant_id, user_id, created_at DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs(tenant_id, action, created_at DESC);
```

---

## API Keys

### api_keys

```sql
CREATE TABLE api_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  name VARCHAR(255) NOT NULL,
  key_hash VARCHAR(255) NOT NULL UNIQUE,  -- SHA-256 of the key
  key_prefix VARCHAR(10) NOT NULL,  -- First 8 chars for identification
  
  scopes JSONB NOT NULL DEFAULT '["read"]',
  
  last_used_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ
);

CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

---

## Migrations & Setup

### Enable pgvector

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### Row-Level Security (Optional)

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE emails ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_chunks ENABLE ROW LEVEL SECURITY;
-- ... etc for all tenant tables

-- Create policy (requires app.current_tenant to be set)
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

---

## Summary

| Table | Purpose | Key Indexes |
|-------|---------|-------------|
| `tenants` | Organizations | slug |
| `users` | Platform users | tenant_id, email |
| `sessions` | Auth sessions | token_hash |
| `external_identities` | User↔External ID mapping | provider + external_id |
| `connector_credentials` | OAuth tokens | tenant_id + type |
| `external_groups` | Group expansion cache | GIN on member_user_ids |
| `documents` | Ingested documents | external_id, content_hash |
| `document_chunks` | Vector embeddings | ivfflat on embedding |
| `emails` | Ingested emails | thread_id, received_at |
| `email_chunks` | Email embeddings | ivfflat on embedding |
| `expanded_acls` | Pre-computed ACLs | GIN on allowed_user_ids |
| `sync_jobs` | Sync progress | status, connector_id |
| `conversations` | Chat history | user_id |
| `messages` | Chat messages | conversation_id |
| `subscriptions` | Billing | stripe_subscription_id |
| `usage_events` | Usage metering | created_at |
| `audit_logs` | Security audit | action, created_at |
