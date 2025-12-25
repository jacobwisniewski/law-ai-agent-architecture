# Billing and Pricing

## Overview

Per-seat pricing with usage credits. Simple to understand, scales with firm size and usage.

## Pricing Model

### Plans

| Plan | Monthly/Seat | Seats | Credits/Month |
|------|-------------|-------|---------------|
| Starter | $49 | 1-10 | 5,000 |
| Professional | $39 | 11-50 | 10,000 |
| Enterprise | Custom | 50+ | Custom |

**No free trial** - users pay from day one.

**Hard limits** - when credits or seats are exhausted, usage is blocked until next billing cycle or upgrade. No overage billing.

### Credit Costs

| Action | Credits | Notes |
|--------|---------|-------|
| Search query | 1 | Keyword or hybrid |
| Chat message | 5 | Includes RAG retrieval |
| Document ingested | 2 | Per document (any size) |
| Email ingested | 1 | Per email |
| Re-sync (no change) | 0 | Delta sync is free |

### What's Not Metered (Included)

- Storage (within reasonable limits)
- Users viewing documents
- Admin operations
- API calls for sync status

## Data Model

```typescript
interface Subscription {
  id: string;
  tenantId: string;
  
  // Stripe references
  stripeCustomerId: string;
  stripeSubscriptionId: string;
  
  // Plan details
  planId: string;
  status: 'active' | 'past_due' | 'cancelled';
  
  // Billing cycle
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  
  // Seats
  seatsQuantity: number;
  seatsPriceId: string;         // Stripe price ID
  
  // Credits
  creditsIncluded: number;      // Per period
  creditsUsed: number;          // This period
}

interface UsageEvent {
  id: string;
  tenantId: string;
  userId: string;
  
  // What happened
  eventType: 'search' | 'chat' | 'document_ingest' | 'email_ingest';
  credits: number;
  
  // Context
  timestamp: Date;
  metadata: Record<string, unknown>;
}

interface Invoice {
  id: string;
  tenantId: string;
  stripeInvoiceId: string;
  
  // Period
  periodStart: Date;
  periodEnd: Date;
  
  // Line items
  seatsAmount: number;
  totalAmount: number;
  
  // Status
  status: 'draft' | 'open' | 'paid' | 'void';
  paidAt?: Date;
}
```

## Stripe Integration

### Setup Flow

```typescript
// 1. Create Stripe customer on tenant creation
async function createStripeCustomer(tenant: Tenant, admin: User): Promise<string> {
  const customer = await stripe.customers.create({
    email: admin.email,
    name: tenant.name,
    metadata: {
      tenantId: tenant.id,
    },
  });
  
  return customer.id;
}

// 2. Create subscription on plan selection
async function createSubscription(
  tenantId: string,
  planId: string,
  seats: number
): Promise<Subscription> {
  const tenant = await getTenant(tenantId);
  const plan = getPlan(planId);
  
  const subscription = await stripe.subscriptions.create({
    customer: tenant.stripeCustomerId,
    items: [
      {
        price: plan.stripeSeatPriceId,
        quantity: seats,
      },
    ],
    metadata: {
      tenantId,
    },
  });
  
  return db.subscriptions.create({
    tenantId,
    stripeCustomerId: tenant.stripeCustomerId,
    stripeSubscriptionId: subscription.id,
    planId,
    status: 'active',
    seatsQuantity: seats,
    creditsIncluded: plan.creditsPerSeat * seats,
    creditsUsed: 0,
  });
}
```

### Webhook Handling

```typescript
// Stripe webhook endpoint
async function handleStripeWebhook(event: Stripe.Event): Promise<void> {
  switch (event.type) {
    case 'invoice.paid':
      await handleInvoicePaid(event.data.object);
      break;
      
    case 'invoice.payment_failed':
      await handlePaymentFailed(event.data.object);
      break;
      
    case 'customer.subscription.updated':
      await handleSubscriptionUpdated(event.data.object);
      break;
      
    case 'customer.subscription.deleted':
      await handleSubscriptionCancelled(event.data.object);
      break;
  }
}

async function handleInvoicePaid(invoice: Stripe.Invoice): Promise<void> {
  const tenantId = invoice.metadata?.tenantId;
  if (!tenantId) return;
  
  // Reset credits for new period
  await db.subscriptions.update({
    where: { tenantId },
    data: {
      creditsUsed: 0,
      currentPeriodStart: new Date(invoice.period_start * 1000),
      currentPeriodEnd: new Date(invoice.period_end * 1000),
    },
  });
  
  // Record invoice
  await db.invoices.create({
    tenantId,
    stripeInvoiceId: invoice.id,
    totalAmount: invoice.amount_paid,
    status: 'paid',
    paidAt: new Date(),
  });
}

async function handlePaymentFailed(invoice: Stripe.Invoice): Promise<void> {
  const tenantId = invoice.metadata?.tenantId;
  if (!tenantId) return;
  
  // Update subscription status
  await db.subscriptions.update({
    where: { tenantId },
    data: { status: 'past_due' },
  });
  
  // Notify admin
  await sendEmail({
    to: await getTenantAdminEmail(tenantId),
    template: 'payment_failed',
    data: { invoiceUrl: invoice.hosted_invoice_url },
  });
}
```

## Usage Metering

### Recording Usage

```typescript
// Middleware to track usage
async function trackUsage(
  tenantId: string,
  userId: string,
  eventType: UsageEventType,
  metadata?: Record<string, unknown>
): Promise<void> {
  const credits = getCreditsForEvent(eventType);
  
  // Record event
  await db.usageEvents.create({
    tenantId,
    userId,
    eventType,
    credits,
    timestamp: new Date(),
    metadata,
  });
  
  // Update subscription counter (Redis for speed)
  await redis.incrby(`credits:${tenantId}:used`, credits);
}

// Check quota before expensive operations
async function checkQuota(tenantId: string): Promise<QuotaStatus> {
  const subscription = await getSubscription(tenantId);
  const used = await redis.get(`credits:${tenantId}:used`) || 0;
  
  const remaining = subscription.creditsIncluded - used;
  
  return {
    remaining,
    isOverQuota: remaining <= 0,
  };
}

// Enforce quota - call before any metered operation
async function enforceQuota(tenantId: string): Promise<void> {
  const quota = await checkQuota(tenantId);
  
  if (quota.isOverQuota) {
    throw new QuotaExceededError(
      'Credit limit reached. Please wait for your next billing cycle or upgrade your plan.'
    );
  }
}
```

### Quota Exceeded Handling

When credits are exhausted:

```typescript
class QuotaExceededError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'QuotaExceededError';
  }
}

// In search endpoint
async function handleSearch(req: Request): Promise<Response> {
  try {
    await enforceQuota(req.tenantId);
    // ... perform search
  } catch (error) {
    if (error instanceof QuotaExceededError) {
      return Response.json(
        { error: 'quota_exceeded', message: error.message },
        { status: 402 }
      );
    }
    throw error;
  }
}
```
```

## Seat Management

### Adding Seats

```typescript
async function addSeats(tenantId: string, additionalSeats: number): Promise<void> {
  const subscription = await getSubscription(tenantId);
  const newQuantity = subscription.seatsQuantity + additionalSeats;
  
  // Update Stripe
  await stripe.subscriptions.update(subscription.stripeSubscriptionId, {
    items: [{
      id: subscription.stripeSeatItemId,
      quantity: newQuantity,
    }],
    proration_behavior: 'create_prorations',
  });
  
  // Update local
  await db.subscriptions.update({
    where: { tenantId },
    data: {
      seatsQuantity: newQuantity,
      creditsIncluded: getPlan(subscription.planId).creditsPerSeat * newQuantity,
    },
  });
}
```

### Seat Enforcement

```typescript
async function canAddUser(tenantId: string): Promise<boolean> {
  const subscription = await getSubscription(tenantId);
  const userCount = await db.users.count({ where: { tenantId, status: 'active' } });
  
  return userCount < subscription.seatsQuantity;
}

// In user invitation flow
async function inviteUser(tenantId: string, email: string): Promise<void> {
  if (!await canAddUser(tenantId)) {
    throw new Error('Seat limit reached. Please add more seats.');
  }
  
  // ... create invitation
}
```

## Admin Dashboard

### Billing UI Components

```typescript
// Current usage display
interface BillingDashboard {
  // Subscription info
  plan: string;
  status: string;
  nextBillingDate: Date;
  
  // Seats
  seatsUsed: number;
  seatsTotal: number;
  
  // Credits this period
  creditsUsed: number;
  creditsTotal: number;
  creditsRemaining: number;
  
  // Usage breakdown
  usageByType: {
    searches: number;
    chatMessages: number;
    documentsIngested: number;
    emailsIngested: number;
  };
  
  // History
  recentInvoices: Invoice[];
}
```

### Usage Charts

Track and display:
- Daily/weekly/monthly usage trends
- Usage by user (for admin visibility)
- Credits consumption rate
- Projected end-of-period usage

## Self-Serve Billing Portal

Use Stripe Customer Portal:

```typescript
async function createBillingPortalSession(tenantId: string): Promise<string> {
  const subscription = await getSubscription(tenantId);
  
  const session = await stripe.billingPortal.sessions.create({
    customer: subscription.stripeCustomerId,
    return_url: `https://app.example.com/settings/billing`,
  });
  
  return session.url;
}
```

Portal allows customers to:
- Update payment method
- View invoices
- Change plan (within limits)
- Cancel subscription
