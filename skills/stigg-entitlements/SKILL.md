---
name: stigg-entitlements
description: RUNTIME skill — use when implementing the runtime side of Stigg — gating feature access (`hasAccess`, `getMeteredEntitlement`, `getBooleanEntitlement`, `getEntitlement`), reporting usage (`reportEvent` for raw events vs `reportUsage` for calculated measurements), promotional entitlements, and configuring local caching / fallback so checks stay reliable when Stigg is unreachable. Triggers include "gate this feature with Stigg", "check entitlement", "is the customer allowed to", "hasAccess", "feature flag with Stigg", "report usage to Stigg", "report a usage event", "increment usage", "events.report vs usage.report", "Stigg fallback strategy", "Stigg cache", "Stigg sidecar", "Stigg goes down", "Stigg offline", "stale entitlements", "promotional entitlement", "grant a one-off entitlement". Skip for catalog modeling (stigg-pricing-modeling), subscription lifecycle (stigg-subscriptions), or generic feature flags with no Stigg integration (e.g., LaunchDarkly).
---

# Stigg Entitlements — Gate Features and Report Usage

This skill covers the runtime side: checking entitlements at request time, reporting usage, and keeping checks reliable when Stigg is briefly unreachable.

## Before You Start

Per the umbrella `stigg` skill: **search first.** SDK init signatures and entitlement-method names evolve — open the relevant SDK / REST page (Mintlify Stigg docs MCP) before authoring.

## The Two Most Important Rules

### Rule 1 — Raw events vs calculated usage is by **throughput**, not by who counts

Stigg accepts usage data through two paths. They look similar; their scaling characteristics are different.

> **Naming note:** the GraphQL mutations / Node SDK methods are `reportEvent` (raw events) and `reportUsage` / `reportUsageBulk` (calculated). Direct REST callers hit `POST /api/v1/events` and `POST /api/v1/usage` for the same operations. The shorthand "events.report" / "usage.report" you'll see used in this skill refers to the conceptual operation, not a specific SDK namespace.

| Path | What it is | Scale model | When to pick |
|---|---|---|---|
| **Raw events** — `reportEvent` (Node SDK / GraphQL) / `POST /api/v1/events` | You stream individual events with dimensions; Stigg filters and aggregates server-side. | **Async, high-throughput.** Designed for production volume. Buffered + deduped. **Limits:** 1 MB per event; 1,000 events per batch; 1,000 events / sec peak. | Default for production usage reporting. Per-request increments, AI-token deductions, billable events at scale. |
| **Calculated usage** — `reportUsage` / `reportUsageBulk` (Node SDK / GraphQL) / `POST /api/v1/usage` | Your app aggregates locally; you push the **delta** or the **set** value to Stigg. | **Sync, measurement-style.** Lower-throughput by design. `reportUsageBulk` accepts up to 100 measurements per call. | Rare, batched updates: "the customer now has 5 seats" (set), or "+2 seats" (delta). Backfills. Periodic infra polling. |

> **Common misconception:** "events" = consumption-y things, "usage" = entitlement-y things. **No.** The split is operational. Both paths can drive metering and billing. Pick by throughput / latency budget, not by what the data represents.

### Rule 2 — Always have a fallback

Entitlement checks are on the critical path of your app. Stigg's SDKs cache locally and support persistent caches; **configure both the cache and the fallback** before going to production. See `references/fallback.md`.

## Gating — The Three Entitlement Shapes

| Feature type | Method (frontend / backend SDK) | Returns |
|---|---|---|
| **Boolean** | `getBooleanEntitlement({ featureId })` | `{ hasAccess, accessDeniedReason? }` |
| **Configuration** | `getNumericEntitlement({ featureId })` / `getEnumEntitlement({ featureId })` | the configured value |
| **Metered** | `getMeteredEntitlement({ featureId, options: { requestedUsage } })` | `{ hasAccess, currentUsage, usageLimit, ... }` |

Re-check the actual method names in the SDK page you're targeting — names occasionally evolve (e.g. `getEntitlement` is also exposed as a unified, generic check on backend SDKs).

Detailed gating patterns — backend / frontend init, per-resource gating, response shape, anti-patterns: `references/gating.md`.

### Frontend (browser) gating

```ts
import Stigg from '@stigg/js-client-sdk';

// initialize() is SYNCHRONOUS — it returns the client.
// To wait for the client to finish loading, await waitForInitialization().
const stiggClient = Stigg.initialize({
  apiKey: 'client-...',          // publishable key, not server key
  customerId: 'customer-123',
});
await stiggClient.waitForInitialization();

const ent = stiggClient.getMeteredEntitlement({
  featureId: 'api_calls',
  options: { requestedUsage: 1 },
});

if (ent.hasAccess) {
  // allow
} else {
  // show paywall / upgrade prompt
}
```

For React: wrap in `<StiggProvider>` and use the hooks the React SDK exposes.

### Backend gating

```ts
import Stigg from '@stigg/node-server-sdk';

const stigg = await Stigg.initialize({ apiKey: process.env.STIGG_SERVER_API_KEY! });

const ent = await stigg.getMeteredEntitlement({
  customerId: 'customer-123',
  featureId: 'api_calls',
  options: { requestedUsage: 1 },
});

if (!ent.hasAccess) throw new HttpError(402, 'Upgrade required');
```

For raw REST, the equivalent is `GET /api/v1/customers/{id}/entitlements`. (The GraphQL query is `getEntitlementsState`.) See the REST API reference for query params.

## Reporting Usage — Pick the Path

> Full event-schema field tables, dimension-design guidance, cloud-ingestion paths (S3 / Pub/Sub), and the GraphQL vs direct-REST surface map: `references/usage-reporting.md`.

### Default — Raw events (`events.report` / `reportEvent`)

```ts
// Backend Node SDK
await stigg.reportEvent({
  events: [{
    customerId: 'customer-123',
    eventName: 'api.request',
    idempotencyKey: 'req-abc-1234',          // required
    dimensions: { region: 'us-east-1', endpoint: '/search' },
    // timestamp is optional; server time is used if omitted
  }],
});
```

Notes:

- **`idempotencyKey` is required.** Events with identical `(timestamp, customerId, resourceId, eventName, idempotencyKey)` are deduped.
- **Dimensions are arbitrary key/value.** Add as many as you can up front — they enable future filtering / aggregation without backfill.
- **Max payload per event: 1 MB.**
- **`resourceId` is required** when the product allows multiple subscriptions (workspaces, projects).

### Set / delta — Calculated usage (`usage.report` / `reportUsage`)

```ts
import { UsageUpdateBehavior } from '@stigg/node-server-sdk';

// "Customer now has 5 seats" — Set behavior
await stigg.reportUsage({
  customerId: 'customer-123',
  featureId: 'seats',
  value: 5,
  updateBehavior: UsageUpdateBehavior.SET,
});

// "+2 seats" — Delta behavior (default)
await stigg.reportUsage({
  customerId: 'customer-123',
  featureId: 'seats',
  value: 2,
  // updateBehavior defaults to DELTA
});
```

Use this when your app has the authoritative value (seat counts, configured quotas) and you don't want Stigg aggregating per-event.

Bulk variant (`reportUsageBulk`) supports up to **100 measurements per call** — note its rate limit.

## Cache + Fallback — Required for Production

Stigg SDKs cache effective entitlements locally so checks don't pay a network round trip on every request. **Configure the cache *and* the fallback** before production:

- **Local in-memory cache** — default in all SDKs. Refreshed via polling (frontend) or real-time updates (backend).
- **Persistent cache** — for serverless / large fleets where in-memory caches die with the process. Backed by Redis via the persistent-cache-service. Supported in Node SDK + Sidecar.
- **Sidecar** — a separate process / deployment that fronts the API. Lower latency, offline mode, global fallback.
- **Fallback rules** — global and per-entitlement. When entitlement data is missing or Stigg is unreachable, the SDK returns the configured fallback (allow / deny, with a marker so you can log).
- **Usage during outages** — entitlement checks keep working from cache; usage reports buffer in-memory (or in the persistent cache) and flush on reconnect.

Full guidance: `references/fallback.md`.

## Promotional Entitlements

Per-customer entitlement overrides — granted directly to a customer, **outside any subscription**. Owned by this skill end-to-end (granting, revoking, reading) since they're per-customer runtime concerns, not catalog modeling.

- **Use cases:** temporary limit increases ("oops, your script blew through your API quota — here's an extra 1k for the rest of the period"), early access to a premium feature for a specific customer, enterprise accommodations, goodwill gestures.
- **Additive** with plan and add-on entitlements (the generous-entitlement rule applies).
- **Time-bounded** (1 month / 1 year / lifetime) or until manually revoked.
- **Granting** is a mutation — `grantPromotionalEntitlements` (GraphQL) / equivalent SDK method / available in the MCP. Pass `customerId`, `featureId`, the override value, and the period.
- **Revoking** uses `revokePromotionalEntitlement`.

Modeling shape (which features exist) lives in `stigg-pricing-modeling`; promotional entitlements just *override* configuration on those features for a specific customer.

## When NOT to Use This Skill

- Modeling features / plans / add-ons → `stigg-pricing-modeling`.
- Provisioning / canceling / migrating subscriptions → `stigg-subscriptions`.
- Granting credit balances at runtime → `stigg-credits`.
- Picking the right pricing model for the customer's product → `stigg-pricing-expert`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Picking `usage.report` because it "sounds canonical" | The split is by throughput. For production volume, default to `events.report`. |
| Omitting `idempotencyKey` on events | Required. Without it you get either rejections or duplicate counting. |
| Reporting usage from the browser | Usage reporting is a backend concern; the publishable key is read-only. |
| Treating cache as optional | Cache + fallback are part of the contract for production. Configure both. |
| Defining no fallback rules | When Stigg is briefly unreachable and the cache is cold, the SDK has nothing to return. Pick a sensible default (usually allow with a log). |
| Synchronously calling Stigg on every API request | The SDK already caches — let it. Don't double-cache in your app. |
| Per-call `await` to `events.report` in the hot path | Buffer / batch and flush async. The events path is designed for this. |
| Using a raw event when a configuration value would do | Configuration features (numeric / enum) are simpler to model and don't need event ingestion. |
