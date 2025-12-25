# Authentication & Authorization Architecture

## Overview

Three separate auth concerns. Don't conflate them.

| Layer | Question | Who | When | What's Stored |
|-------|----------|-----|------|---------------|
| **Platform Login** | "Who is this person?" | Every user | Every session | Session cookie |
| **Connector Auth** | "Can we access their data?" | Tenant admin | Once per connector | OAuth tokens |
| **Identity Linking** | "Their user = our user?" | Automatic + admin | Ongoing | User ID mappings |

### How They Interact

```
TENANT ADMIN (once per connector):
  "Connect M365" → OAuth consent → tokens stored

USER (every login):
  "Login" → session created
  
BACKGROUND (ongoing):
  use tokens → sync documents → extract permissions → map to platform users

USER SEARCH:
  query → find docs → filter by "can this user see it?" → return results
```

### Common Questions

**Q: User logs in with Google but firm uses M365 for documents?**
A: Fine. Login is separate from data access. Documents come from M365 via connector tokens.

**Q: User has different email in M365 vs platform?**
A: Admin manually links them. Or user logs in once with Microsoft to auto-capture the link.

**Q: Connector token expires?**
A: Background job refreshes it. If refresh fails, email tenant admin to re-authorize.

**Q: Firm uses M365 AND NetDocuments?**
A: Two connectors, two sets of tokens. Documents merged at search time. User has identity links for both (usually same email, auto-matched).

### Adding a New Connector

Only need to implement:
1. **OAuth flow** - Different URL/scopes, same pattern
2. **Document sync** - Different API, same output (documents + permissions)  
3. **Identity mapping** - Usually email matching works

Platform login and ACL filtering don't change.

---

## 1. Platform Authentication

### What It Is

User proving they're allowed to use your app. Has **nothing** to do with accessing their documents.

### Auth Methods

| Method | MVP | Use Case |
|--------|-----|----------|
| Email/Password | Yes | Quick start, small firms |
| Microsoft OAuth | Yes | Natural for M365 customers |
| Google OAuth | Yes | Google Workspace customers |
| SAML/OIDC SSO | Post-MVP | Enterprise with existing IdP |

### Platform vs Connector OAuth Apps

**Critical distinction:** We need separate OAuth apps for platform auth vs connector access:

| Purpose | OAuth App | Permissions |
|---------|-----------|-------------|
| Platform login | Platform App | `openid`, `profile`, `email` only |
| M365 data access | Connector App | `Files.Read.All`, `Mail.Read`, etc. |

Why separate?
- **Principle of least privilege** - Login doesn't need data access
- **User clarity** - Consent screens show appropriate permissions
- **Security** - Compromised login token can't access data

### User Model

```typescript
interface PlatformUser {
  id: string;                    // Platform UUID
  tenantId: string;              // Organization they belong to
  
  // Core identity
  email: string;
  name: string;
  
  // Platform auth
  passwordHash?: string;         // If using email/password
  emailVerified: boolean;
  
  // Social auth links (for platform login)
  microsoftId?: string;          // Microsoft account ID
  googleId?: string;             // Google account ID
  
  // SSO (post-MVP)
  ssoProviderId?: string;        // External IdP user ID
  
  // Status
  status: 'invited' | 'active' | 'suspended';
  role: 'admin' | 'member';
  
  // External identity links (for ACL resolution)
  externalIdentities: ExternalIdentity[];
  
  createdAt: Date;
  lastLoginAt?: Date;
}
```

## 2. Connector Authorization

### Overview

Connector auth is about authorizing our **platform** to access a **tenant's data** in external systems. This is typically done once by a tenant admin.

### Permission Models by Connector

| Connector | Auth Type | Who Consents | Permissions Granted |
|-----------|-----------|--------------|---------------------|
| M365 | OAuth 2.0 | Tenant Admin | Application permissions (app-only) |
| NetDocuments | OAuth 2.0 | Tenant Admin | Repository access |
| iManage | OAuth 2.0 / API Key | Tenant Admin | Library access |
| Google Workspace | OAuth 2.0 | Domain Admin | Domain-wide delegation |

### M365 Connector Auth Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    M365 CONNECTOR AUTH FLOW                                  │
│                                                                             │
│  ┌──────────────┐                                                           │
│  │ Tenant Admin │                                                           │
│  │ clicks       │                                                           │
│  │ "Connect M365"                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │  Platform    │────▶│   Azure AD   │────▶│   Admin      │                │
│  │  redirects   │     │   /authorize │     │   Consent    │                │
│  │  to Azure    │     │              │     │   Screen     │                │
│  └──────────────┘     └──────────────┘     └──────┬───────┘                │
│                                                    │                        │
│                            Admin approves          │                        │
│                                                    ▼                        │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │  Store       │◀────│   Exchange   │◀────│   Callback   │                │
│  │  tokens      │     │   code for   │     │   with code  │                │
│  │  (encrypted) │     │   tokens     │     │              │                │
│  └──────────────┘     └──────────────┘     └──────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Application vs Delegated Permissions

**Application permissions (recommended for connectors):**
- App acts as itself, not as a user
- Requires admin consent once
- Can access all users' data (scoped by admin)
- No per-user token management

**Delegated permissions:**
- App acts on behalf of a user
- Each user must consent
- Only accesses what user can access
- Requires per-user token storage

**Our approach:** Use application permissions for data sync (simpler, more reliable), use delegated for user-specific actions if needed.

### Connector Credentials Storage

```typescript
interface ConnectorCredentials {
  id: string;
  tenantId: string;
  connectorType: 'm365' | 'netdocuments' | 'imanage' | 'google';
  
  // OAuth tokens (encrypted at rest)
  accessToken: string;
  refreshToken: string;
  tokenExpiresAt: Date;
  
  // Connector-specific config
  config: ConnectorConfig;
  
  // Status
  status: 'active' | 'expired' | 'revoked' | 'error';
  lastUsedAt: Date;
  lastRefreshAt: Date;
  lastError?: string;
  
  // Audit
  authorizedBy: string;          // User ID who connected
  authorizedAt: Date;
}

// M365-specific config
interface M365ConnectorConfig {
  azureTenantId: string;         // Customer's Azure tenant ID
  driveIds: string[];            // SharePoint/OneDrive drives to sync
  includeEmail: boolean;
  emailFolders?: string[];       // Specific folders, or all
}

// NetDocuments-specific config
interface NetDocumentsConnectorConfig {
  repositoryId: string;
  cabinetIds: string[];
}
```

### Token Management

```
GET VALID TOKEN:
  if no credentials → error "not configured"
  if status = revoked → error "access revoked"
  if token expiring soon → refresh it
  return token

REFRESH TOKEN:
  try:
    exchange refresh token for new tokens
    store new tokens (encrypted)
    return new access token
  catch:
    mark connector as expired
    notify tenant admin
    throw error
```

### Admin Consent URL Generation

```typescript
function getM365AdminConsentUrl(tenantId: string): string {
  const state = jwt.sign({ tenantId, type: 'connector' }, SECRET, { expiresIn: '10m' });
  
  const params = new URLSearchParams({
    client_id: process.env.MS_CONNECTOR_CLIENT_ID!,
    redirect_uri: `${APP_URL}/api/connectors/m365/callback`,
    response_type: 'code',
    response_mode: 'query',
    scope: [
      'https://graph.microsoft.com/.default',  // Request all configured app permissions
    ].join(' '),
    state,
    prompt: 'admin_consent',  // Force admin consent flow
  });
  
  // Use /common for multi-tenant, or specific tenant ID
  return `https://login.microsoftonline.com/common/oauth2/v2.0/authorize?${params}`;
}
```

## 3. Identity Linking

### The Problem

When we sync a document from M365 with permissions `[user-abc-123, group-xyz-456]`, we need to know which **platform users** those M365 IDs correspond to.

### Identity Mapping Model

```typescript
interface ExternalIdentity {
  id: string;
  userId: string;                // Platform user ID
  tenantId: string;
  
  // External identity
  provider: 'm365' | 'netdocuments' | 'imanage' | 'google';
  externalId: string;            // ID in external system
  externalEmail: string;         // Email in external system
  
  // Linking metadata
  linkMethod: 'email_match' | 'oauth' | 'admin_manual' | 'scim';
  linkedAt: Date;
  linkedBy?: string;             // Admin who linked, if manual
  
  // Confidence
  verified: boolean;             // Has user verified this link?
}
```

### Linking Methods

#### Method 1: Email Matching (Default)

Most common and simplest - match by email address.

```
FOR each external user:
  find platform user with same email
  IF found:
    create/update identity link
  ELSE:
    log as unlinked (no platform account)
```

#### Method 2: OAuth Linking

When user logs in with same provider, capture the external ID.

```
ON OAuth login (e.g., "Login with Microsoft"):
  create identity link:
    platform user → external ID from OAuth profile
  mark as verified (OAuth is proof)
```

#### Method 3: Admin Manual Mapping

For cases where emails don't match (contractor accounts, etc.).

```
ADMIN links identity:
  verify admin has permission
  create identity link
  mark as unverified (until user confirms)
  audit log the action
```

#### Method 4: SCIM Provisioning (Enterprise)

For enterprise customers with identity management systems.

```
ON SCIM user create:
  create platform user
  link external identities from SCIM attributes
  mark as verified
```

### ACL Resolution Flow

When processing document permissions, resolve external IDs to platform users:

```
FOR each external principal (user or group):
  IF user:
    look up by external ID in identity links
    IF not found, try email match (and auto-create link)
    IF still not found, skip (no platform account)
  IF group:
    get expanded group members (already resolved to platform user IDs)
    add all members

RETURN list of platform user IDs
```

### Group Expansion and Caching

```typescript
interface ExternalGroup {
  id: string;
  tenantId: string;
  provider: string;
  externalGroupId: string;
  displayName: string;
  
  // Expanded members (platform user IDs)
  memberUserIds: string[];
  
  // Sync metadata
  lastSyncedAt: Date;
  memberCount: number;
}

async function syncExternalGroup(
  tenantId: string,
  provider: string,
  groupId: string
): Promise<void> {
  // Fetch group members from external system
  const connector = getConnector(provider);
  const externalMembers = await connector.getGroupMembers(groupId);
  
  // Resolve each member to platform user
  const platformUserIds: string[] = [];
  
  for (const member of externalMembers) {
    if (member.type === 'user') {
      const userId = await resolveExternalUserToPlatform(tenantId, provider, member.id);
      if (userId) platformUserIds.push(userId);
    } else if (member.type === 'group') {
      // Recursive expansion
      await syncExternalGroup(tenantId, provider, member.id);
      const nestedGroup = await db.externalGroups.findFirst({
        where: { tenantId, provider, externalGroupId: member.id },
      });
      if (nestedGroup) {
        platformUserIds.push(...nestedGroup.memberUserIds);
      }
    }
  }
  
  // Store expanded group
  await db.externalGroups.upsert({
    where: { tenantId_provider_externalGroupId: { tenantId, provider, externalGroupId: groupId } },
    create: {
      tenantId,
      provider,
      externalGroupId: groupId,
      displayName: await connector.getGroupDisplayName(groupId),
      memberUserIds: [...new Set(platformUserIds)],
      lastSyncedAt: new Date(),
      memberCount: platformUserIds.length,
    },
    update: {
      memberUserIds: [...new Set(platformUserIds)],
      lastSyncedAt: new Date(),
      memberCount: platformUserIds.length,
    },
  });
  
  // Invalidate ACL cache
  await invalidateGroupACLCache(tenantId, provider, groupId);
}
```

## Token Security

### Encryption at Rest

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.TOKEN_ENCRYPTION_KEY!, 'hex'); // 32 bytes

function encrypt(plaintext: string): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv(ALGORITHM, KEY, iv);
  
  let encrypted = cipher.update(plaintext, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  // Format: iv:authTag:ciphertext
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
}

function decrypt(encrypted: string): string {
  const [ivHex, authTagHex, ciphertext] = encrypted.split(':');
  
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  
  const decipher = createDecipheriv(ALGORITHM, KEY, iv);
  decipher.setAuthTag(authTag);
  
  let decrypted = decipher.update(ciphertext, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}
```

### Token Rotation

```typescript
// Scheduled job: rotate tokens proactively
async function rotateExpiringTokens(): Promise<void> {
  const ROTATION_THRESHOLD_HOURS = 24;
  
  const expiringCreds = await db.connectorCredentials.findMany({
    where: {
      status: 'active',
      tokenExpiresAt: {
        lt: new Date(Date.now() + ROTATION_THRESHOLD_HOURS * 60 * 60 * 1000),
      },
    },
  });
  
  for (const creds of expiringCreds) {
    try {
      await tokenManager.refreshToken(creds);
      logger.info('Token rotated', { tenantId: creds.tenantId, connector: creds.connectorType });
    } catch (error) {
      logger.error('Token rotation failed', { tenantId: creds.tenantId, error });
    }
  }
}
```

## Error Handling & Recovery

### Connector Auth Errors

| Error | Cause | Recovery |
|-------|-------|----------|
| `AUTH_EXPIRED` | Token expired, refresh failed | Prompt re-auth |
| `AUTH_REVOKED` | Admin revoked app access | Prompt re-auth |
| `CONSENT_REQUIRED` | New permissions needed | Prompt admin consent |
| `TENANT_MISMATCH` | Wrong Azure tenant | Guide admin to correct tenant |

### Recovery Flow

```typescript
async function handleConnectorAuthError(
  tenantId: string,
  connectorType: string,
  error: ConnectorError
): Promise<void> {
  // Update connector status
  await db.connectorCredentials.update({
    where: { tenantId_connectorType: { tenantId, connectorType } },
    data: {
      status: error.code === 'AUTH_REVOKED' ? 'revoked' : 'expired',
      lastError: error.message,
    },
  });
  
  // Pause sync jobs
  await pauseSyncJobs(tenantId, connectorType);
  
  // Notify tenant admin
  const admins = await getTenantAdmins(tenantId);
  for (const admin of admins) {
    await sendEmail({
      to: admin.email,
      template: 'connector_auth_error',
      data: {
        connectorType,
        errorMessage: error.message,
        reconnectUrl: `${APP_URL}/settings/connectors/${connectorType}/reconnect`,
      },
    });
    
    await createNotification({
      userId: admin.id,
      type: 'connector_error',
      title: `${connectorType} connection lost`,
      message: 'Please reconnect to continue syncing.',
      actionUrl: `/settings/connectors/${connectorType}`,
    });
  }
}
```

## SSO Configuration (Post-MVP)

### Tenant SSO Setup

```typescript
interface TenantSSOConfig {
  tenantId: string;
  
  // Provider
  provider: 'okta' | 'azure_ad' | 'onelogin' | 'custom_oidc';
  
  // OIDC/SAML settings
  issuerUrl: string;             // e.g., https://firm.okta.com
  clientId: string;
  clientSecret: string;          // Encrypted
  
  // SAML-specific
  samlCertificate?: string;
  samlEntityId?: string;
  
  // Behavior
  allowPasswordLogin: boolean;   // Allow email/password alongside SSO
  autoProvision: boolean;        // Create users on first SSO login
  allowedDomains: string[];      // Restrict to specific email domains
  
  // Attribute mapping
  attributeMapping: {
    email: string;               // OIDC claim or SAML attribute
    name: string;
    groups?: string;             // For role assignment
  };
}
```

### SSO Login Flow

```typescript
async function initiateSSOLogin(tenantSlug: string): Promise<string> {
  const tenant = await getTenantBySlug(tenantSlug);
  
  if (!tenant.ssoConfig) {
    throw new Error('SSO not configured for this organization');
  }
  
  const state = jwt.sign(
    { tenantId: tenant.id, type: 'sso_login' },
    SECRET,
    { expiresIn: '10m' }
  );
  
  const nonce = randomBytes(16).toString('hex');
  await redis.setex(`sso:nonce:${nonce}`, 600, tenant.id);
  
  const params = new URLSearchParams({
    client_id: tenant.ssoConfig.clientId,
    redirect_uri: `${APP_URL}/api/auth/sso/callback`,
    response_type: 'code',
    scope: 'openid profile email',
    state,
    nonce,
  });
  
  return `${tenant.ssoConfig.issuerUrl}/oauth2/v1/authorize?${params}`;
}
```

## Database Schema

```sql
-- External identity links
CREATE TABLE external_identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  
  provider VARCHAR(50) NOT NULL,
  external_id VARCHAR(255) NOT NULL,
  external_email VARCHAR(255),
  
  link_method VARCHAR(50) NOT NULL,
  linked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  linked_by UUID REFERENCES users(id),
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  
  UNIQUE(user_id, provider),
  UNIQUE(tenant_id, provider, external_id)
);

CREATE INDEX idx_external_identities_lookup 
  ON external_identities(tenant_id, provider, external_id);

-- Connector credentials
CREATE TABLE connector_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  connector_type VARCHAR(50) NOT NULL,
  
  access_token TEXT NOT NULL,      -- Encrypted
  refresh_token TEXT NOT NULL,     -- Encrypted
  token_expires_at TIMESTAMPTZ NOT NULL,
  
  config JSONB NOT NULL DEFAULT '{}',
  
  status VARCHAR(50) NOT NULL DEFAULT 'active',
  last_used_at TIMESTAMPTZ,
  last_refresh_at TIMESTAMPTZ,
  last_error TEXT,
  
  authorized_by UUID NOT NULL REFERENCES users(id),
  authorized_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  UNIQUE(tenant_id, connector_type)
);

-- External groups (for ACL expansion)
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

-- Tenant SSO config (post-MVP)
CREATE TABLE tenant_sso_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE UNIQUE,
  
  provider VARCHAR(50) NOT NULL,
  issuer_url VARCHAR(500) NOT NULL,
  client_id VARCHAR(255) NOT NULL,
  client_secret TEXT NOT NULL,     -- Encrypted
  
  saml_certificate TEXT,
  saml_entity_id VARCHAR(500),
  
  allow_password_login BOOLEAN NOT NULL DEFAULT TRUE,
  auto_provision BOOLEAN NOT NULL DEFAULT FALSE,
  allowed_domains TEXT[] NOT NULL DEFAULT '{}',
  
  attribute_mapping JSONB NOT NULL DEFAULT '{}',
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Summary

| Concern | Solution | When |
|---------|----------|------|
| Platform login | Better Auth (email/password, social) | MVP |
| Platform SSO | Better Auth + tenant SSO config | Post-MVP |
| Connector auth | Per-connector OAuth with admin consent | MVP |
| Identity linking | Email match (auto) + OAuth + manual | MVP |
| Token storage | AES-256-GCM encrypted in PostgreSQL | MVP |
| Token refresh | Background job + on-demand | MVP |
| Group expansion | Sync job + cache in PostgreSQL | MVP |
