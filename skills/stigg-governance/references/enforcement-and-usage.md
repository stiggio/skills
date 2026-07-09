# Enforcement and Usage — Attribution, Access Checks, and the Governance Tree

The runtime half of governance: how usage lands on entities, how access is gated, and how balances are read. **Confirm response shapes via the Stigg MCP's `search_docs` before building against them.**

## Usage attribution — the standard events pipeline

Governance adds **no new ingestion path**. Usage flows through the same events pipeline as everything else (`POST /api/v1/events` / SDK `client.v1.events.report` — full mechanics in `stigg-entitlements/references/usage-reporting.md`). Attribution happens through **dimensions**:

1. The entity type declares up to 2 `attributionKeys` (e.g. `departmentId`, `agentId`).
2. Your events carry those keys in `dimensions`.
3. Stigg matches dimension values to entity IDs and attributes the usage to the entity **and all of its ancestors**.

```jsonc
// POST /api/v1/events
{
  "events": [{
    "customerId": "customer-123",
    "eventName": "tokens.consumed",
    "idempotencyKey": "req-abc-1234",
    "dimensions": { "departmentId": "dept-legal", "agentId": "agent-7" }
  }]
}
```

An event with no attribution-key dimensions attributes to no entity — customer-level metering still works, but governance budgets never see it.

## Access — the entitlements check

Gate requests with the standard `hasAccess`-style entitlements check (`client.v1Beta.customers.entitlements.check` / `/entitlements/check`). When governance is enabled for the account, the **governance chain enriches the check** — no separate "governance check" call exists, and the endpoint is **never** gated by the enablement flag.

### The ancestor rule

> An entity's `hasAccess` is **blocked when ANY ancestor has hit its hard limit.**

`agent-7` with a fresh personal budget is still denied if `team-ip` or `dept-legal` is exhausted. Corollaries:

- Debugging "this entity has budget but is denied" means walking **up** the tree, not inspecting the leaf.
- A parent's budget is a real ceiling on the whole subtree — children's limits can sum past it, and the parent still wins.

## Balances — the governance tree query

```text
GET /api/v1-beta/customers/{customerId}/governance      (X-API-KEY)
SDK: client.v1.events.beta.customers.retrieveGovernance(customerId)
     — yes, under v1.events.beta, not v1Beta; known naming inconsistency.
```

Returns the customer's governance **tree**: entity nodes with their **usage configuration** (the assignments) and **current usage**. Remaining budget per assignment = `usageLimit − currentUsage`.

### The lag contract

The tree's usage figures come from a **read model that may lag by minutes**. Consequences:

- **Never gate access from the tree.** Gating belongs to the entitlements check, which enforces in the access path.
- Do use the tree for: budget dashboards, burn-down alerts ("dept-legal at 80%"), reconciliation, admin UIs.
- A freshly reported event not showing in the tree for a few minutes is expected behavior, not data loss.

## Putting it together — the runtime loop

```text
per request:
  1. entitlements check (hasAccess, entity-attributed)  → allow / deny
  2. do the work
  3. report event with attribution-key dimensions       → async, buffered

out of band:
  governance tree query → dashboards, alerts, budget reviews
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Polling the tree per-request to "pre-check" budget | Use the entitlements check; the tree lags and isn't an enforcement surface. |
| Treating tree lag as an ingestion bug | Minutes of lag is the contract. Verify events in the Stigg app's Usage events view instead. |
| Expecting a child to keep access after a parent exhausts | Any exhausted ancestor blocks the subtree. |
| Summing children's limits to infer the parent's capacity | Limits are independent gates, not an allocation ledger. |
| Looking for `retrieveGovernance` under `client.v1Beta.customers` | It lives at `client.v1.events.beta.customers.retrieveGovernance(id)`. |
