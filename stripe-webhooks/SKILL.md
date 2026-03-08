---
name: stripe-webhooks
description: >
  Stripe webhook handling patterns for HireFlow's subscription billing. Use this skill
  whenever writing or modifying the Stripe webhook endpoint, handling subscription
  lifecycle events, updating org plan state, writing billing-related database logic,
  debugging payment or subscription issues, adding new Stripe event handlers, or any
  time the words "webhook", "Stripe event", "subscription", "billing", "plan upgrade",
  "dunning", "invoice", or "checkout" come up. Always use this skill before touching
  app/api/billing/ or lib/stripe/. The patterns here prevent real money bugs.
---

# Stripe Webhook Handling — HireFlow

## The Cardinal Rules

1. **Verify the signature first.** Always. No exceptions. An unverified webhook can be spoofed.
2. **Write to `billing_events` before processing.** The insert is the idempotency check — if it fails with a unique key violation, the event was already processed.
3. **Never trust client-supplied plan data.** Always derive plan state from Stripe's webhook payload, not from what the frontend sends.
4. **Store the full payload as JSONB.** You will need it for debugging. Always.
5. **Return 200 even on business logic errors.** Returning 4xx/5xx makes Stripe retry. Only return non-200 for genuine infrastructure failures.

---

## File Structure

```
app/api/billing/
├── webhook/
│   └── route.ts        # Stripe webhook endpoint
├── checkout/
│   └── route.ts        # Create Checkout session for upgrade
└── portal/
    └── route.ts        # Create Customer Portal session URL

lib/stripe/
├── client.ts           # Stripe SDK instance
├── webhooks.ts         # Event handler functions
└── plans.ts            # Plan → price ID mapping and limits
```

---

## Webhook Endpoint

```typescript
// app/api/billing/webhook/route.ts
import { headers } from 'next/headers'
import { stripe } from '@/lib/stripe/client'
import { handleStripeEvent } from '@/lib/stripe/webhooks'
import { adminClient } from '@/lib/supabase/admin'

export async function POST(req: Request) {
  const body = await req.text()  // Must be raw text for signature verification
  const sig = headers().get('stripe-signature')

  if (!sig) {
    return Response.json({ error: 'Missing signature' }, { status: 400 })
  }

  // Step 1: Verify signature — reject anything that fails
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    console.error('Webhook signature verification failed:', err)
    return Response.json({ error: 'Invalid signature' }, { status: 400 })
  }

  // Step 2: Idempotency check — insert billing_event first
  // If stripe_event_id already exists, the insert throws a unique violation
  const { error: insertError } = await adminClient
    .from('billing_events')
    .insert({
      org_id: await getOrgIdFromEvent(event),  // see helper below
      event_type: event.type,
      stripe_event_id: event.id,               // UNIQUE — duplicate = already processed
      stripe_payload: event,                   // Full payload stored as JSONB
      processed_at: new Date().toISOString(),
    })

  if (insertError) {
    if (insertError.code === '23505') {
      // Unique violation — already processed, Stripe is retrying
      console.log(`Duplicate webhook ignored: ${event.id}`)
      return Response.json({ received: true })  // Return 200 to stop retries
    }
    // Genuine DB error — return 500 so Stripe retries
    console.error('Failed to insert billing event:', insertError)
    return Response.json({ error: 'Database error' }, { status: 500 })
  }

  // Step 3: Handle the event
  try {
    await handleStripeEvent(event)
  } catch (err) {
    // Log but return 200 — the billing_event row was written, don't retry
    // Investigate via billing_events table
    console.error(`Error handling ${event.type}:`, err)
  }

  return Response.json({ received: true })
}
```

---

## Event Handlers

```typescript
// lib/stripe/webhooks.ts
import Stripe from 'stripe'
import { adminClient } from '@/lib/supabase/admin'
import { PLAN_FROM_PRICE_ID } from './plans'

export async function handleStripeEvent(event: Stripe.Event) {
  switch (event.type) {

    case 'customer.subscription.created':
    case 'customer.subscription.updated': {
      const sub = event.data.object as Stripe.Subscription
      await syncSubscription(sub)
      break
    }

    case 'customer.subscription.deleted': {
      const sub = event.data.object as Stripe.Subscription
      await cancelSubscription(sub)
      break
    }

    case 'invoice.payment_succeeded': {
      const invoice = event.data.object as Stripe.Invoice
      await handlePaymentSucceeded(invoice)
      break
    }

    case 'invoice.payment_failed': {
      const invoice = event.data.object as Stripe.Invoice
      await handlePaymentFailed(invoice)
      break
    }

    case 'customer.subscription.trial_will_end': {
      const sub = event.data.object as Stripe.Subscription
      await handleTrialEnding(sub)
      break
    }

    default:
      // Unknown events are fine — just log them, don't error
      console.log(`Unhandled Stripe event type: ${event.type}`)
  }
}

async function syncSubscription(sub: Stripe.Subscription) {
  const org = await getOrgByStripeCustomerId(sub.customer as string)
  if (!org) throw new Error(`No org for customer ${sub.customer}`)

  const priceId = sub.items.data[0]?.price.id
  const plan = PLAN_FROM_PRICE_ID[priceId] ?? 'starter'

  const planStatus = mapStripeStatus(sub.status)
  const limits = PLAN_LIMITS[plan]

  await adminClient
    .from('organizations')
    .update({
      plan,
      plan_status: planStatus,
      stripe_subscription_id: sub.id,
      current_period_ends_at: new Date(sub.current_period_end * 1000).toISOString(),
      seat_limit: limits.seat_limit,
      job_limit: limits.job_limit,
      updated_at: new Date().toISOString(),
    })
    .eq('id', org.id)
}

async function cancelSubscription(sub: Stripe.Subscription) {
  const org = await getOrgByStripeCustomerId(sub.customer as string)
  if (!org) return

  await adminClient
    .from('organizations')
    .update({
      plan_status: 'canceled',
      updated_at: new Date().toISOString(),
    })
    .eq('id', org.id)
}

async function handlePaymentSucceeded(invoice: Stripe.Invoice) {
  if (!invoice.subscription) return  // One-time payment, not subscription
  const sub = await stripe.subscriptions.retrieve(invoice.subscription as string)
  await syncSubscription(sub)
}

async function handlePaymentFailed(invoice: Stripe.Invoice) {
  if (!invoice.customer) return
  const org = await getOrgByStripeCustomerId(invoice.customer as string)
  if (!org) return

  await adminClient
    .from('organizations')
    .update({
      plan_status: 'past_due',
      updated_at: new Date().toISOString(),
    })
    .eq('id', org.id)

  // TODO: Send dunning email via Resend
}

async function handleTrialEnding(sub: Stripe.Subscription) {
  // Fires 3 days before trial ends
  const org = await getOrgByStripeCustomerId(sub.customer as string)
  if (!org) return
  // TODO: Send trial ending warning email via Resend
}
```

---

## Plan Configuration

```typescript
// lib/stripe/plans.ts

export const PLAN_LIMITS = {
  trial:   { seat_limit: 1,    job_limit: 3    },
  starter: { seat_limit: 1,    job_limit: 3    },
  pro:     { seat_limit: 5,    job_limit: null }, // null = unlimited
} as const

// Map Stripe Price IDs → HireFlow plan names
export const PLAN_FROM_PRICE_ID: Record<string, 'starter' | 'pro'> = {
  [process.env.STRIPE_STARTER_PRICE_ID!]: 'starter',
  [process.env.STRIPE_PRO_PRICE_ID!]:     'pro',
}

// Map Stripe subscription status → HireFlow plan_status enum
export function mapStripeStatus(stripeStatus: string): string {
  const map: Record<string, string> = {
    trialing:        'trialing',
    active:          'active',
    past_due:        'past_due',
    canceled:        'canceled',
    unpaid:          'past_due',
    incomplete:      'past_due',
    incomplete_expired: 'canceled',
    paused:          'canceled',
  }
  return map[stripeStatus] ?? 'canceled'
}
```

---

## Checkout Session (Plan Upgrade)

```typescript
// app/api/billing/checkout/route.ts
import { stripe } from '@/lib/stripe/client'
import { createClient } from '@/lib/supabase/server'

export async function POST(req: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return Response.json({ error: 'Unauthorized' }, { status: 401 })

  const { priceId } = await req.json()
  const { data: org } = await supabase
    .from('organizations')
    .select('stripe_customer_id, plan')
    .eq('id', user.user_metadata.org_id)
    .single()

  // Create or reuse Stripe customer
  let customerId = org?.stripe_customer_id
  if (!customerId) {
    const customer = await stripe.customers.create({
      email: user.email,
      metadata: { org_id: user.user_metadata.org_id },
    })
    customerId = customer.id
    // Store immediately — don't wait for webhook
    await supabase.from('organizations').update({
      stripe_customer_id: customerId
    }).eq('id', user.user_metadata.org_id)
  }

  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: 'subscription',
    payment_method_types: ['card'],
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/settings?upgraded=true`,
    cancel_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/settings?upgrade=canceled`,
    subscription_data: {
      metadata: { org_id: user.user_metadata.org_id },
    },
  })

  return Response.json({ url: session.url })
}
```

---

## Customer Portal (Self-Service)

```typescript
// app/api/billing/portal/route.ts
export async function POST(req: Request) {
  // ... auth check ...
  const session = await stripe.billingPortal.sessions.create({
    customer: org.stripe_customer_id,
    return_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/settings`,
  })
  return Response.json({ url: session.url })
}
```

---

## Org Helper

```typescript
// lib/stripe/webhooks.ts (helper)
async function getOrgIdFromEvent(event: Stripe.Event): Promise<string | null> {
  const obj = event.data.object as any
  const customerId = obj.customer ?? obj.id

  if (!customerId) return null

  const { data: org } = await adminClient
    .from('organizations')
    .select('id')
    .eq('stripe_customer_id', customerId)
    .single()

  return org?.id ?? null
}

async function getOrgByStripeCustomerId(customerId: string) {
  const { data } = await adminClient
    .from('organizations')
    .select('*')
    .eq('stripe_customer_id', customerId)
    .single()
  return data
}
```

---

## Local Testing with Stripe CLI

```bash
# Install Stripe CLI, then forward webhooks to local dev server
stripe listen --forward-to localhost:3000/api/billing/webhook

# Trigger specific events for testing
stripe trigger customer.subscription.created
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.deleted

# The CLI prints your local webhook signing secret — use this in .env.local:
# STRIPE_WEBHOOK_SECRET=whsec_...
```

---

## Common Bugs

```typescript
// ❌ WRONG: Reading raw body after parsing
export async function POST(req: Request) {
  const json = await req.json()        // This consumes the body
  const body = await req.text()        // Now empty — signature check fails!

// ✅ CORRECT: Read as text first, parse manually if needed
export async function POST(req: Request) {
  const body = await req.text()        // Raw text for signature verification
  const event = stripe.webhooks.constructEvent(body, sig, secret)
  // event.data.object is already parsed


// ❌ WRONG: Returning 500 on business logic failure (causes infinite retries)
  } catch (err) {
    return Response.json({ error: 'Failed' }, { status: 500 })
  }

// ✅ CORRECT: Log and return 200 after billing_events row is written
  } catch (err) {
    console.error('Handler error:', err)
    return Response.json({ received: true })  // Stop retries
  }


// ❌ WRONG: Updating org plan without idempotency check
await updateOrgPlan(orgId, 'pro')  // Runs twice if Stripe retries

// ✅ CORRECT: Insert billing_event first (unique constraint = idempotency)
const { error } = await insert billing_events with stripe_event_id
if (error?.code === '23505') return 200  // Already done
await updateOrgPlan(orgId, 'pro')  // Safe to run now
```

---

## Checklist for New Event Handlers

- [ ] Event type added to the switch statement in `handleStripeEvent`
- [ ] Uses `adminClient` (service role), not session-scoped client
- [ ] Looks up org by `stripe_customer_id`, not by client-supplied ID
- [ ] Updates `organizations` table with correct plan limits from `PLAN_LIMITS`
- [ ] Returns nothing (throws on error — caught by the outer try/catch in the route)
- [ ] Tested locally with `stripe trigger [event-type]`
- [ ] Edge case handled: what if the org doesn't exist?
