# Onboarding Flow

## Overview

Self-serve onboarding to minimize friction. Goal: firm can go from sign-up to first search in under 10 minutes.

## Onboarding Steps

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ONBOARDING FLOW                                      │
│                                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│  │ Sign Up │───▶│ Create  │───▶│  Plan   │───▶│ Connect │───▶│  Sync   │   │
│  │         │    │   Org   │    │ Select  │    │  Data   │    │         │   │
│  │         │    │         │    │         │    │         │    │         │   │
│  │ Email   │    │ Name    │    │ Choose  │    │ M365    │    │ Wait/   │   │
│  │ or SSO  │    │ Slug    │    │ tier    │    │ OAuth   │    │ Tour    │   │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Step 1: Sign Up

### Options

1. **Email + Password**
   - Quick start, no dependencies
   - Email verification required

2. **Google Sign-In**
   - One-click for Google Workspace firms
   - Auto-fills name/email

3. **Microsoft Sign-In**
   - One-click for M365 firms
   - Natural path to M365 connector

### Implementation

```typescript
// Sign-up page options
interface SignUpOptions {
  email: boolean;
  google: boolean;
  microsoft: boolean;
}

// After successful auth
async function handleSignUp(profile: AuthProfile): Promise<void> {
  // Check if user already exists
  const existingUser = await db.users.findByEmail(profile.email);
  
  if (existingUser) {
    // Existing user - redirect to their tenant
    return redirect(`/app/${existingUser.tenant.slug}`);
  }
  
  // New user - start onboarding
  return redirect('/onboarding/create-org');
}
```

## Step 2: Create Organization

### Form Fields

```typescript
interface CreateOrgForm {
  organizationName: string;     // Required
  slug?: string;                // Auto-generated, editable
  
  // Optional (can set later)
  website?: string;
  size?: '1-10' | '11-50' | '51-200' | '200+';
}
```

### Validation

```typescript
async function validateOrg(input: CreateOrgForm): Promise<ValidationResult> {
  const errors: string[] = [];
  
  // Name
  if (!input.organizationName || input.organizationName.length < 2) {
    errors.push('Organization name is required');
  }
  
  // Slug uniqueness
  const slug = input.slug || generateSlug(input.organizationName);
  const existing = await db.tenants.findBySlug(slug);
  if (existing) {
    errors.push('This URL is already taken');
  }
  
  return { valid: errors.length === 0, errors };
}
```

## Step 3: Plan Selection

### UI

```
┌─────────────────────────────────────────────────────────────┐
│                    Choose Your Plan                          │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │    Starter      │  │  Professional   │                  │
│  │                 │  │                 │                  │
│  │   $49/seat/mo   │  │   $39/seat/mo   │                  │
│  │                 │  │                 │                  │
│  │ • Up to 10 seats│  │ • 11-50 seats   │                  │
│  │ • 5,000 credits │  │ • 10,000 credits│                  │
│  │ • Email support │  │ • Priority supp │                  │
│  │                 │  │ • SSO included  │                  │
│  │  [Select]       │  │  [Select]       │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                             │
│  Need more? Contact us for Enterprise pricing               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Payment Flow

```typescript
async function handlePlanSelection(
  tenantId: string,
  planId: string,
  seats: number
): Promise<string> {
  const plan = getPlan(planId);
  
  // Create Stripe Checkout session
  const session = await stripe.checkout.sessions.create({
    customer_email: await getTenantAdminEmail(tenantId),
    mode: 'subscription',
    line_items: [{
      price: plan.stripePriceId,
      quantity: seats,
    }],
    success_url: `${APP_URL}/onboarding/connect?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${APP_URL}/onboarding/plan`,
    metadata: {
      tenantId,
      planId,
    },
  });
  
  return session.url;
}
```

## Step 4: Connect Data Source

### M365 Connection Flow

```
┌─────────────────────────────────────────────────────────────┐
│                Connect Microsoft 365                         │
│                                                             │
│  We'll connect to your SharePoint, OneDrive, and Outlook    │
│  to index your firm's documents and emails.                 │
│                                                             │
│  What we access:                                            │
│  ✓ Read documents in SharePoint and OneDrive                │
│  ✓ Read emails (with your permission settings respected)    │
│  ✓ Read user and group info for permissions                 │
│                                                             │
│  What we DON'T do:                                          │
│  ✗ Modify or delete any documents                           │
│  ✗ Send emails on your behalf                               │
│  ✗ Access personal OneDrive (unless configured)             │
│                                                             │
│  [Connect Microsoft 365]                                    │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│  Skip for now (you can connect later in Settings)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### OAuth Flow

```typescript
// Initiate OAuth
async function initiateM365Auth(tenantId: string): Promise<string> {
  const state = jwt.sign({ tenantId }, SECRET, { expiresIn: '10m' });
  
  const authUrl = new URL('https://login.microsoftonline.com/common/oauth2/v2.0/authorize');
  authUrl.searchParams.set('client_id', M365_CLIENT_ID);
  authUrl.searchParams.set('response_type', 'code');
  authUrl.searchParams.set('redirect_uri', `${APP_URL}/api/connectors/m365/callback`);
  authUrl.searchParams.set('scope', M365_SCOPES.join(' '));
  authUrl.searchParams.set('state', state);
  authUrl.searchParams.set('prompt', 'consent');  // Force consent screen
  
  return authUrl.toString();
}

// Handle callback
async function handleM365Callback(code: string, state: string): Promise<void> {
  // Verify state
  const { tenantId } = jwt.verify(state, SECRET);
  
  // Exchange code for tokens
  const tokens = await exchangeCodeForTokens(code);
  
  // Store connector config
  await db.connectors.create({
    tenantId,
    type: 'm365',
    credentials: encrypt(tokens),
    status: 'active',
  });
  
  // Trigger initial sync
  await queue.add('sync_tenant', {
    tenantId,
    connectorId: 'm365',
    full: true,
  });
}
```

## Step 5: Initial Sync

### Progress UI

```
┌─────────────────────────────────────────────────────────────┐
│                Setting Up Your Workspace                     │
│                                                             │
│  We're indexing your documents and emails. This usually     │
│  takes 5-15 minutes depending on your data volume.          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ████████████████████░░░░░░░░░░░░  45%               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  📄 Documents: 1,234 / 2,500                                │
│  📧 Emails: 5,678 / 15,000                                  │
│  ⏱️  Estimated time remaining: 8 minutes                    │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  While you wait, take a quick tour:                         │
│  [Watch 2-min demo video]                                   │
│                                                             │
│  Or we'll email you when it's ready:                        │
│  [Notify me by email]                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Sync Progress Tracking

```typescript
interface SyncProgress {
  tenantId: string;
  connectorId: string;
  
  status: 'pending' | 'in_progress' | 'completed' | 'failed';
  startedAt: Date;
  completedAt?: Date;
  
  // Counts
  documentsTotal: number;
  documentsProcessed: number;
  emailsTotal: number;
  emailsProcessed: number;
  
  // Errors
  errorCount: number;
  lastError?: string;
}

// SSE endpoint for real-time progress
async function* streamSyncProgress(tenantId: string): AsyncGenerator<SyncProgress> {
  while (true) {
    const progress = await redis.get(`sync:${tenantId}:progress`);
    yield JSON.parse(progress);
    
    if (progress.status === 'completed' || progress.status === 'failed') {
      break;
    }
    
    await sleep(2000);  // Poll every 2 seconds
  }
}
```

### Completion Notification

```typescript
async function handleSyncComplete(tenantId: string): Promise<void> {
  const progress = await getSyncProgress(tenantId);
  const admin = await getTenantAdmin(tenantId);
  
  // Send email
  await sendEmail({
    to: admin.email,
    template: 'sync_complete',
    data: {
      documentsIndexed: progress.documentsProcessed,
      emailsIndexed: progress.emailsProcessed,
      appUrl: `${APP_URL}/app/${tenant.slug}`,
    },
  });
  
  // In-app notification
  await createNotification({
    tenantId,
    userId: admin.id,
    type: 'sync_complete',
    title: 'Your workspace is ready!',
    message: `We've indexed ${progress.documentsProcessed} documents and ${progress.emailsProcessed} emails.`,
  });
}
```

## Step 6: First-Time User Experience

### Welcome Tour

```typescript
interface TourStep {
  target: string;           // CSS selector
  title: string;
  content: string;
  position: 'top' | 'bottom' | 'left' | 'right';
}

const welcomeTour: TourStep[] = [
  {
    target: '#search-bar',
    title: 'Search Everything',
    content: 'Type a question or search for documents. We\'ll search across all your connected sources.',
    position: 'bottom',
  },
  {
    target: '#chat-panel',
    title: 'AI-Powered Answers',
    content: 'Ask questions in natural language. We\'ll find relevant documents and summarize the answer with citations.',
    position: 'left',
  },
  {
    target: '#sources-panel',
    title: 'Click to Open Source',
    content: 'Every answer includes citations. Click to open the original document in SharePoint or Outlook.',
    position: 'left',
  },
  {
    target: '#settings-menu',
    title: 'Invite Your Team',
    content: 'Add team members from Settings. They\'ll only see documents they have access to.',
    position: 'bottom',
  },
];
```

### Sample Queries

Pre-populate suggestions for first-time users:

```typescript
const sampleQueries = [
  "What are the key terms in our most recent client contract?",
  "Find all documents related to [matter name]",
  "What did [client name] say about the deadline?",
  "Show me recent emails about the merger",
];
```

## Onboarding State Machine

```typescript
type OnboardingState =
  | 'signed_up'
  | 'org_created'
  | 'plan_selected'
  | 'connector_linked'
  | 'sync_in_progress'
  | 'sync_complete'
  | 'tour_complete'
  | 'onboarding_complete';

async function getOnboardingState(tenantId: string): Promise<OnboardingState> {
  const tenant = await getTenant(tenantId);
  
  if (!tenant) return 'signed_up';
  if (!tenant.stripeSubscriptionId) return 'org_created';
  if (!tenant.connectors.length) return 'plan_selected';
  
  const sync = await getLatestSync(tenantId);
  if (!sync) return 'connector_linked';
  if (sync.status === 'in_progress') return 'sync_in_progress';
  if (sync.status !== 'completed') return 'connector_linked';
  
  if (!tenant.tourCompleted) return 'sync_complete';
  
  return 'onboarding_complete';
}

// Redirect based on state
async function getOnboardingRedirect(tenantId: string): Promise<string> {
  const state = await getOnboardingState(tenantId);
  
  const redirects: Record<OnboardingState, string> = {
    'signed_up': '/onboarding/create-org',
    'org_created': '/onboarding/plan',
    'plan_selected': '/onboarding/connect',
    'connector_linked': '/onboarding/connect',
    'sync_in_progress': '/onboarding/sync',
    'sync_complete': '/app?tour=true',
    'tour_complete': '/app',
    'onboarding_complete': '/app',
  };
  
  return redirects[state];
}
```

## Error Handling

### Common Issues

| Issue | Detection | Resolution |
|-------|-----------|------------|
| M365 auth fails | OAuth error | Show retry button, help link |
| Insufficient permissions | Graph API 403 | Guide to grant admin consent |
| Sync stalls | No progress for 10 min | Alert user, offer support chat |
| Payment fails | Stripe webhook | Show update payment form |

### Recovery UI

```typescript
interface OnboardingError {
  step: OnboardingState;
  errorCode: string;
  message: string;
  actions: {
    label: string;
    action: 'retry' | 'skip' | 'contact_support' | 'url';
    url?: string;
  }[];
}
```
