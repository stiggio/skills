---
name: stigg-governance
description: Use for per-entity budget and spend-control work in Stigg Governance — enforcing usage limits over whatever hierarchy of sub-units a customer needs to budget (the entity model is customer-defined, not fixed) — defining entity types with attribution keys, building the customer's own tree, assigning limits per entity for a feature or credit currency, dimensional scoping, querying the governance tree for balances, and runtime enforcement. Triggers include "per-entity budget", "per-team spend limit", "per-agent token cap", "cost center limits", "sub-customer limits", "per-project budget", "entity type", "attribution keys", "governance tree", "budget enforcement", "scoped assignment", "GovernanceNotEnabled", "keep entities in sync", "re-parent entity", "CannotReportUsageForEntitlementWithMeterError". Skip for account-level credit grants / pools / ledger (stigg-credits), feature gating with no entity hierarchy (stigg-entitlements), or catalog modeling (stigg-pricing-modeling).
---

# Stigg Governance — Per-Entity Budget Enforcement

Governance layers **per-entity budget enforcement** on top of Stigg credits and features. Where credits answer "how much can this *customer* consume?", governance answers "how much can this customer's *own sub-units* consume?" — whatever the vendor needs to budget, with hierarchical roll-up and hard-limit enforcement. **The entity model is arbitrary and customer-defined** — Stigg imposes no fixed hierarchy, so don't assume one (see Step 1).

All Stigg API actions in this skill run **through the Stigg MCP** — the `execute` tool runs SDK code, `search_docs` resolves exact shapes. Governance operations use the SDK's beta namespace (e.g. `client.v1Beta.…`). This skill owns the *mental model*; it does not cover connection, keys, or setup — those belong to the **`stigg-mcp`** skill.

> **Private-beta surface.** Governance is currently a **private-beta** part of the Stigg API. Its shape may change at GA, so **re-check exact routes and field names via `search_docs`** rather than treating anything here as permanent.

## Step 0 — Preflight: Is Governance Enabled? (do this FIRST)

Run ONE probe via the MCP `execute` tool: `client.v1Beta.entityTypes.list()` (no customer id needed).

- Fails with error `code` **`GovernanceNotEnabled`** → **STOP.** Tell the user to contact Stigg to enable Governance. Do not retry or work around it.
- Succeeds → proceed to Step 1. An empty list is expected — nothing is configured yet.

## Step 1 — Elicit the Model: ASK, Don't Assume (do this BEFORE defining entity types)

**The governance entity model is fully arbitrary and customer-defined.** There is **no default hierarchy** — Stigg ships none, and any dept/team/agent shape is just *one* possibility, not a template. Before defining a single entity type, **ask the customer what they want to govern** and build strictly from their answer. Do **not** propose or assume a hierarchy. Ask:

- **What units do you budget?** (their own vocabulary — teams, projects, regions, tenants, sub-accounts, cost centers, agents, workspaces…)
- **How do they nest / roll up?** (which contains which, and how deep — the tree caps at 4 levels)
- **Which levels actually carry budgets?** (only levels that hold a limit or need roll-up belong in the tree; fold the rest away)
- **What identifies a unit on a usage event?** (the dimension key(s) your events already emit — these become the attribution keys, max 2 per type)

Model from their answer: "we budget by project, and projects belong to a client" → build `client → project`. Every worked example below and in the references is *one possible model* — it teaches the API's *shape*, never the model to adopt.

**Sequence:** preflight (Step 0) → elicit the model (this step) → entity types → entities → assignments → runtime.

## Prerequisites — an entitling subscription

Governance budgets sit *on top of* an existing entitlement; they don't replace it. The gating check only reaches the governance chain when the customer has an **ACTIVE subscription that entitles the feature/credit** — otherwise it short-circuits with a standard denial (`NoActiveSubscription`; `NoFeatureEntitlementInSubscription` = subscribed but the plan lacks the feature) and never consults the tree. This is a **permanent ordering requirement** from the Stigg 1.0 ↔ 2.0 coupling, not a beta gap: **entitle the feature via an active subscription BEFORE checks can pass.** Assignment upsert does **no** entitlement validation — it accepts a budget for an un-entitled feature; the gap surfaces later at check time. Order it: catalog + subscription first, then budgets.

Setting up that catalog (plan, feature, subscription — a FREE plan needs `charges.pricingType` before publishing) is **not** governance's job: do it via **`stigg-pricing-modeling`** (catalog) and **`stigg-subscriptions`** (provisioning), then add per-entity budgets.

## Before You Start

Per the umbrella `stigg` skill: **search first.** Governance is a private-beta surface whose shape may change — confirm operation names, field names, and body shapes via `search_docs` (or the Mintlify Stigg docs MCP) before authoring. This skill owns the mental model; the live surface owns the shapes.

## Mental Model — Four Pieces

```text
┌─────────────────────────────────────────────┐
│  Entity type   (per environment)            │
│  a kind of unit the customer defines — with │
│  attributionKeys (max 2) that map event     │
│  dimensions → entities                      │
└───────────────────┬─────────────────────────┘
                    │ instantiated per customer as
                    ▼
┌─────────────────────────────────────────────┐
│  Entities      (per customer)               │
│  flat records; hierarchy set via parentId   │
│  on the ASSIGNMENT — up to 4 levels,        │
│  in whatever shape the customer defined     │
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

An **entity type** declares a kind of sub-customer unit — whatever the customer named in Step 1 (a project, region, tenant, cost center, agent, …) — and, critically, its **attribution keys**: up to **2** event-dimension keys whose values identify which entity a usage event belongs to. Operations: **list** and **upsert** (bulk `PUT`, idempotent). Design attribution keys before reporting any usage; retro-attributing old events is not a thing.

## Entities

**Entities** are the per-customer instances of those types — up to **4 levels** deep, in whatever shape Step 1 surfaced (e.g. `client → project`, or `region → tenant → workspace`). The record is flat (`id`, `entityTypeId`, `metadata`); **hierarchy is `parentId` on the *assignment*, not the entity** — the entity upsert rejects `parentId`, and `metadata` (free-form, no structural role) is **not** a place to model a parent. Operations: **list / get / upsert (bulk `PUT`) / archive / unarchive**. **Re-parenting is leaf-only** (a non-leaf move is rejected); **archive is NOT leaf-gated** — archiving a parent orphans its still-active children, so archive/re-home descendants bottom-up yourself. Full rules: `references/entity-model.md`.

## Keeping Entities in Sync

Entities are an **ongoing mirror of the vendor's source of truth**, not a one-time import — the hierarchy lives in your system first; the tree is a projection. Wire governance ops into the code paths that mutate your model: **register → entity upsert; rename/attribute-change → idempotent re-upsert; deprovision/delete → archive (bottom-up); restructure → re-upsert the assignment's `parentId` (leaf-only)**. Since upsert is idempotent bulk `PUT`, a declarative "push the desired tree each sync" pattern works. Full mapping + the unshipped allocation-return caveat: `references/entity-lifecycle.md`.

## Assignments — the Budgets

An **assignment** sets a limit for one `(entity, capability)` pair:

| Field | Meaning |
|---|---|
| `entityId` | The entity being budgeted (any level of the customer's tree). |
| `featureId` **or** `currencyId` | The capability being limited — exactly one: a **`featureId`** (metered feature) **or** a **`currencyId`** (credit currency). Mutually exclusive; there is no `capability` field on the wire. |
| `usageLimit` | The budget for the cadence window. |
| `cadence` | ISO-8601 duration, e.g. `'P1M'` (monthly), `'P1D'` (daily). |
| `scopeEntityIds` | Optional — restricts the limit to usage attributed to specific (typically dimensional) entities, instead of everything under the assignee. |

Operations: **list / upsert**. A `currencyId` capability budgets *credit consumption* — unrelated to billing currency (USD/EUR). Worked examples: `references/assignments.md`.

## Reporting Usage — the Meter-Type Fork

Ingest never gates; it records consumption and rolls up the tree. **The endpoint depends on the target feature's meter type:**

- **Calculated / incremental meters** (your app computes the number) → **`client.v1.usage.report`**: synchronous, echoes `currentUsage`. Required: `customerId`, `featureId` (feature OR credit id), `value`.
- **Raw-events meters** (Stigg aggregates raw events — count/sum/unique) → **`client.v1.events.report`** (reportEvent): the **required** path for these today (usage-reporting events isn't supported); high-volume, eventually-consistent. reportEvent has **no `featureId`** — the meter is matched by **`eventName`** (+`idempotencyKey`).

Both carry attribution in **`dimensions`** (the entity type's attributionKey values, e.g. `{ "projectId": "proj-atlas", "regionId": "eu-west" }`) → attributes to the matching entity **and every ancestor**.

> **Meter type isn't reliably readable via the API** (the `meterType` enum has no "raw events" value; event meters live in a separate `meter` config). So **try then fall back**: call `usage.report`; on **`CannotReportUsageForEntitlementWithMeterError`** it's a raw-events meter — switch to `events.report`. Shapes + examples: `references/enforcement-and-usage.md`.

## Enforcement and Balances

Two distinct reads — do not mix them up:

1. **Access (gates requests):** the standard entitlements **check** — `client.v1Beta.customers.entitlements.check`, passing `featureId` **or** `currencyId`, optional `requestedUsage` (default 1), and attribution `dimensions`. When governance is enabled, the governance chain enriches the check. **An entity is blocked when ANY ancestor has hit its hard limit** — a child with budget left is still blocked if any of its parents is exhausted. Requires an entitling active subscription (see Prerequisites).
2. **Balances (observability):** the **governance tree query** — the `retrieveGovernance` operation (see Surface Map for its exact namespace) — returns the entity nodes; remaining budget = `usageLimit − currentUsage`. **By default it runs tree-only: `usageLimit` / `currentUsage` / `utilization` / `cadence` come back `null`;** pass `featureIds` and/or `currencyIds` params (repeat for multiple) to hydrate config + usage. The read model **may lag by minutes** — use it for dashboards and alerts, **never** to gate access. Full semantics: `references/enforcement-and-usage.md`.

## Surface Map

Operations run through the MCP `execute` tool as `@stigg/typescript` SDK calls (bulk upserts are idempotent):

| Operation | SDK call (via MCP `execute`) |
|---|---|
| Entity types — list / upsert (bulk) | `client.v1Beta.entityTypes.*` |
| Entities — list / get / upsert (bulk) / archive / unarchive | `client.v1Beta.customers.entities.*` |
| Assignments — list / upsert | `client.v1Beta.customers.assignments.*` |
| Entitlements check (never gated) | `client.v1Beta.customers.entitlements.check` |
| Governance tree query (pass `featureIds` / `currencyIds` to hydrate) | `client.v1.events.beta.customers.retrieveGovernance(id)` |
| Ingest — calculated/incremental meter | `client.v1.usage.report` |
| Ingest — raw-events meter (by `eventName`) | `client.v1.events.report` |

Wart to expect: **the tree query lives under `client.v1.events.beta.customers`**, not `client.v1Beta.customers` like its siblings — a known naming inconsistency. Don't "fix" it by guessing a `v1Beta` equivalent; confirm exact operations via `search_docs`.

**API vs UI — different assignment models, by design.** The API targets **one capability per assignment** (`(entity, featureId|currencyId)`). The dashboard UI intentionally **auto-creates an unlimited assignment across ALL capabilities** per entity (you edit limits per row). So UI- and API-created entities look different — don't be surprised, and don't "repair" one to match the other.

**Via the Stigg MCP:** the MCP exposes only **`execute`** (sandboxed TypeScript over the SDK) and **`search_docs`** — no one-tool-per-endpoint mapping. Governance ops go through `execute` using the SDK namespaces above. Connection, keys, and environment setup are the **`stigg-mcp`** skill's job — if `execute` or `search_docs` won't connect, that's where the troubleshooting lives, not here.

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
| Debugging "child has budget but is blocked" as a bug | Ancestor semantics: any exhausted ancestor blocks the whole subtree. Check the tree upward. |
| Defining more than 2 attribution keys, or >4 hierarchy levels | Hard limits. Redesign the model (e.g. fold a level into `scopeEntityIds`). |
| Reporting usage without attribution-key dimensions | Usage won't attribute to any entity — budgets never accrue. Design dimensions up front. |
| Always calling `usage.report`, or always `events.report` | It's a meter-type fork. Calculated/incremental → `usage.report`; raw-events → `events.report` (reportEvent, by `eventName`). If `usage.report` returns `CannotReportUsageForEntitlementWithMeterError`, switch to reportEvent. |
| Passing a `featureId` to reportEvent | reportEvent targets the meter by `eventName` (+`idempotencyKey`), not `featureId`; attribution rides in `dimensions`. |
| Creating an assignment and expecting checks to pass | Assignment upsert does no entitlement validation. The feature must be entitled via an active subscription first (permanent 1.0↔2.0 ordering), else check denies with `NoActiveSubscription`/`NoFeatureEntitlementInSubscription`. |
| Reading the tree and getting `null` limits/usage | Tree-only mode. Pass `featureIds`/`currencyIds` to hydrate config + usage. |
| Treating an MCP connection error as a governance problem | Connection/setup issues (bad key, wrong environment) are `stigg-mcp`'s domain — fix them there, then re-run the governance op. |
| Re-parenting a node that still has children | Re-parent is leaf-only — a non-leaf move is rejected (surfaces as an opaque 500, not transient). Move/archive children first, then move the former parent. |
| Assuming archive is leaves-only | It isn't — archiving a parent orphans its active children. Archive/re-home descendants bottom-up yourself; the platform won't stop you. |
| Modeling an entity's parent via `metadata` | `metadata` is free-form and structural-role-free. Hierarchy is `parentId` on the assignment only. |
| Treating entities as a one-time import | They're an ongoing mirror of your source of truth — wire upsert/archive/re-parent into your create/update/delete/restructure paths (`references/entity-lifecycle.md`). |
| Reading `currencyId` capability as billing currency | It's a credit currency (see `stigg-credits`), not USD/EUR. |
| Expecting per-endpoint MCP tools for governance | The MCP has only `execute` + `search_docs`. Drive the SDK through `execute`. |
