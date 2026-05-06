# Webhook Handler Patterns

Production-ready patterns for receiving Stigg webhooks. Companion to `SKILL.md` — go there first for the contract (signature, retries, payload envelope).

## The golden path

```ts
// app/api/stigg/webhooks/route.ts (Next.js App Router)
import { NextRequest, NextResponse } from 'next/server';
import { redis } from '@/lib/redis';
import { queue } from '@/lib/queue';

const SECRET = process.env.STIGG_WEBHOOK_SECRET!;

export async function POST(req: NextRequest) {
  // 1. Verify the secret
  const headerSecret = req.headers.get('Stigg-Webhooks-Secret');
  if (headerSecret !== SECRET) {
    return NextResponse.json({ error: 'Invalid secret' }, { status: 401 });
  }

  // 2. Parse payload
  let event;
  try {
    event = await req.json();
  } catch {
    return NextResponse.json({ error: 'Invalid JSON' }, { status: 400 });
  }
  const { messageId, type, data } = event;
  if (!messageId || !type) {
    return NextResponse.json({ error: 'Missing messageId or type' }, { status: 400 });
  }

  // 3. Idempotency — atomic check-and-set
  const fresh = await redis.set(
    `stigg:webhook:${messageId}`,
    '1',
    { NX: true, EX: 60 * 60 * 24 * 7 },  // 7-day dedup window
  );
  if (!fresh) {
    return NextResponse.json({ ok: true, dedup: true });
  }

  // 4. Acknowledge fast — queue heavy work async
  await queue.enqueue('stigg.webhook', { type, data, messageId });

  return NextResponse.json({ ok: true });
}
```

The handler stays under ~50ms in normal cases. The worker does the actual processing.

## Worker-side dispatch

```ts
// workers/stigg-webhook.worker.ts
async function process(event: { type: string; data: any; messageId: string }) {
  switch (event.type) {
    case 'subscription.created':
      await onSubscriptionCreated(event.data);
      break;
    case 'subscription.canceled':
      await onSubscriptionCanceled(event.data);
      break;
    case 'trial.ends_soon':
      await scheduleWinbackEmail(event.data);
      break;
    case 'trial.expired':
      await onTrialExpired(event.data);
      break;
    case 'credit.balance.low':
      await sendLowBalanceNotification(event.data);
      break;
    case 'credit.balance.depleted':
      await sendDepletedNotification(event.data);
      break;
    case 'automatic_recharge.configuration.changed':
      // Especially important if disabled: { ..., disabled: true }
      if (event.data.disabled) await alertOps('Auto-recharge disabled', event.data);
      break;
    case 'customer.payment_failed':
      await alertOps('Payment failed', event.data);
      await flagCustomerAtRisk(event.data.customerId);
      break;
    case 'third_party_sync_failed':
      await alertOps('Third-party sync failed', event.data);
      break;
    // Silently ignore unknown event types — DON'T return 4xx; that triggers retries.
    default:
      logger.debug('Stigg webhook: unrecognized event', { type: event.type });
  }
}
```

## Order-tolerance pattern

Stigg doesn't guarantee delivery order. When ordering matters, **fetch fresh state from the API instead of trusting the event**:

```ts
async function onSubscriptionCanceled(data: { subscriptionId: string }) {
  // Don't assume this was the latest change — refetch the subscription.
  const sub = await stigg.getSubscription({ subscriptionId: data.subscriptionId });
  if (sub.status !== 'CANCELED' && sub.status !== 'CANCELLATION_SCHEDULED') {
    // Event arrived but a later update reactivated it — ignore.
    return;
  }
  await markUserAccessRevoked(sub.customerId);
}
```

## Reconciliation pattern

Webhooks are best-effort. Pair them with periodic reconciliation:

```ts
// Run hourly via cron or queue
async function reconcileSubscriptions() {
  const customers = await getActiveCustomersFromOurDB();
  for (const c of customers) {
    const stiggSubs = await stigg.listSubscriptions({ customerId: c.id });
    if (driftDetected(c.localState, stiggSubs)) {
      await healState(c.id, stiggSubs);
      logger.warn('Drift between local state and Stigg', { customerId: c.id });
    }
  }
}
```

Critical for any flow where a missed webhook leads to access drift or revenue loss.

## Operational alerting

A sustained high error rate from your webhook endpoint is itself a signal. Recommended alerts:

- **Webhook-handler 5xx rate > 1%** for > 5 minutes — your endpoint is broken.
- **Webhook-handler latency p95 > 5s** — risk of 30s timeout.
- **`automatic_recharge.configuration.changed` with `disabled: true`** → customer-facing alert.
- **`customer.payment_failed`** → revenue-recovery flow.
- **`third_party_sync_failed`** → billing-integration health alert.

## Local-dev webhook testing

Stigg's "Send test event" button (in the webhook detail panel, ⋮ on a specific event) lets you trigger a real-shaped payload at your endpoint without performing the underlying action. Use this in dev:

1. Point the webhook's Endpoint URL at a tunnel (ngrok / Cloudflare Tunnel).
2. Click **Send test event** for the event you're handling.
3. Watch your logs.

For automated tests, mock Stigg by sending a hand-crafted payload at your handler with a valid `Stigg-Webhooks-Secret` header.

## Common mistakes (handler-specific)

| Mistake | Fix |
|---|---|
| Synchronous DB writes / external API calls in the handler | Queue async; ack 2XX in <500ms. |
| `if (event.type === 'X') { ... }` without a default | Unknown event types should be silently logged + acked, not 4xx'd. |
| Idempotency key = `customerId` instead of `messageId` | One customer can have many events. Dedup must use `messageId`. |
| 7-day TTL too short for retry coverage | Stigg's retry window is ~30 minutes total, but consider longer windows for safety on slow workers. |
| Trusting the IP allowlist alone | Combine IP allowlist with secret verification. |
| Returning 4xx for "I don't care about this event" | Triggers retries. Always 2xx for events you intentionally ignore. |
| No reconciliation job | Webhooks are best-effort. Periodic reconciliation catches misses. |
