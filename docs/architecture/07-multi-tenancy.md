# Multi-Tenancy

## Overview

The platform is multi-tenant from day one. Each law firm (tenant) has completely isolated data, configurations, and users. This doc covers the isolation model and data architecture.

## Tenant Model

```typescript
interface Tenant {
  id: string;                   // UUID
  slug: string;                 // URL-friendly identifier (e.g., "acme-law")
  name: string;                 // Display name
  
  // Status
  status: 'active' | 'suspended' | 'cancelled';
  createdAt: Date;
  
  // Subscription (see subscriptions table for full billing details)
  planId: string;
  seatsIncluded: number;
  seatsUsed: number;
  
  // Stripe (denormalized for quick access)
  stripeCustomerId?: string;
  stripeSubscriptionId?: string;
  
  // Onboarding
  tourCompleted: boolean;
  
  // Configuration
  settings: TenantSettings;
}

interface TenantSettings {
  // Branding
  logoUrl?: string;
  primaryColor?: string;
  
  // Features
  emailIngestionEnabled: boolean;
  maxDocumentSizeMb: number;
  retentionDays: number;
  
  // Sync
  syncIntervalMinutes: number;
  webhooksEnabled: boolean;
}

// SSO config stored separately - see tenant_sso_configs table in 00-database-schema.md
// Connector credentials stored separately - see connector_credentials table
```

## Data Isolation

### Database Strategy

**Approach: Shared database, tenant_id column**

All tables include a `tenant_id` column. This is simpler than separate schemas or databases per tenant, while still providing strong isolation.

```sql
-- Every table has tenant_id
CREATE TABLE documents (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  title TEXT NOT NULL,
  -- ...
);

-- All indexes include tenant_id
CREATE INDEX idx_documents_tenant ON documents(tenant_id);

-- Row-level security (optional, defense in depth)
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

### Query Enforcement

Every query must include tenant_id. Enforce at ORM level:

```typescript
// Drizzle/Prisma middleware example
async function withTenantScope<T>(
  tenantId: string,
  query: () => Promise<T>
): Promise<T> {
  // Validate UUID format to prevent injection
  if (!isValidUUID(tenantId)) {
    throw new Error('Invalid tenant ID');
  }
  
  // Set tenant context for RLS using parameterized query
  await db.execute(sql`SELECT set_config('app.current_tenant', ${tenantId}, true)`);
  
  try {
    return await query();
  } finally {
    await db.execute(sql`SELECT set_config('app.current_tenant', '', true)`);
  }
}

function isValidUUID(id: string): boolean {
  return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(id);
}

// Repository pattern
class DocumentRepository {
  constructor(private tenantId: string) {}
  
  async findById(id: string): Promise<Document | null> {
    return db.documents.findFirst({
      where: {
        id,
        tenantId: this.tenantId,  // Always scoped
      },
    });
  }
  
  async search(query: string): Promise<Document[]> {
    return db.documents.findMany({
      where: {
        tenantId: this.tenantId,  // Always scoped
        // ... search conditions
      },
    });
  }
}
```

### Vector Index Isolation

pgvector indexes are shared, but queries filter by tenant:

```sql
-- Single index for all tenants
CREATE INDEX idx_chunks_embedding 
  ON document_chunks 
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Query always filters by tenant
SELECT * FROM document_chunks
WHERE tenant_id = $1
ORDER BY embedding <=> $2
LIMIT 20;
```

**Consideration:** For very large tenants or strict isolation requirements, could partition table by tenant_id for better performance.

## User Model

See [Database Schema](./00-database-schema.md) for the authoritative SQL schema.

```typescript
interface User {
  id: string;
  tenantId: string;
  
  // Identity
  email: string;
  name: string;
  
  // Auth (for email/password users)
  passwordHash?: string;
  emailVerified: boolean;
  
  // Social auth IDs (for platform login)
  microsoftId?: string;
  googleId?: string;
  
  // SSO (post-MVP)
  ssoProviderId?: string;
  
  // Status
  status: 'invited' | 'active' | 'suspended';
  role: 'admin' | 'member';
  
  // Platform admin (cross-tenant)
  isPlatformAdmin: boolean;
  
  // Timestamps
  invitedAt?: Date;
  activatedAt?: Date;
  lastLoginAt?: Date;
  createdAt: Date;
  
  // External identity links (for ACL resolution) - see external_identities table
  // Preferences - see user_preferences table
}
```

### User Provisioning

Two modes:

1. **Manual invite:** Admin invites users by email
2. **Auto-provision:** Users created on first SSO login (if domain matches)

```typescript
async function handleSSOLogin(
  tenantId: string,
  profile: OIDCProfile
): Promise<User> {
  const tenant = await getTenant(tenantId);
  
  // Check if user exists
  let user = await db.users.findByEmail(tenantId, profile.email);
  
  if (!user) {
    // Check auto-provision settings
    if (!tenant.authConfig.autoProvision) {
      throw new Error('User not found and auto-provision disabled');
    }
    
    // Check domain allowlist
    const domain = profile.email.split('@')[1];
    if (tenant.authConfig.allowedDomains?.length) {
      if (!tenant.authConfig.allowedDomains.includes(domain)) {
        throw new Error('Email domain not allowed');
      }
    }
    
    // Create user
    user = await db.users.create({
      tenantId,
      email: profile.email,
      name: profile.name,
      externalId: profile.sub,
      status: 'active',
      role: 'member',
    });
  }
  
  // Update last login
  await db.users.update(user.id, { lastLoginAt: new Date() });
  
  return user;
}
```

## Tenant Onboarding

### Self-Serve Flow

```
1. Sign Up
   └── Create account (email/password or social)
   
2. Create Organization
   └── Tenant record created
   └── User set as admin
   
3. Choose Plan
   └── Select tier
   └── Enter payment (Stripe Checkout)
   
4. Configure Auth (optional)
   └── Connect SSO provider
   └── Or use email/password
   
5. Connect Data Source
   └── M365 OAuth flow
   └── Grant permissions
   
6. Initial Sync
   └── Background job starts
   └── Progress shown in UI
   
7. Ready
   └── Redirect to search/chat
```

### Tenant Creation

```typescript
async function createTenant(input: CreateTenantInput): Promise<Tenant> {
  return db.transaction(async (tx) => {
    // Create tenant
    const tenant = await tx.tenants.create({
      id: uuid(),
      slug: generateSlug(input.organizationName),
      name: input.organizationName,
      status: 'active',
      planId: 'starter',
      seatsIncluded: 5,
      seatsUsed: 1,
      settings: defaultSettings(),
      authConfig: {
        provider: 'email',  // Default, can change later
        autoProvision: false,
      },
    });
    
    // Create admin user
    await tx.users.create({
      id: uuid(),
      tenantId: tenant.id,
      email: input.adminEmail,
      name: input.adminName,
      status: 'active',
      role: 'admin',
    });
    
    // Create default resources
    await tx.apiKeys.create({
      tenantId: tenant.id,
      name: 'Default',
      key: generateApiKey(),
    });
    
    return tenant;
  });
}
```

## Cross-Tenant Operations

Some operations span tenants (admin/platform level):

| Operation | Who | Purpose |
|-----------|-----|---------|
| List all tenants | Platform admin | Management dashboard |
| Suspend tenant | Platform admin | Non-payment, abuse |
| Usage aggregation | System | Billing, analytics |
| Global search | Never | Not allowed |

```typescript
// Platform admin middleware
function requirePlatformAdmin(req: Request): void {
  if (!req.user?.isPlatformAdmin) {
    throw new ForbiddenError('Platform admin access required');
  }
}

// Tenant admin middleware  
function requireTenantAdmin(req: Request): void {
  if (req.user?.role !== 'admin') {
    throw new ForbiddenError('Tenant admin access required');
  }
}
```

## Tenant Deletion

GDPR and data hygiene require clean deletion:

```typescript
async function deleteTenant(tenantId: string): Promise<void> {
  // 1. Verify tenant can be deleted
  const tenant = await getTenant(tenantId);
  if (tenant.status === 'active') {
    throw new Error('Cannot delete active tenant');
  }
  
  // 2. Queue deletion job (heavy operation)
  await queue.add('delete_tenant', { tenantId }, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 60000 },
  });
}

// Deletion worker
async function processTenantDeletion(tenantId: string): Promise<void> {
  // Delete in order (respecting foreign keys)
  await db.documentChunks.deleteMany({ tenantId });
  await db.documents.deleteMany({ tenantId });
  await db.emails.deleteMany({ tenantId });
  await db.connectors.deleteMany({ tenantId });
  await db.auditLogs.deleteMany({ tenantId });
  await db.users.deleteMany({ tenantId });
  await db.tenant.delete({ id: tenantId });
  
  // Clear caches (scan + delete pattern - redis.del doesn't support wildcards)
  const keys = await redis.keys(`tenant:${tenantId}:*`);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
  
  // Log for compliance
  await platformAuditLog.create({
    action: 'tenant_deleted',
    tenantId,
    deletedAt: new Date(),
  });
}
```

## Scaling Considerations

### Current Design (MVP)
- Single database, shared tables
- tenant_id filtering on all queries
- Works well up to ~100 tenants, ~10M documents total

### Future Scaling Options

| Scale | Approach |
|-------|----------|
| 100+ tenants | Partition tables by tenant_id |
| Large single tenant | Dedicated schema or database |
| Geographic distribution | Regional deployments |
| Strict isolation | Dedicated infrastructure per tenant |

### Monitoring Per-Tenant

Track per-tenant metrics:

```typescript
interface TenantMetrics {
  tenantId: string;
  
  // Volume
  documentCount: number;
  emailCount: number;
  chunkCount: number;
  storageSizeBytes: number;
  
  // Usage
  searchesLast24h: number;
  chatMessagesLast24h: number;
  tokensUsedLast24h: number;
  
  // Health
  lastSyncAt: Date;
  syncErrorCount: number;
  avgQueryLatencyMs: number;
}
```
