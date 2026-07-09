# Enforcement and Usage — Attribution, Access Checks, and the Governance Tree

The runtime half of governance: how usage lands on entities, how access is gated, and how balances are read. **Confirm response shapes via the Stigg MCP's `search_docs` before building against them.**

## Usage attribution — report usage, not raw events

The governance ingest path is **`POST /api/v1/usage`** (SDK `client.v1.usage.report`): a **synchronous, meter-free** report that increments the counter and rolls up the tree immediately. The response echoes the new `currentUsage`. Attribution happens through **dimensions**:

1. The entity type declares up to 2 `attributionKeys` (e.g. `departmentId`, `agentId`).
2. Your usage report carries those keys' values in `dimensions`.
3. Stigg matches dimension values to entity IDs and attributes the usage to the entity **and all of its ancestors**.

```jsonc
// POST /api/v1/usage — primary path; required fields: customerId, featureId, value
{
  "usages": [{
    "customerId": "customer-123",
    "featureId": "feature-ai-tokens",
    "value": 1250,
    "dimensions": { "departmentId": "dept-legal", "agentId": "agent-7" }
  }]
}
// → 201; data[].currentUsage reflects the rolled-up total (ancestors included)
```

A usage report with no attribution-key dimensions attributes to no entity — customer-level metering still works, but governance budgets never see it. `featureId` accepts either a metered feature id **or** a credit currency id.

### Raw events — secondary, and only with a meter

`POST /api/v1/events` (`client.v1.events.report`, requires `eventName` + `idempotencyKey`) is the **high-volume, eventually-consistent** path (~seconds of lag). It's meaningful for governance **only when the target feature has a correctly configured event meter** — that meter is what turns raw events into feature usage. **With no meter, events are accepted (`201`) but silently never accrue** to any governance budget, and no error tells you. Prefer `POST /api/v1/usage` unless you specifically need high-throughput metering *and* have set up the meter. Configuring event meters is a catalog concern — see `stigg-pricing-modeling` / `stigg-entitlements`.

## Access — the entitlements check

Gate requests with the standard entitlements check — `GET /api/v1-beta/customers/{customerId}/entitlements/check` (SDK `client.v1Beta.customers.entitlements.check`). Pass `featureId` **or** `currencyId` (not both), an optional `requestedUsage` (defaults to 1), and the attribution `dimensions`. When governance is enabled for the account, the **governance chain enriches the check** — no separate "governance check" call exists, and the endpoint is **never** gated by the enablement flag.

The response is `{ isGranted, type, accessDeniedReason, usageLimit, currentUsage, chains: [...] }`. Each `chains[][]` node carries `entityId`, `usageLimit`, `currentUsage`, and its own `isGranted` — to find the binding constraint on a denial, flatten `chains` and pick the first node with `isGranted: false`.

### Prerequisite — an entitling active subscription

The governance chain is only consulted when the customer already has an **ACTIVE subscription entitling the requested feature/credit**. Without one, the check short-circuits before governance:

- `accessDeniedReason: "NoActiveSubscription"` — no active subscription at all.
- `accessDeniedReason: "NoFeatureEntitlementInSubscription"` — subscribed, but the plan doesn't include this feature.

(Also seen: `CustomerNotFound`, `BudgetExceeded`, `InsufficientCredits`.) Provisioning the catalog and subscription is out of scope here — set it up via `stigg-pricing-modeling` and `stigg-subscriptions` first, then layer budgets on top.

### The ancestor rule

> An entity's `hasAccess` is **blocked when ANY ancestor has hit its hard limit.**

`agent-7` with a fresh personal budget is still denied if `team-ip` or `dept-legal` is exhausted. Corollaries:

- Debugging "this entity has budget but is denied" means walking **up** the tree, not inspecting the leaf.
- A parent's budget is a real ceiling on the whole subtree — children's limits can sum past it, and the parent still wins.

## Balances — the governance tree query

```text
GET /api/v1-beta/customers/{customerId}/governance      (X-API-KEY)
    ?featureIds=<id>&featureIds=<id>&currencyIds=<id>    (see below)
SDK: client.v1.events.beta.customers.retrieveGovernance(customerId)
     — yes, under v1.events.beta, not v1Beta; known naming inconsistency.
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
- Do use the tree for: budget dashboards, burn-down alerts ("dept-legal at 80%"), reconciliation, admin UIs.
- A freshly reported event not showing in the tree for a few minutes is expected behavior, not data loss.

## Putting it together — the runtime loop

```text
per request:
  1. entitlements check (isGranted, entity-attributed)  → allow / deny
  2. do the work
  3. report usage with attribution-key dimensions        → POST /api/v1/usage, synchronous roll-up

out of band:
  governance tree query (with featureIds/currencyIds) → dashboards, alerts, budget reviews
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Polling the tree per-request to "pre-check" budget | Use the entitlements check; the tree lags and isn't an enforcement surface. |
| Tree query returns `null` limits/usage | You're in tree-only mode. Pass `featureIds`/`currencyIds` to hydrate config + usage. |
| Feeding governance via `POST /api/v1/events` without a meter | Events need a configured event meter or they never accrue. Report via `POST /api/v1/usage`. |
| Treating tree lag as an ingestion bug | Minutes of lag is the contract. Verify events in the Stigg app's Usage events view instead. |
| Expecting a child to keep access after a parent exhausts | Any exhausted ancestor blocks the subtree. |
| Summing children's limits to infer the parent's capacity | Limits are independent gates, not an allocation ledger. |
| Looking for `retrieveGovernance` under `client.v1Beta.customers` | It lives at `client.v1.events.beta.customers.retrieveGovernance(id)`. |
