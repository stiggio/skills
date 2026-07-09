---
name: stigg-governance
description: Use for per-entity budget and spend-control work in Stigg Governance — defining entity types with attribution keys, building a customer's entity hierarchy (departments / teams / agents / cost centers), assigning usage limits per entity for a feature or credit currency, dimensional scoping, querying the governance tree for balances, and enforcing budgets at runtime. Triggers include "per-department budget", "per-team spend limit", "per-agent token cap", "entity type", "entity hierarchy", "attribution keys", "governance tree", "usage limit per entity", "budget enforcement", "scoped assignment", "GovernanceNotEnabled", "cost center", "sub-customer limits". Skip for account-level credit grants / pools / ledger (stigg-credits), feature gating with no entity hierarchy (stigg-entitlements), or catalog modeling (stigg-pricing-modeling).
---

# Stigg Governance — Per-Entity Budget Enforcement

Governance layers **per-entity budget enforcement** on top of Stigg's credits and features. Where credits answer "how much can this *customer* consume?", governance answers "how much can this customer's *department / team / agent / cost center* consume?" — with hierarchical roll-up and hard-limit enforcement. The surface lives under **REST `/api/v1-beta`** (header `X-API-KEY`).

**Base URLs.** Production Core API is `https://api.stigg.io`; the edge (low-latency read) host is `https://edge.api.stigg.io`; staging is `https://api-staging.stigg.io`. All governance calls use the Core host with your server-side key in `X-API-KEY`.

## Step 0 — Preflight: Is Governance Enabled? (do this FIRST)

Governance is **account-gated**. Before any governance work, hit one governance endpoint (the tree query is a safe read) and check the response:

- **HTTP 403** with body exactly:

  ```json
  {"code":"GovernanceNotEnabled","message":"Governance is not enabled for this account. Please contact Stigg to enable Governance."}
  ```

  → **Stop.** Tell the user to contact Stigg to enable Governance for their account. Do **not** retry, do **not** try alternate endpoints or keys, and do **not** guess at workarounds — every governance endpoint is gated by the same flag.

- **HTTP 200 with empty data** → governance is **enabled but not configured yet**. Proceed to model entity types, entities, and assignments.

`/entitlements/check` is **never** gated — entitlement checks keep working regardless of governance enablement.

## Prerequisites — an entitling subscription

Governance budgets sit *on top of* an existing entitlement, they don't replace it. The gating check only reaches the governance chain when the customer has an **ACTIVE subscription that entitles the feature/credit** — otherwise it short-circuits with a standard denial and never consults the tree. The two you'll see most:

- `NoActiveSubscription` — the customer has no active subscription at all.
- `NoFeatureEntitlementInSubscription` — the customer is subscribed, but the plan doesn't include this feature.

Setting up that catalog (plan, feature, subscription — and note a FREE plan needs `charges.pricingType` before it can be published) is **not** governance's job. Do it via **`stigg-pricing-modeling`** (catalog) and **`stigg-subscriptions`** (provisioning), then come back here to add per-entity budgets.

## Before You Start

Per the umbrella `stigg` skill: **search first.** Governance is a v1-beta surface and evolves quickly — confirm endpoint paths, field names, and body shapes via the Stigg MCP's `search_docs` (or the Mintlify Stigg docs MCP) before authoring. This skill owns the mental model; the live surface owns the shapes.

## Mental Model — Four Pieces

```text
┌─────────────────────────────────────────────┐
│  Entity type   (per environment)            │
│  "department", "agent" — with               │
│  attributionKeys (max 2) that map event     │
│  dimensions → entities                      │
└───────────────────┬─────────────────────────┘
                    │ instantiated per customer as
                    ▼
┌─────────────────────────────────────────────┐
│  Entities      (per customer)               │
│  flat records; hierarchy set via parentId   │
│  on the ASSIGNMENT — up to 4 levels         │
│  e.g. org → department → team → agent       │
└───────────────────┬─────────────────────────┘
                    │ budgeted by
                    ▼
┌─────────────────────────────────────────────┐
│  Assignments   = limit per (entity,         │
│  capability), capability = featureId OR     │
│  currencyId; usageLimit + cadence (ISO-8601,│
│  e.g. 'P1M') + optional scopeEntityIds      │
└───────────────────┬─────────────────────────┘
                    │ observed via
                    ▼
┌─────────────────────────────────────────────┐
│  Governance tree  (read model)              │
│  nodes + usage config + current usage —     │
│  may lag by minutes; NEVER gates access     │
└─────────────────────────────────────────────┘
```

Deep dives: `references/entity-model.md`, `references/assignments.md`, `references/enforcement-and-usage.md`.

## Entity Types

An **entity type** declares a kind of sub-customer unit ("department", "team", "agent") and — critically — its **attribution keys**: up to **2** event-dimension keys whose values identify which entity a usage event belongs to. Operations: **list** and **upsert** (bulk `PUT` — idempotent create-or-update). Design attribution keys before reporting any usage; retro-attributing old events is not a thing.

## Entities

**Entities** are the per-customer instances of those types — up to **4 levels** deep (e.g. org → department → team → agent). The entity record itself is flat (`id`, `entityTypeId`, `metadata`); **hierarchy is expressed by `parentId` on the *assignment*, not on the entity** — the entity upsert rejects `parentId`. Operations: **list / get / upsert (bulk `PUT`) / archive / unarchive** — **archive applies to leaves only**; re-parent or archive children first. Full rules: `references/entity-model.md`.

## Assignments — the Budgets

An **assignment** sets a limit for one `(entity, capability)` pair:

| Field | Meaning |
|---|---|
| `capability` | What's being limited — a **`featureId`** (metered feature) **or** a **`currencyId`** (credit currency). |
| `usageLimit` | The budget for the cadence window. |
| `cadence` | ISO-8601 duration, e.g. `'P1M'` (monthly), `'P1D'` (daily). |
| `scopeEntityIds` | Optional — restricts the limit to usage attributed to specific (typically high-cardinality / dimensional) entities, instead of everything under the assignee. |

Operations: **list / upsert**. A `currencyId` capability budgets *credit consumption* — it is unrelated to billing currency (USD/EUR). Details and worked examples: `references/assignments.md`.

## Reporting Usage — Report Usage, not Raw Events

The **primary governance ingest path is `POST /api/v1/usage`** (SDK `client.v1.usage.report`) — a **synchronous, meter-free** report that increments the counter and rolls up the entity tree immediately (the response echoes `currentUsage`). This is what governance budgets accrue against. Attribution is carried in **`dimensions`**: put the entity type's **attributionKey values** there and Stigg attributes the usage to the matching entity **and every ancestor** up the tree.

```jsonc
// POST /api/v1/usage — the primary, synchronous, meter-free path
{
  "usages": [{
    "customerId": "customer-123",
    "featureId": "feature-ai-tokens",     // required — the feature OR credit id
    "value": 1250,                          // required — non-negative amount to add
    "dimensions": { "departmentId": "dept-legal", "agentId": "agent-7" }
  }]
}
// → 201, data[].currentUsage reflects the new rolled-up total
```

Up to 100 usages per request.

> **Do NOT default to `POST /api/v1/events` for governance.** Raw events (`client.v1.events.report`, requires `eventName` + `idempotencyKey`) are a **secondary, high-volume, eventually-consistent** path (~seconds of lag). Critically, raw events only aggregate into a feature's usage when that feature has a **correctly configured event meter** behind it — with no meter, the events are accepted but **silently never accrue** to any governance budget. Reach for events only for high-throughput metering where you've already set up the meter and don't need a real-time balance; otherwise report usage.

## Enforcement and Balances

Two distinct reads — do not mix them up:

1. **Access (gates requests):** the standard entitlements **check** — `GET /api/v1-beta/customers/{customerId}/entitlements/check` (SDK `client.v1Beta.customers.entitlements.check`), passing `featureId` **or** `currencyId`, optional `requestedUsage` (default 1), and the attribution `dimensions`. It returns `isGranted` plus an `accessDeniedReason` and per-entity `chains`. When governance is enabled, the governance chain enriches the check. **An entity is blocked when ANY ancestor has hit its hard limit** — a team with budget left is still blocked if its department is exhausted. Requires an entitling active subscription first (see Prerequisites).
2. **Balances (observability):** the **governance tree query** — `GET /api/v1-beta/customers/{customerId}/governance` — returns the entity nodes; remaining budget = `usageLimit − currentUsage`. **By default it runs in tree-only mode: `usageLimit`, `currentUsage`, `utilization`, and `cadence` come back `null`.** To get the budget config and usage, pass `featureIds` and/or `currencyIds` query params (repeat the param for multiple, e.g. `?featureIds=feature-ai-tokens&featureIds=feature-api-calls`). The usage read model **may lag by minutes** — use it for dashboards and alerts, **never** to gate access.

Full enforcement semantics: `references/enforcement-and-usage.md`.

## Surface Map

| Operation | REST (`/api/v1-beta`, `X-API-KEY`) | `@stigg/typescript` SDK (beta) |
|---|---|---|
| Entity types — list / upsert (PUT bulk) | entity-type routes | `client.v1Beta.entityTypes.*` |
| Entities — list / get / upsert (PUT bulk) / archive / unarchive | customer entity routes | `client.v1Beta.customers.entities.*` |
| Assignments — list / upsert | customer assignment routes | `client.v1Beta.customers.assignments.*` |
| Entitlements check | `GET /api/v1-beta/customers/{id}/entitlements/check` (never gated) | `client.v1Beta.customers.entitlements.check` |
| Governance tree query | `GET /api/v1-beta/customers/{id}/governance?featureIds=…&currencyIds=…` | `client.v1.events.beta.customers.retrieveGovernance(id)` |
| Usage reporting (primary) | `POST /api/v1/usage` | `client.v1.usage.report` |
| Raw events (secondary, meter required) | `POST /api/v1/events` | `client.v1.events.report` |

Two warts to expect:

- **The tree query lives under `client.v1.events.beta.customers`**, not `client.v1Beta.customers` like its siblings — a known naming inconsistency. Don't "fix" it by guessing a `v1Beta` equivalent. Without `featureIds`/`currencyIds` it returns nodes only (config/usage `null`).
- Confirm exact route paths and body shapes via `search_docs` before calling — the v1-beta surface moves.

**Via the Stigg MCP:** the MCP exposes only **`execute`** (sandboxed TypeScript over the SDK) and **`search_docs`** — there is **no** one-tool-per-endpoint mapping. Governance ops through the MCP go through `execute` using the SDK namespaces above.

### Fallback — when the MCP can't serve your environment

The MCP backend binds to a specific environment via the server key. If it isn't provisioned for the environment your key targets, MCP calls fail — you may see a **401** on `execute` (or `search_docs`), or a client-init error like *"STIGG_API_KEY … is missing or empty."* This is not a transient error and not a reason to stop. Do this instead:

1. Take the request shapes from `search_docs` (or the Mintlify Stigg docs) — reading docs does not require a working execute backend.
2. Call the **REST API directly** with your `X-API-KEY`: `/api/v1` for usage/events, `/api/v1-beta` for the governance surface (entity types, entities, assignments, tree, check). The shapes are identical to what the SDK sends.

Reserve MCP `execute` for when it works; the REST path is always available with the server key.

## When NOT to Use This Skill

- Account-level credit currencies, grants, pools, ledger, auto-recharge → `stigg-credits`.
- Feature gating / usage reporting with no per-entity budgets → `stigg-entitlements`.
- Modeling features / plans / credit currencies in the catalog → `stigg-pricing-modeling`.
- Provisioning / canceling subscriptions → `stigg-subscriptions`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Retrying after `GovernanceNotEnabled` | It's an account flag, not a transient error. Tell the user to contact Stigg; stop. |
| Treating an empty 200 as "not enabled" | Empty = enabled but unconfigured. Proceed to model. |
| Gating access from the tree query | The tree is a read model that may lag by minutes. Gate with the entitlements check only. |
| Debugging "team has budget but is blocked" as a bug | Ancestor semantics: any exhausted ancestor blocks the whole subtree. Check the tree upward. |
| Defining more than 2 attribution keys, or >4 hierarchy levels | Hard limits. Redesign the model (e.g. fold a level into `scopeEntityIds`). |
| Reporting usage without attribution-key dimensions | Usage won't attribute to any entity — budgets never accrue. Design dimensions up front. |
| Reaching for `POST /api/v1/events` to feed governance | Events need a configured event meter or they silently never accrue. Use `POST /api/v1/usage` (synchronous, meter-free) unless you specifically need high-volume metering and have set up the meter. |
| Reading the tree and getting `null` limits/usage | Tree-only mode. Pass `featureIds`/`currencyIds` to hydrate config + usage. |
| Assuming a 401 / missing-key MCP error means "stop" | The MCP backend may not serve your key's environment. Take shapes from `search_docs`/docs and call REST directly with `X-API-KEY`. |
| Archiving an entity that still has children | Archive is leaves-only. Re-parent or archive children first. |
| Reading `currencyId` capability as billing currency | It's a credit currency (see `stigg-credits`), not USD/EUR. |
| Expecting per-endpoint MCP tools for governance | The MCP has only `execute` + `search_docs`. Drive the SDK through `execute`. |
| Hardcoding v1-beta routes / body shapes from memory | Beta surface. `search_docs` first, per the umbrella rule. |
