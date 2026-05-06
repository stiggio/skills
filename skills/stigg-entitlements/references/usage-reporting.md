# Usage Reporting — Runtime Reference

How to push consumption data to Stigg, and how to pick the right path.

## Surfaces

| Concept | Node SDK / GraphQL mutation | Direct REST endpoint |
|---|---|---|
| Raw events | `reportEvent` | `POST /api/v1/events` |
| Calculated usage | `reportUsage` | `POST /api/v1/usage` (request body: `{ usages: [{ customerId, featureId, value, ... }] }`) |
| Calculated usage (bulk) | `reportUsageBulk` (up to 100 measurements per call) | `POST /api/v1/usage` with the array form |

The "events.report" / "usage.report" shorthand used elsewhere in this skill refers to the conceptual operation, not any specific SDK namespace. Pick the SDK / endpoint that matches your call surface.

## The two paths — full picture

| Aspect | Raw events | Calculated usage |
|---|---|---|
| **What you send** | Individual events with `eventName`, `idempotencyKey`, optional `dimensions`, optional `timestamp` | A scalar value, with `updateBehavior: SET` or `DELTA` |
| **Where aggregation happens** | Server-side, against the metered feature's **meter** (sum / count / count_unique / etc.) | Your app already aggregated; Stigg stores the value |
| **Throughput target** | High (production traffic, per-request) | Lower (periodic / batch / authoritative-state updates) |
| **Latency** | Async; events buffer + dedupe | Sync; designed for low-volume callers |
| **Bulk variant** | n/a (events are already designed for high volume) | `reportUsageBulk` — Stigg's docs at the time of writing describe batches of up to 100 measurements per call; verify the current cap before sizing batch jobs |

**Pick by throughput.** "Events vs measurements" is not "consumption vs entitlement". Both can drive metering and billing. Both can deduct credits. Same featureId can — in principle — accept either path; pick the one that fits your scale.

## Raw events — schema

| Field | Required | Notes |
|---|---|---|
| `customerId` | ✅ | The customer being charged. |
| `eventName` | ✅ | Logical action (`api.request`, `email.sent`, `token.consumed`). |
| `idempotencyKey` | ✅ | Unique per `(timestamp, customerId, resourceId, eventName, idempotencyKey)`. Used for dedup. |
| `resourceId` | Conditional | Required when the product allows multiple subscriptions (workspaces / projects). |
| `dimensions` | optional | Map of `string → string \| number \| boolean`. Free-form; used for filtering and aggregation. |
| `timestamp` | optional | UTC; server time used if omitted. |

**Limits:** max 1 MB per event; max **1,000 events per batch** call.

### Dimensions — design them up front

A metered feature with a `sum` aggregation needs to know *which dimension* to sum. A `count_unique` needs to know *which dimension* makes events unique (e.g., `customerId` for active-users, `userId` for active-seats).

Backfill is painful — when in doubt, **add the dimension up front**. Common dimensions:

- `region` — `us-east-1`, `eu-central-1`
- `environment` — `production`, `dev`
- `tier` / `endpoint` / `model` — to support per-cohort meters in the future

### Bulk send pattern

```ts
// Buffer events in your app, flush in batches
const buffer: Event[] = [];

function recordEvent(e: Event) {
  buffer.push(e);
  if (buffer.length >= 100) flush();
}

async function flush() {
  const batch = buffer.splice(0);
  if (batch.length === 0) return;
  await stigg.reportEvent({ events: batch });
}

setInterval(flush, 1000);  // and on graceful shutdown
```

For very high volume, prefer the cloud ingestion paths (S3 / GCP Pub/Sub) covered in the docs — Stigg picks events up directly without your app being on the request path.

## Calculated usage — `SET` vs `DELTA`

```ts
import { UsageUpdateBehavior } from '@stigg/node-server-sdk';
// (typed SDKs expose the enum; raw GraphQL accepts the string "DELTA" / "SET")

// Default: DELTA — "the value changed by this much"
await stigg.reportUsage({
  customerId: 'c-123',
  featureId: 'seats',
  value: +2,
  // updateBehavior defaults to DELTA
});

// SET — "the value is now this"
await stigg.reportUsage({
  customerId: 'c-123',
  featureId: 'seats',
  value: 5,
  updateBehavior: UsageUpdateBehavior.SET,
});
```

Use `SET` when your app has the **authoritative current value**. Use `DELTA` when you only know the *change* (and Stigg holds the running total).

GraphQL mutation `reportUsage` input fields (per the docs):

| Field | Type | Notes |
|---|---|---|
| `customerId` | required | Customer reference ID |
| `featureId` | required | Feature reference ID |
| `value` | required | Usage value to report |
| `resourceId` | optional | Required for multi-resource subscriptions |
| `updateBehavior` | optional | `DELTA` (default) or `SET` |
| `createdAt` | optional | Timestamp of when the usage was recorded |
| `dimensions` | optional | Additional dimensions for the usage report |
| `environmentId` | optional | Environment ID |

## Cloud ingestion (raw events at scale)

For production volume, push events directly from your infrastructure rather than from app code:

- **Amazon S3** — drop event files in a configured bucket; Stigg ingests.
- **Google Cloud Pub/Sub** — publish events to a topic Stigg subscribes to.

This bypasses your app entirely on the report path, eliminating buffering / retry concerns from app code. Re-fetch the docs page when adopting — bucket / topic configuration is environment-specific.

## Viewing usage events (debugging)

In the Stigg app, **Usage events** in the left nav shows recently reported events. Use it to confirm events are arriving and dimensions look right. Most "the meter shows zero" issues are visible there within seconds.

## Common mistakes

| Mistake | Fix |
|---|---|
| `idempotencyKey` reused across distinct events | Same key collapses to one event. Use a unique key per logical occurrence. |
| Forgetting `resourceId` in a multi-subscription product | The event is attributed to the customer-level subscription, often wrong. Always pass `resourceId`. |
| Per-call `await stigg.reportEvent` in the request hot path | Buffer + batch + flush async. Latency stays predictable. |
| Mixing `SET` and `DELTA` for the same feature | Pick one. `SET` is "I know the truth"; `DELTA` is "here's the change". |
| Reporting calculated usage from many parallel processes | If you don't centralize, totals drift. Either centralize, or use `events.report` with `count`/`sum` aggregation. |
| Trying to undo an over-reported event | Idempotency means dedup, not delete. Re-model with corrective events or use the credit-ledger model for true reversibility. |
| Not setting a meter on a metered feature | A metered feature without a meter does nothing. Configure meter + dimensions before sending events. |
