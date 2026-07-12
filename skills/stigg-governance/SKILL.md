---
name: stigg-governance
description: Use for per-entity budget and spend-control work in Stigg Governance — defining entity types with attribution keys, building a customer's entity hierarchy (departments / teams / agents / cost centers), assigning usage limits per entity for a feature or credit currency, dimensional scoping, querying the governance tree for balances, and enforcing budgets at runtime. Triggers include "per-department budget", "per-team spend limit", "per-agent token cap", "entity type", "entity hierarchy", "attribution keys", "governance tree", "usage limit per entity", "budget enforcement", "scoped assignment", "GovernanceNotEnabled", "cost center", "sub-customer limits", "keep entities in sync", "re-parent entity", "archive entity", "CannotReportUsageForEntitlementWithMeterError". Skip for account-level credit grants / pools / ledger (stigg-credits), feature gating with no entity hierarchy (stigg-entitlements), or catalog modeling (stigg-pricing-modeling).
---

# Stigg Governance — Per-Entity Budget Enforcement

Governance layers **per-entity budget enforcement** on top of Stigg credits and features. Where credits answer "how much can this *customer* consume?", governance answers "how much can this customer's *department / team / agent / cost center* consume?" — with hierarchical roll-up and hard-limit enforcement. The surface lives under **REST `/api/v1-beta`** (header `X-API-KEY`).

**Base URLs.** Prod Core API `https://api.stigg.io`; edge (low-latency read) `https://edge.api.stigg.io`; staging `https://api-staging.stigg.io`. Governance calls use the Core host with your server-side key in `X-API-KEY`.

## Step 0 — Preflight: Is Governance Enabled? (do this FIRST)

Governance is **account-gated**. Before any governance work, hit one governance endpoint (the tree query is a safe read) and check the response:

- **HTTP 403** whose error `code` is **`GovernanceNotEnabled`** — key on the code, not the message. The body looks like this (message is illustrative and i18n-fragile, so don't match on it):

  ```json
  {"code":"GovernanceNotEnabled","message":"Governance is not enabled for this account. Please contact Stigg to enable Governance."}
  ```

  → **Stop.** Tell the user to contact Stigg to enable Governance for their account. Do **not** retry, do **not** try alternate endpoints or keys, and do **not** guess at workarounds — every governance endpoint is gated by the same flag.

- **HTTP 200 with empty data** → governance is **enabled but not configured yet**. Proceed to model entity types, entities, and assignments.

The gate fires **before customer resolution**, so a **placeholder/fake customer id is a valid preflight probe** — you don't need to provision a real customer first; `GET /governance` on any id returns the 403 (or the empty 200) purely off the account flag.

To confirm **"not enabled ≠ bad key,"** pair the probe with a non-governance read like `customers.list`: if that succeeds but the governance call 403s, it's the feature flag, not auth (a wrong key 401s everywhere).

`/entitlements/check` is **never** gated — entitlement checks keep working regardless of governance enablement.

## Prerequisites — an entitling subscription

Governance budgets sit *on top of* an existing entitlement, they don't replace it. The gating check only reaches the governance chain when the customer has an **ACTIVE subscription that entitles the feature/credit** — otherwise it short-circuits with a standard denial (`NoActiveSubscription`; `NoFeatureEntitlementInSubscription` = subscribed but the plan lacks the feature) and never consults the tree. This is a **permanent ordering requirement** from the Stigg 1.0 ↔ 2.0 coupling, not a beta gap: **entitle the feature via an active subscription BEFORE checks can pass.** Assignment upsert does **no** entitlement validation — it accepts a budget for an un-entitled feature and the gap only surfaces later at check time. Order it: catalog + subscription first, then budgets.

Setting up that catalog (plan, feature, subscription — and note a FREE plan needs `charges.pricingType` before it can be published) is **not** governance's job. Do it via **`stigg-pricing-modeling`** (catalog) and **`stigg-subscriptions`** (provisioning), then come back to add per-entity budgets.

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

## Entity Types

An **entity type** declares a kind of sub-customer unit ("department", "team", "agent") and — critically — its **attribution keys**: up to **2** event-dimension keys whose values identify which entity a usage event belongs to. Operations: **list** and **upsert** (bulk `PUT`, idempotent). Design attribution keys before reporting any usage; retro-attributing old events is not a thing.

## Entities

**Entities** are the per-customer instances of those types — up to **4 levels** deep (e.g. org → department → team → agent). The record is flat (`id`, `entityTypeId`, `metadata`); **hierarchy is `parentId` on the *assignment*, not the entity** — the entity upsert rejects `parentId`, and `metadata` (free-form, no structural role) is **not** a place to model a parent. Operations: **list / get / upsert (bulk `PUT`) / archive / unarchive**. **Re-parenting is leaf-only** (a non-leaf move is rejected); **archive is NOT leaf-gated** — archiving a parent orphans its still-active children, so archive/re-home descendants bottom-up yourself. Full rules: `references/entity-model.md`.

## Keeping Entities in Sync

Entities are an **ongoing mirror of the vendor's source of truth**, not a one-time import — the hierarchy lives in your system first; the tree is a projection. Wire governance ops into the code paths that mutate your model: **register → entity upsert; rename/attribute-change → idempotent re-upsert; deprovision/delete → archive (bottom-up); restructure → re-upsert the assignment's `parentId` (leaf-only)**. The model is arbitrary — user/team/workspace/org are illustrative only. Since upsert is idempotent bulk `PUT`, a declarative "push the desired tree each sync" pattern works. Full mapping + the unshipped allocation-return caveat: `references/entity-lifecycle.md`.

## Assignments — the Budgets

An **assignment** sets a limit for one `(entity, capability)` pair:

| Field | Meaning |
|---|---|
| `entityId` | The entity being budgeted (any tree level — org / department / team / agent). |
| `featureId` **or** `currencyId` | The capability being limited — exactly one: a **`featureId`** (metered feature) **or** a **`currencyId`** (credit currency). Mutually exclusive; there is no `capability` field on the wire. |
| `usageLimit` | The budget for the cadence window. |
| `cadence` | ISO-8601 duration, e.g. `'P1M'` (monthly), `'P1D'` (daily). |
| `scopeEntityIds` | Optional — restricts the limit to usage attributed to specific (typically dimensional) entities, instead of everything under the assignee. |

Operations: **list / upsert**. A `currencyId` capability budgets *credit consumption* — unrelated to billing currency (USD/EUR). Worked examples: `references/assignments.md`.

## Reporting Usage — the Meter-Type Fork

Ingest never gates; it records consumption and rolls up the tree. **The endpoint depends on the target feature's meter type:**

- **Calculated / incremental meters** (your app computes the number) → **`POST /api/v1/usage`** (`client.v1.usage.report`): synchronous, echoes `currentUsage`. Required: `customerId`, `featureId` (feature OR credit id), `value`.
- **Raw-events meters** (Stigg aggregates raw events — count/sum/unique) → **`POST /api/v1/events`** (`client.v1.events.report`, reportEvent): the **required** path for these today (usage-reporting events isn't supported); high-volume, eventually-consistent. reportEvent has **no `featureId`** — the meter is matched by **`eventName`** (+`idempotencyKey`).

Both carry attribution in **`dimensions`** (the entity type's attributionKey values, e.g. `{ "departmentId": "dept-legal", "agentId": "agent-7" }`) → attributes to the matching entity **and every ancestor**.

> **Meter type isn't reliably readable via the API** (the `meterType` enum has no "raw events" value; event meters live in a separate `meter` config). So **try then fall back**: call `usage.report`; on **`CannotReportUsageForEntitlementWithMeterError`** it's a raw-events meter — switch to `events.report`. Shapes + examples: `references/enforcement-and-usage.md`.

## Enforcement and Balances

Two distinct reads — do not mix them up:

1. **Access (gates requests):** the standard entitlements **check** — `GET /api/v1-beta/customers/{customerId}/entitlements/check` (SDK `client.v1Beta.customers.entitlements.check`), passing `featureId` **or** `currencyId`, optional `requestedUsage` (default 1), and attribution `dimensions`. When governance is enabled, the governance chain enriches the check. **An entity is blocked when ANY ancestor has hit its hard limit** — a team with budget left is still blocked if its department is exhausted. Requires an entitling active subscription (see Prerequisites).
2. **Balances (observability):** the **governance tree query** — `GET /api/v1-beta/customers/{customerId}/governance` — returns the entity nodes; remaining budget = `usageLimit − currentUsage`. **By default it runs tree-only: `usageLimit` / `currentUsage` / `utilization` / `cadence` come back `null`;** pass `featureIds` and/or `currencyIds` query params (repeat for multiple) to hydrate config + usage. The read model **may lag by minutes** — use it for dashboards and alerts, **never** to gate access. Full semantics: `references/enforcement-and-usage.md`.

## Surface Map

| Operation | REST (`/api/v1-beta`, `X-API-KEY`) | `@stigg/typescript` SDK (beta) |
|---|---|---|
| Entity types — list / upsert (PUT bulk) | entity-type routes | `client.v1Beta.entityTypes.*` |
| Entities — list / get / upsert (PUT bulk) / archive / unarchive | customer entity routes | `client.v1Beta.customers.entities.*` |
| Assignments — list / upsert | customer assignment routes | `client.v1Beta.customers.assignments.*` |
| Entitlements check | `GET /api/v1-beta/customers/{id}/entitlements/check` (never gated) | `client.v1Beta.customers.entitlements.check` |
| Governance tree query | `GET /api/v1-beta/customers/{id}/governance?featureIds=…&currencyIds=…` | `client.v1.events.beta.customers.retrieveGovernance(id)` |
| Ingest — calculated/incremental meter | `POST /api/v1/usage` | `client.v1.usage.report` |
| Ingest — raw-events meter (by `eventName`) | `POST /api/v1/events` | `client.v1.events.report` |

Wart to expect: **the tree query lives under `client.v1.events.beta.customers`**, not `client.v1Beta.customers` like its siblings — a known naming inconsistency. Don't "fix" it by guessing a `v1Beta` equivalent. Confirm exact routes via `search_docs` first.

**API vs UI — different assignment models, by design.** The API is the programmatic source of truth and targets **one capability per assignment** (`(entity, featureId|currencyId)`). The dashboard UI intentionally **auto-creates an unlimited assignment across ALL capabilities** per entity (you edit limits per row). So UI-created and API-created entities look different — don't be surprised switching between them, and don't "repair" one to match the other.

**Via the Stigg MCP:** the MCP exposes only **`execute`** (sandboxed TypeScript over the SDK) and **`search_docs`** — there is **no** one-tool-per-endpoint mapping. Governance ops go through `execute` using the SDK namespaces above.

### Fallback — when the MCP can't serve your environment

The MCP backend binds to an environment via the server key. If it can't serve your key's environment, MCP calls fail — a **401** on `execute` (or `search_docs`), or a client-init error like *"STIGG_API_KEY … is missing or empty."* Not transient, not a reason to stop. Instead:

1. Take the request shapes from `search_docs` (or the Mintlify Stigg docs) — reading docs needs no working execute backend.
2. Call the **REST API directly** with your `X-API-KEY`: `/api/v1` for usage/events, `/api/v1-beta` for the governance surface. Shapes match what the SDK sends.

A **401 on `execute` while `search_docs` still works** usually means the key is a **staging key** — the MCP execute backend targets prod. If a direct REST call to the prod Core host (`api.stigg.io`) also 401s, retry against **`api-staging.stigg.io`**. The REST path is always available with the server key.

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
| Always calling `POST /api/v1/usage`, or always `POST /api/v1/events` | It's a meter-type fork. Calculated/incremental → `usage.report`; raw-events → `events.report` (reportEvent, by `eventName`). If `usage.report` returns `CannotReportUsageForEntitlementWithMeterError`, switch to reportEvent. |
| Passing a `featureId` to reportEvent | reportEvent targets the meter by `eventName` (+`idempotencyKey`), not `featureId`; attribution rides in `dimensions`. |
| Creating an assignment and expecting checks to pass | Assignment upsert does no entitlement validation. The feature must be entitled via an active subscription first (permanent 1.0↔2.0 ordering), else check denies with `NoActiveSubscription`/`NoFeatureEntitlementInSubscription`. |
| Reading the tree and getting `null` limits/usage | Tree-only mode. Pass `featureIds`/`currencyIds` to hydrate config + usage. |
| Assuming a 401 / missing-key MCP error means "stop" | The MCP backend may not serve your key's environment (often a staging key). Take shapes from `search_docs`/docs and call REST directly with `X-API-KEY`. |
| Re-parenting a node that still has children | Re-parent is leaf-only — a non-leaf move is rejected (surfaces as an opaque 500, not transient). Move/archive children first, then move the former parent. |
| Assuming archive is leaves-only | It isn't — archiving a parent orphans its active children. Archive/re-home descendants bottom-up yourself; the platform won't stop you. |
| Modeling an entity's parent via `metadata` | `metadata` is free-form and structural-role-free. Hierarchy is `parentId` on the assignment only. |
| Treating entities as a one-time import | They're an ongoing mirror of your source of truth — wire upsert/archive/re-parent into your create/update/delete/restructure paths (`references/entity-lifecycle.md`). |
| Reading `currencyId` capability as billing currency | It's a credit currency (see `stigg-credits`), not USD/EUR. |
| Expecting per-endpoint MCP tools for governance | The MCP has only `execute` + `search_docs`. Drive the SDK through `execute`. |
