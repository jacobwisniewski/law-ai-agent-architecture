# ACL and Security

## Overview

Access Control Lists (ACLs) determine who can see what. This is **critical** for law firms - showing a user documents they shouldn't access is a serious breach. This doc covers the ACL model, sync strategy, and query-time enforcement.

**Related:** See [Auth Architecture](./12-auth-architecture.md) for identity linking between platform users and external system identities.

## ACL Model

### Core Concepts

```typescript
// An ACL entry grants access to a resource
interface ACLEntry {
  resourceId: string;           // Document or email ID
  resourceType: 'document' | 'email';
  tenantId: string;
  
  // Who has access
  principalId: string;          // User ID or Group ID
  principalType: 'user' | 'group';
  
  // What access (simplified for read-only system)
  permission: 'read';           // We only need read for search/RAG
  
  // Source tracking
  sourceSystem: string;         // 'sharepoint', 'onedrive', 'outlook'
  sourcePermissionId?: string;  // Original permission ID for sync
  
  // Timestamps
  syncedAt: Date;
  expiresAt?: Date;             // For time-limited shares
}
```

### Principal Expansion

Groups must be expanded to individual users for efficient query filtering:

```
Group "Legal Team"
├── User: alice@firm.com
├── User: bob@firm.com
└── Nested Group: "Partners"
    ├── User: carol@firm.com
    └── User: dave@firm.com
```

```typescript
interface ExpandedACL {
  resourceId: string;
  tenantId: string;
  
  // Flattened list of all users who can access
  allowedUserIds: string[];
  
  // Original groups (for audit/debug)
  sourceGroups: string[];
  
  // Expansion metadata
  expandedAt: Date;
  expansionVersion: number;     // Increment on group membership change
}
```

## ACL Sync from M365

### SharePoint/OneDrive Permissions

```typescript
// Get permissions for a file
GET /drives/{driveId}/items/{itemId}/permissions

// Response
{
  "value": [
    {
      "id": "permission-id",
      "grantedTo": {
        "user": {
          "id": "user-id",
          "email": "alice@firm.com"
        }
      },
      "roles": ["read"]
    },
    {
      "id": "permission-id-2",
      "grantedToV2": {
        "group": {
          "id": "group-id",
          "displayName": "Legal Team"
        }
      },
      "roles": ["read", "write"]
    }
  ]
}
```

### Permission Inheritance

SharePoint has folder-level inheritance:

```
Site
└── Document Library
    └── Folder A (inherits from library)
        └── Document 1 (inherits from Folder A)
        └── Document 2 (unique permissions - inheritance broken)
    └── Folder B (unique permissions)
        └── Document 3 (inherits from Folder B)
```

**Sync strategy:**
1. Track inheritance chain for each document
2. On folder permission change, re-sync all children
3. Store `inheritedFrom` to optimize updates

### Group Membership Sync

```typescript
// Get group members
GET /groups/{groupId}/members

// Get nested groups
GET /groups/{groupId}/members?$filter=@odata.type eq 'microsoft.graph.group'

// Recursive expansion
async function expandGroup(groupId: string): Promise<string[]> {
  const members = await graph.getGroupMembers(groupId);
  const userIds: string[] = [];
  
  for (const member of members) {
    if (member.type === 'user') {
      userIds.push(member.id);
    } else if (member.type === 'group') {
      // Recursive expansion
      const nested = await expandGroup(member.id);
      userIds.push(...nested);
    }
  }
  
  return [...new Set(userIds)]; // Dedupe
}
```

## ACL Caching Strategy

### Cache Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     ACL CACHE LAYERS                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Redis Cache                       │   │
│  │                                                      │   │
│  │  Key: acl:user:{tenantId}:{userId}                  │   │
│  │  Value: Set of resourceIds user can access          │   │
│  │  TTL: 5 minutes                                      │   │
│  │                                                      │   │
│  │  Key: acl:resource:{tenantId}:{resourceId}          │   │
│  │  Value: Set of userIds who can access               │   │
│  │  TTL: 5 minutes                                      │   │
│  │                                                      │   │
│  │  Key: acl:group:{tenantId}:{groupId}                │   │
│  │  Value: Set of expanded userIds                      │   │
│  │  TTL: 15 minutes                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                    Cache Miss                               │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   PostgreSQL                         │   │
│  │                                                      │   │
│  │  acl_entries table                                   │   │
│  │  expanded_acls table                                 │   │
│  │  group_memberships table                            │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Cache Invalidation

Triggers for invalidation:

| Event | Invalidation |
|-------|--------------|
| Document permission change | Invalidate resource ACL, affected user ACLs |
| Group membership change | Invalidate group expansion, all member user ACLs |
| User added/removed from tenant | Invalidate user ACL |
| Webhook: permission changed | Targeted invalidation |
| Periodic sync | Full refresh for tenant |

```typescript
// Invalidation helper
async function invalidateACL(event: ACLChangeEvent): Promise<void> {
  const { tenantId, resourceId, affectedUserIds, affectedGroupIds } = event;
  
  const keysToDelete = [
    `acl:resource:${tenantId}:${resourceId}`,
    ...affectedUserIds.map(u => `acl:user:${tenantId}:${u}`),
    ...affectedGroupIds.map(g => `acl:group:${tenantId}:${g}`),
  ];
  
  await redis.del(...keysToDelete);
}
```

## Query-Time ACL Filtering

### Two Approaches

**Option A: Pre-filter**
```sql
-- Get user's allowed documents first, then search within
WITH user_docs AS (
  SELECT resource_id 
  FROM expanded_acls 
  WHERE tenant_id = $1 
    AND $2 = ANY(allowed_user_ids)
)
SELECT d.*, e.embedding <-> $3 AS distance
FROM document_chunks d
JOIN user_docs ud ON d.document_id = ud.resource_id
WHERE d.tenant_id = $1
ORDER BY distance
LIMIT 20;
```

**Option B: Post-filter**
```sql
-- Search first, filter after
SELECT d.*, e.embedding <-> $1 AS distance
FROM document_chunks d
JOIN expanded_acls a ON d.document_id = a.resource_id
WHERE d.tenant_id = $2
  AND $3 = ANY(a.allowed_user_ids)
ORDER BY distance
LIMIT 20;
```

### Performance Considerations

| Approach | Pros | Cons |
|----------|------|------|
| Pre-filter | Smaller search space | Two-step query, may miss relevant if user has few docs |
| Post-filter | Single query | May scan many unauthorized docs |
| Hybrid | Best of both | More complex |

**Recommendation:** Start with post-filter (simpler), add pre-filter optimization if performance issues arise with users who have very limited access.

### Index Strategy

```sql
-- Essential indexes for ACL filtering
CREATE INDEX idx_expanded_acls_tenant_users 
  ON expanded_acls USING GIN (allowed_user_ids) 
  WHERE tenant_id = ?;  -- Partial index per tenant

CREATE INDEX idx_document_chunks_tenant_doc 
  ON document_chunks (tenant_id, document_id);

-- For vector search with ACL
CREATE INDEX idx_chunks_embedding 
  ON document_chunks USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
```

## Security Audit Trail

### Audit Log Schema

```typescript
interface AuditLogEntry {
  id: string;
  tenantId: string;
  timestamp: Date;
  
  // Actor
  userId: string;
  userEmail: string;
  ipAddress: string;
  userAgent: string;
  
  // Action
  action: AuditAction;
  resourceType: 'document' | 'email' | 'search' | 'chat';
  resourceId?: string;
  
  // Details
  query?: string;              // For search/chat actions
  resultsCount?: number;
  resourcesAccessed?: string[]; // Document IDs included in response
  
  // ACL context
  aclDecision: 'allowed' | 'denied';
  aclReason?: string;
}

type AuditAction = 
  | 'search'
  | 'chat'
  | 'document_view'
  | 'document_download'
  | 'acl_check'
  | 'login'
  | 'logout';
```

### What to Log

| Event | Log? | Details |
|-------|------|---------|
| Every search query | Yes | Query, results count, docs accessed |
| Every chat message | Yes | Message, sources cited |
| Document view | Yes | Document ID, access method |
| ACL denial | Yes | What was denied, why |
| Login/logout | Yes | IP, device, success/failure |
| ACL sync | Yes | Counts, errors |
| Admin actions | Yes | What changed, by whom |

### Retention

- **Default:** 90 days hot storage, 2 years cold (S3/Glacier)
- **Configurable per tenant:** Some firms require longer retention
- **Export capability:** Firms may need to produce audit logs for compliance

## Security Best Practices

### Defense in Depth

1. **Tenant isolation:** All queries include `tenant_id` - enforced at ORM/query layer
2. **User isolation:** ACL check on every data access - no exceptions
3. **API authentication:** JWT validation, token expiry, refresh rotation
4. **Rate limiting:** Prevent enumeration attacks
5. **Input validation:** Prevent injection in search queries
6. **Encryption:** TLS in transit, AES-256 at rest

### Common Attack Vectors

| Attack | Mitigation |
|--------|------------|
| Cross-tenant data access | `tenant_id` filter in every query, ORM hooks |
| Privilege escalation | ACL check at service layer, not just API |
| Token theft | Short expiry, refresh tokens, token binding |
| Search result enumeration | Rate limiting, ACL filtering |
| Prompt injection | Input sanitization, output validation |
| Data exfiltration via chat | Rate limits, anomaly detection |

### Compliance Considerations

- **GDPR:** Right to deletion - must be able to purge user data
- **Client confidentiality:** Ethical walls between matters (future feature)
- **Legal hold:** Ability to preserve data during litigation
- **SOC 2:** Audit logging, access controls, encryption
