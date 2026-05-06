---
name: stigg-webhooks
description: Use when receiving, verifying, or handling webhook events from Stigg — subscription lifecycle (`subscription.created`, `subscription.canceled`, `trial.expired`, `trial.ends_soon`, etc.), entitlement events (`entitlement_usage_exceeded`), credit events (`credit.balance.low`, `credit.balance.depleted`, `automatic_recharge.*`), and others. Triggers include "Stigg webhook handler", "verify Stigg webhook", "Stigg webhook signature", "credit.balance.low", "trial.expired webhook", "Stigg webhook payload shape", "Stigg webhook retry", "messageId", "Stigg-Webhooks-Secret". Skip for inbound API calls TO Stigg (use stigg-api / stigg-entitlements).
---

# Stigg Webhooks — Receive and Handle Events

Stigg notifies your app about changes via webhooks: subscription lifecycle, trial transitions, credit balance thresholds, auto-recharge attempts, entitlement-usage exceeded, and more. This skill covers **receiving** them — registering the endpoint, verifying authenticity, handling retries, and the payload envelope.

## Before You Start

Per the umbrella `stigg` skill: **search first.** Webhook event names and payloads evolve. Use the Mintlify Stigg docs MCP — `query_docs_filesystem_stigg` against `/documentation/native-integrations/webhooks/events.mdx` and the per-event reference pages — to confirm the exact payload schema before authoring handler code.

## Registering a Webhook

In the Stigg app: **Settings → Integrations → Webhooks → + New webhook**.

| Field | Notes |
|---|---|
| **Service name** | Human-friendly label (e.g. "Production webhooks", "Dev"). |
| **Endpoint URL** | Your handler — e.g. `https://yourapp.com/api/stigg/webhooks`. Must respond `2XX` within 30 seconds or Stigg retries. |
| **Events to send** | Pick from the catalog (subscription, trial, credits, entitlement, customer, etc.). |

Created webhooks **activate immediately**. Use **Send test event** (⋮ menu on a specific event) to verify your handler before wiring real events.

## Authenticity — Verify the Secret

Every webhook event carries a header:

```text
Stigg-Webhooks-Secret: <secret>
```

Compare this header value to the secret shown in the webhook's detail page. **They must match.**

> **HMAC signature support is on Stigg's roadmap.** Today, verification is plain secret comparison — treat the secret like an API key (env var, never in source).

## Payload Envelope

Every webhook event has a top-level shape (per-event details vary; verify the per-event page):

```json
{
  "messageId": "msg_abc123",     // unique per event delivery — use for idempotency
  "type": "subscription.created", // event type
  "createdAt": "2026-01-15T10:30:00Z",
  "data": { /* event-specific payload */ }
}
```

The exact field names may vary slightly across event types — **always confirm against the live event reference page** (`/documentation/native-integrations/webhooks/events.mdx` and the per-event subpages).

## Idempotency — De-Duplicate via `messageId`

Stigg may deliver the same event multiple times (retries, network, edge cases). Always:

1. Read `messageId` from the payload.
2. Check it against a deduplication store (Redis, DB row, idempotency table).
3. **If already processed, return 200 immediately** — don't re-execute the handler.
4. If new, process the event, then record `messageId` as processed.

## Order Is NOT Guaranteed

Stigg does not guarantee delivery order. Your handlers must tolerate out-of-order events:

- A `subscription.canceled` may arrive before the matching `subscription.updated` it followed in time.
- A `credit.balance.depleted` may arrive after a follow-up grant has already been issued.

When order matters, **fetch fresh state from the Stigg API** rather than reconstructing it from the event stream.

## Retry Behavior

If your endpoint returns non-2XX or times out (>30s), Stigg retries:

1. **3 immediate retries** (no backoff).
2. **3 more retries every 30 seconds.**

After 6 total failures the event is **failed**. Failed events are stored on Stigg's backend and **can be resent on request** (contact support; a self-service replay UI is on the roadmap). Stigg also actively monitors webhook delivery failures and reaches out when failure rates spike.

> **Always return 200 fast.** Acknowledge receipt, then queue the work async. A handler that synchronously processes complex logic risks the 30-second timeout and unnecessary retries.

## IP Allowlist

Webhook events originate from Stigg's IPs. At the time of writing: **`18.119.35.43`**. The list **may change** — re-fetch from `/documentation/native-integrations/webhooks/index.mdx` before pinning.

## Event Catalog (by domain)

The full event list lives at [`/documentation/native-integrations/webhooks/events.mdx`](https://docs.stigg.io/documentation/native-integrations/webhooks/events). Re-fetch for the canonical set; below is a curated map by domain:

| Domain | Common events |
|---|---|
| **Customer** | `customer.created`, `customer.updated`, `customer.deleted`, `customer.payment_failed`, `payment_method.attached`, `payment_method.detached` |
| **Subscription** | `subscription.created`, `subscription.updated`, `subscription.canceled`, `subscription.creation_failed`, `subscription.expired_non_recurring`, `billing.month_ends_soon`, `subscription.spend_limit_exceeded` |
| **Trial** | `trial.started`, `trial.ends_soon` (7 days before expiry), `trial.expired`, `trial.converted` |
| **Entitlement** | `entitlement.usage_exceeded`, `entitlements.updated` |
| **Credits — balance** | `credit.balance.low`, `credit.balance.depleted` |
| **Credits — grants** | `credit.grant.created`, `credit.grant.updated`, `credit.grant.depleted`, `credit.grant.expired`, `credit.grant.balance_low` |
| **Credits — auto-recharge** | `automatic_recharge.operation.attempted`, `credits.automatic_recharge_limit_exceeded` (80/90/100% of cap), `automatic_recharge.configuration.changed` |
| **Promotional entitlements** | `promotional_entitlement.granted`, `promotional_entitlement.updated`, `promotional_entitlement.revoked`, `promotional_entitlement.expired`, `promotional_entitlement.ends_soon` |
| **Catalog** | `plan.created`, `plan.updated_new_version_published`, `plan.deleted`, `addon.created`, `addon.updated_new_version_published`, `addon.deleted`, `feature.created`, `feature.updated`, `feature.deleted`, `coupon.created`, `coupon.updated`, `coupon.archived` |
| **Third-party** | `third_party_sync_failed` |

Exact event names may evolve — always re-confirm in the docs.

## Handler Skeleton (Next.js App Router + Node)

```ts
// app/api/stigg/webhooks/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { redis } from '@/lib/redis';

const SECRET = process.env.STIGG_WEBHOOK_SECRET!;

export async function POST(req: NextRequest) {
  // 1. Verify the secret
  const headerSecret = req.headers.get('Stigg-Webhooks-Secret');
  if (headerSecret !== SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 });
  }

  const event = await req.json();
  const { messageId, type, data } = event;

  // 2. Idempotency
  const seen = await redis.set(
    `stigg:webhook:${messageId}`,
    '1',
    { NX: true, EX: 60 * 60 * 24 * 7 }, // 7-day window
  );
  if (!seen) {
    return NextResponse.json({ ok: true, dedup: true }); // already processed
  }

  // 3. Acknowledge fast — queue heavy work async
  await queue.enqueue({ type, data });
  return NextResponse.json({ ok: true });
}
```

Keep the handler under ~500ms; do all real work in a worker. Stigg only cares about the 2XX.

## When NOT to Use This Skill

- Calling Stigg APIs *outbound* (provisioning, gating, reporting) — use `stigg-api`, `stigg-subscriptions`, `stigg-entitlements`, `stigg-credits`.
- Listening for *frontend* state changes (cache refresh after checkout, etc.) — that's an SDK concern, not a webhook concern.
- Setting up the Stigg MCP server — use `stigg-mcp`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Skipping signature verification | The `Stigg-Webhooks-Secret` header is the only authenticity check today. Compare it. |
| Processing synchronously in the handler | Return 2XX fast; queue heavy work async. Sync processing risks the 30-second timeout. |
| Assuming event order | Stigg explicitly does not guarantee order. Either tolerate out-of-order or fetch fresh state from the API. |
| No idempotency on `messageId` | Stigg retries on failure — without dedup, you'll process the same event twice. |
| Trusting the IP allowlist as your only check | The IP list can change. Use both IP allowlisting AND secret verification. |
| Returning 4XX for "I don't recognize this event" | That triggers retries. Return 2XX and silently ignore unknown event types. |
| Hand-coding HMAC verification | HMAC support isn't shipped yet — a roadmap item. Today, plain secret comparison is the contract. |
| Not handling failed delivery | If Stigg's 6-attempt schedule fails, events are stored but not auto-replayed. Contact support to resend; build operational alerting for sustained failures. |
| Treating webhooks as the only source of truth | Order isn't guaranteed and replay isn't self-service. Pair webhooks with periodic state reconciliation against the Stigg API. |
| Subscribing to "all events" | Noisy, expensive on your side. Pick the events you actually act on. |
