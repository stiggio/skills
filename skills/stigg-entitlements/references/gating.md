# Gating — Runtime Reference

How to ask "does this customer have access?" at request time, on backend or frontend.

## What gating actually does

Stigg pre-computes a customer's **effective entitlements** (plan + add-ons + promotional + trial) into a single set per customer. Gating reads this set. Your app does **not** stitch entitlements together — Stigg already did.

## Backend gating

The unified, recommended check on backend SDKs:

```ts
const ent = await stigg.getEntitlement({
  customerId: 'customer-123',
  featureId: 'api_calls',
  options: { requestedUsage: 1 },
});

if (!ent.hasAccess) {
  throw new HttpError(402, 'Upgrade required');
}
```

Type-specific shortcuts also exist (re-confirm names per SDK):

- `getBooleanEntitlement({ customerId, featureId })`
- `getNumericEntitlement` / `getEnumEntitlement`
- `getMeteredEntitlement({ customerId, featureId, options: { requestedUsage } })`

`requestedUsage` is the amount the customer is *about to consume*. Stigg returns `hasAccess: false` if granting it would exceed the entitlement.

## Frontend gating

Use the **publishable** key. Initialize once per session, scope to the current customer.

`Stigg.initialize()` on the JS client SDK is **synchronous** — it returns the client immediately. Use `await client.waitForInitialization()` to wait for the client to finish loading entitlement data.

```ts
import Stigg from '@stigg/js-client-sdk';

const stiggClient = Stigg.initialize({
  apiKey: 'client-...',
  customerId: 'customer-123',
});
await stiggClient.waitForInitialization();

const ent = stiggClient.getMeteredEntitlement({
  featureId: 'api_calls',
  options: { requestedUsage: 3 },
});

if (ent.hasAccess) {
  // allow the action
}
```

For React, prefer the React SDK's hooks (`useStiggContext`, etc.). Re-fetch the React SDK page to confirm the current API.

## Per-resource gating (multi-tenancy)

When the product allows multiple subscriptions per customer (workspaces, projects), gating accepts a `resourceId`:

```ts
await stigg.getEntitlement({
  customerId: 'customer-123',
  resourceId: 'workspace-abc',
  featureId: 'api_calls',
  options: { requestedUsage: 1 },
});
```

Without `resourceId`, the check uses the customer-level subscription set — usually wrong in multi-tenant setups.

## Entitlement responses — structure

A typical metered entitlement response includes:

- `hasAccess: boolean`
- `currentUsage: number`
- `usageLimit: number | null` (null = unlimited)
- `isFallback: boolean` — set when the SDK returned a fallback (cache miss + Stigg unreachable)
- `accessDeniedReason?: string` — why access was denied
- `feature: { id, displayName, ... }` — for UI

The exact shape evolves — re-check the SDK page.

## Reading the cached set in bulk

For UI rendering (e.g., a paywall card showing all features and their current quotas), you usually don't want to call `getEntitlement` per feature. Use the bulk path:

- **GraphQL:** `getEntitlementsState` query.
- **REST:** `GET /api/v1/customers/{id}/entitlements`.
- **Frontend:** the React / JS SDK exposes a way to read the whole effective set in one round-trip — see the SDK page.

## Gating decisions — patterns

| Decision | Pattern |
|---|---|
| Allow up to a quota, deny over | `getMeteredEntitlement` with `requestedUsage`, branch on `hasAccess`. |
| Allow soft, log over | Always allow; report usage; surface a warning when `currentUsage > usageLimit`. (Common for "soft enforcement" while you tune the quota.) |
| Block in UI before the call | Call `getMeteredEntitlement({ requestedUsage: 0 })` to read state; show paywall if `usageLimit - currentUsage <= 0`. |
| Different behavior per tier | Read the entitlement value, branch in code. Don't gate by **plan ID** — features are the right level of abstraction. |

## Anti-patterns

| Anti-pattern | Why it's wrong |
|---|---|
| Gating by **plan ID** in code (`if (planId === 'pro') ...`) | Plans evolve; features stay. Gate by featureId. |
| Calling Stigg on every request without caching | The SDK already caches — let it. |
| Reading from the cache and immediately stale-checking against Stigg | The SDK refreshes periodically (or in real time on backend). Don't hand-roll. |
| Mixing entitlement values from your DB with Stigg's cache | One source of truth. Stigg is it for entitlements. |
| Running gating logic in the browser only | Always also gate on the backend. The browser is untrusted. |
