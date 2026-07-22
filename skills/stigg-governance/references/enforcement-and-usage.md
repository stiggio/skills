# Enforcement and Usage — Attribution, Access Checks, and the Governance Tree

The runtime half of governance: how usage lands on entities, how access is gated, and how balances are read. **Confirm response shapes via the Stigg MCP's `search_docs` before building against them.**

## Usage attribution — the meter-type fork

Ingest never gates; it records consumption and rolls it up the tree. **Which endpoint to call depends on the target feature's meter type.** In both paths, attribution happens through **dimensions**:

1. The entity type declares up to 2 `attributionKeys` (e.g. `projectId`, `regionId` — whatever identifies the customer's units).
2. Your ingest call carries those keys' values in `dimensions`.
3. Stigg matches dimension values to entity IDs and attributes the usage to the entity **and all of its ancestors**.

A call with no attribution-key dimensions attributes to no entity — customer-level metering still works, but governance budgets never see it.

### Calculated / incremental meters → `usage.report`

Features whose usage your app computes and reports directly use **`client.v1.usage.report`** (run via the MCP `execute` tool): a **synchronous** report that increments the counter and rolls up the tree immediately. The response echoes the new `currentUsage`. `featureId` accepts either a metered feature id **or** a credit currency id.

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
// Calculated/incremental meters; required: customerId, featureId, value.
await client.v1.usage.report({
  usages: [{
    customerId: 'customer-123',
    featureId: 'feature-ai-tokens',
    value: 1250,
    dimensions: { projectId: 'proj-atlas', regionId: 'eu-west' },
  }],
});
// → response.currentUsage reflects the rolled-up total (ancestors included)
```

### Raw-events meters → `events.report` (reportEvent)

Features backed by an **event meter** (Stigg aggregates raw events — count/sum/unique — into usage) **cannot** be fed via `usage.report`; they **require** **`client.v1.events.report`** (reportEvent, run via the MCP `execute` tool). This is the current reality — usage-reporting events for these features is not supported. It's **high-volume, eventually-consistent** (~seconds of lag).

The event does **not** carry a `featureId`. The meter is matched by **`eventName`** (must equal the meter's configured event); `idempotencyKey` dedupes; **entity attribution rides in `dimensions`**.

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
// Raw-events meters; no featureId — the meter is matched by eventName.
await client.v1.events.report({
  events: [{
    customerId: 'customer-123',
    eventName: 'ai-tokens-consumed',
    idempotencyKey: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890',
    dimensions: { projectId: 'proj-atlas', regionId: 'eu-west' },
  }],
});
// → accepted; accrues asynchronously
```

### Picking the path — try, then fall back

Meter type is **not reliably readable via the API** — the feature's `meterType` enum (`NONE`/`INCREMENTAL`/`FLUCTUATING`) has no "raw events" value; event meters live in a separate `meter`/`aggregation` config. So don't try to detect it up front: **call `usage.report`; if it returns `CannotReportUsageForEntitlementWithMeterError`, the feature is a raw-events meter — switch to `events.report`.** Configuring event meters is a catalog concern — see `stigg-pricing-modeling` / `stigg-entitlements`.

## Access — the entitlements check

Gate requests with the standard entitlements check — `client.v1Beta.customers.entitlements.check` (run via the MCP `execute` tool). Pass `featureId` **or** `currencyId` (not both), an optional `requestedUsage` (defaults to 1), and the attribution `dimensions`. When governance is enabled for the account, the **governance chain enriches the check** — no separate "governance check" call exists, and the endpoint is **never** gated by governance enablement.

The response is `{ isGranted, type, accessDeniedReason, usageLimit, currentUsage, chains: [...] }`. Each `chains[][]` node carries `entityId`, `usageLimit`, `currentUsage`, and its own `isGranted` — to find the binding constraint on a denial, flatten `chains` and pick the first node with `isGranted: false`.

An over-limit governance denial can surface as `accessDeniedReason: "RequestedUsageExceedingLimit"` — not only `"BudgetExceeded"`. Both mean the same thing (`isGranted: false` with the binding node's `isGranted: false` in the chain); which string you get depends on whether the *requested* usage alone would exceed the limit vs. the budget being already spent. Don't branch behavior on the string — read the chain for the binding node.

### Prerequisite — depends on feature vs. credit

The prerequisite differs by capability type, matching this skill's Prerequisites in `SKILL.md`:

- **Feature check (`featureId`) — needs an entitling active subscription.** The governance chain is only consulted when the customer has an **ACTIVE subscription entitling the requested feature**. Without one, the check short-circuits before governance:
  - `accessDeniedReason: "NoActiveSubscription"` — no active subscription at all.
  - `accessDeniedReason: "NoFeatureEntitlementInSubscription"` — subscribed, but the plan doesn't include this feature.
- **Credit check (`currencyId`) — no subscription required.** Credits run on a different path: the check reads the customer's **credit pool** directly, and credits can be granted and consumed with **no active subscription or assigned plan**. A credit denial is `accessDeniedReason: "InsufficientCredits"` (the pool is empty), **not** a subscription error. Set the credit currency + funded pool up via `stigg-credits`.

(Also seen: `CustomerNotFound`, `BudgetExceeded`.) Provisioning the catalog and subscription is out of scope here — set it up via `stigg-pricing-modeling` and `stigg-subscriptions` first, then layer budgets on top.

### The ancestor rule

> An entity's `hasAccess` is **blocked when ANY ancestor has hit its hard limit.**

A leaf with a fresh budget of its own is still denied if any ancestor above it is exhausted. Corollaries:

- Debugging "this entity has budget but is denied" means walking **up** the tree, not inspecting the leaf.
- A parent's budget is a real ceiling on the whole subtree — children's limits can sum past it, and the parent still wins.

## Balances — the governance tree query

```ts
// Run via the MCP execute tool. Confirm the exact call shape via search_docs.
client.v1Beta.customers.retrieveGovernance(customerId, {
  // featureIds / currencyIds hydrate config + usage (see below); repeat for several.
  featureIds: ['<id>'], currencyIds: ['<id>'],
});
```

Returns the customer's governance **tree**: entity nodes with their **usage configuration** (the assignments) and **current usage**. Remaining budget per assignment = `usageLimit − currentUsage`.

### Tree-only vs. hydrated mode

By default the query is **tree-only**: each node comes back with `usageLimit`, `currentUsage`, `utilization`, and `cadence` set to `null` — you get the shape of the hierarchy but no budgets. To hydrate config + usage, pass the capabilities you want:

- `featureIds` — one or more metered-feature ids (repeat the param for several).
- `currencyIds` — one or more credit-currency ids.

With those, each row fills in `usageLimit` / `currentUsage` / `utilization` / `cadence` for the matching capability. Each row also carries `parentId` (reconstruct the tree client-side) and `scopeEntityIds` (`[]` = node-wide, non-empty = a dimension-scoped sub-budget).

### The lag contract

The tree's usage figures come from a **read model that may lag by minutes**. Consequences:

- **Never gate access from the tree.** Gating belongs to the entitlements check, which enforces in the access path.
- Do use the tree for: budget dashboards, burn-down alerts ("proj-atlas at 80%"), reconciliation, admin UIs.
- A freshly reported event not showing in the tree for a few minutes is expected behavior, not data loss.

## Putting it together — the runtime loop

```text
per request:
  1. entitlements check (isGranted, entity-attributed)  → allow / deny
  2. do the work
  3. ingest with attribution-key dimensions              → usage.report (calc/incremental)
                                                            or events.report (raw-events, by eventName)

out of band:
  governance tree query (with featureIds/currencyIds) → dashboards, alerts, budget reviews
```

## Common mistakes

Each mistake — the fix:

- Polling the tree per-request to "pre-check" budget — Use the entitlements check; the tree lags and isn't an enforcement surface.
- Tree query returns `null` limits/usage — You're in tree-only mode. Pass `featureIds`/`currencyIds` to hydrate config + usage.
- Always using one ingest endpoint for every feature — Fork by meter type: calc/incremental → `usage.report`; raw-events → `events.report`. On `CannotReportUsageForEntitlementWithMeterError` from `usage.report`, switch to reportEvent.
- Passing `featureId` to reportEvent — reportEvent matches the meter by `eventName` (+`idempotencyKey`), not `featureId`; attribution is in `dimensions`.
- Treating tree lag as an ingestion bug — Minutes of lag is the contract. Verify events in the Stigg app's Usage events view instead.
- Expecting a child to keep access after a parent exhausts — Any exhausted ancestor blocks the subtree.
- Summing children's limits to infer the parent's capacity — Limits are independent gates, not an allocation ledger.
