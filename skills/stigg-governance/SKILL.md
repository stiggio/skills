---
name: stigg-governance
description: Use for per-entity usage-limit and spend-control work in Stigg Governance — enforcing usage limits over whatever hierarchy of sub-units a customer needs (the entity model is customer-defined, not fixed) — defining entity types with attribution keys, setting entity entitlements for a feature or credit currency, dimensional scoping, querying the governance tree, runtime enforcement, and moving v1-beta code to v1. Triggers include "per-entity budget", "per-team spend limit", "per-agent token cap", "cost center limits", "sub-customer limits", "entity type", "entity entitlement", "attribution keys", "governance tree", "budget enforcement", "scoped assignment", "v1Beta governance", "keep entities in sync", "re-parent entity", "CannotReportUsageForEntitlementWithMeterError". Skip for account-level credit grants / pools / ledger (stigg-credits), feature gating with no entity hierarchy (stigg-entitlements), or catalog modeling (stigg-pricing-modeling).
---

# Stigg Governance — Per-Entity Usage Limits

Governance layers **per-entity limit enforcement** on top of Stigg credits and features. Where credits answer "how much can this *customer* consume?", governance answers "how much can this customer's *own sub-units* consume?" — whatever the vendor needs to limit, with hierarchical roll-up and hard-limit enforcement. **The entity model is arbitrary and customer-defined** — Stigg imposes no fixed hierarchy, so don't assume one (see Step 1).

You drive Stigg governance **through the Stigg MCP** — the `execute` tool runs SDK code, `search_docs` resolves exact shapes. That's how the agent explores, configures, and test-checks governance: entity types, entities, entity entitlements, tree queries, and one-off checks all go through `execute` using the SDK's `v1` namespace (`client.v1.…`). **Production runtime enforcement is separate:** the live entitlements check that gates real customer requests runs on the vendor's **hot path via the Stigg SDK/edge**, not through the MCP — the MCP is for agent-side setup and exploration, not the request-time gate. This skill owns the *mental model*; it does not cover connection, keys, or setup — those belong to the **`stigg-mcp`** skill.

> **Governance is GA on the `v1` API.** Use the `client.v1.*` calls. The `client.v1Beta.*` governance calls still work but are deprecated. One exception: the governance tree query has no v1 version and stays on `client.v1Beta`. For existing v1-beta code, see **Moving from v1-beta** below.

## Step 1 — Elicit the Model: ASK, Don't Assume (do this BEFORE defining entity types)

**The governance entity model is fully arbitrary and customer-defined.** There is **no default hierarchy** — Stigg ships none, and any dept/team/agent shape is just *one* possibility, not a template. Before defining a single entity type, **ask the customer what they want to govern** and build strictly from their answer. Do **not** propose or assume a hierarchy. Ask:

- **What units do you limit?** (their own vocabulary — teams, projects, regions, tenants, sub-accounts, cost centers, agents, workspaces…)
- **How do they nest / roll up?** (which contains which, and how deep — the tree caps at 8 levels)
- **Which levels actually carry limits?** (only levels that hold a limit or need roll-up belong in the tree; fold the rest away)
- **What identifies a unit on a usage event?** (the dimension key(s) your events already emit — these become the attribution keys)

Model from their answer: "we limit by project, and projects belong to a client" → build `client → project`. Every worked example below and in the references is *one possible model* — it teaches the API's *shape*, never the model to adopt.

**Sequence:** elicit the model (this step) → entity types → entities → entity entitlements → runtime.

## Prerequisites — depend on what you're limiting

Governance limits sit *on top of* an existing feature or credit currency; they don't replace it. **What's required differs for a feature vs. a credit currency** — the two follow different paths, so read the row that matches your entity entitlement (`featureId` or `currencyId`).

- **Feature (`featureId`) — needs an entitling active subscription.** The gating check can pass only when the customer has an **ACTIVE subscription that entitles the feature** — otherwise it denies with a standard reason (`NoActiveSubscription`; `NoFeatureEntitlementInSubscription` = subscribed but the plan lacks the feature). Governance still runs and `chains` can still come back, but the check cannot pass. **Entitle the feature via an active subscription BEFORE checks can pass.** The entity entitlement upsert does **no** entitlement validation — it accepts a limit for an un-entitled feature, so the gap surfaces only later at check time. Order it: catalog + subscription first (see `stigg-pricing-modeling` / `stigg-subscriptions`), then limits.

- **Credit currency (`currencyId`) — does NOT require a subscription.** Credits run on a different path: the check reads the customer's **credit pool** directly, and credits can be granted and consumed with **no active subscription or assigned plan**. What a credit limit needs instead is a **credit currency that exists and a funded pool** (a grant — recurring, prepaid, promotional, or manual). Set the currency up via **`stigg-credits`**, then add per-entity limits.

Setting up a feature catalog (plan, feature, subscription — a FREE plan needs `charges.pricingType` before publishing) is **not** governance's job: do it via **`stigg-pricing-modeling`** (catalog) and **`stigg-subscriptions`** (provisioning); credit currencies and grants live in **`stigg-credits`**. Then add per-entity limits.

## Before You Start

Per the umbrella `stigg` skill: **search first.** Confirm operation names, field names, and body shapes via `search_docs` (or the Mintlify Stigg docs MCP) before authoring. This skill owns the mental model; the live surface owns the shapes.

## The Four Pieces

`entity type → entity → entity entitlement → governance tree` — a **type** (which declares the attribution keys) is defined **once per environment** and instantiated **per customer** as **entities**; an **entity entitlement** limits one `(entity, featureId|currencyId)` pair, optionally narrowed by a scope; the **tree** reports usage-vs-limit and never gates. An entity entitlement is what v1-beta called an *assignment*. Each piece is detailed below.

## Entity Types

An **entity type** declares a kind of sub-customer unit — whatever the customer named in Step 1 (a project, region, tenant, cost center, agent, …) — and, critically, its **attribution keys**: the event-dimension keys whose values identify which entity a usage event belongs to. An entity type is **defined once per environment** and shared across every customer in that environment. Operations: **list**, **upsert** (bulk `PUT`, idempotent), **archive**, and **checkActiveEntities**. **Archive fails with 400 while any entity of that type is still active** (in any customer): call `checkActiveEntities(id)` first, archive the entities, then the type. An upsert of an archived type restores it; list hides archived types unless you pass `includeArchived: 'true'`. Design attribution keys before reporting any usage; retro-attributing old events is not a thing.

## Entities

**Entities** are the instances of those types, **created per customer** (under a customer) — many per customer, up to **8 levels** deep, in whatever shape Step 1 surfaced (e.g. `client → project`, or `region → tenant → workspace`). To **place or move a node in the tree, set `parentId` on the entity entitlement upsert** (`client.v1.customers.entities.entitlements.upsert`) — tri-state: **omit** = leave the parent unchanged, **`null`** = make it a root, an **entity id** = set/change it. `parentId` is the entity's **single parent** (one position per entity), set on the entity entitlement, not per feature/currency. (Practical note: the *entity* upsert rejects `parentId`, and `metadata` has no structural role, so parenting happens on the entity entitlement upsert.) Operations: **list / get / upsert (bulk `PUT`) / archive**. There is no unarchive: **an upsert of an archived entity restores it**. An entity on an archived entity type cannot be upserted (400). **Re-parenting is leaf-only** (a non-leaf move is rejected, and an archived child still counts as a child); **archive is NOT leaf-gated** — archiving a parent orphans its still-active children, so archive/re-home descendants bottom-up yourself. Full rules: `references/entity-model.md`.

## Keeping Entities in Sync

Entities are an **ongoing mirror of the vendor's source of truth**, not a one-time import — the hierarchy lives in your system first; the tree is a projection. Wire governance ops into the code paths that mutate your model: **register → entity upsert; rename/attribute-change → idempotent re-upsert; deprovision/delete → archive (bottom-up); reactivate → re-upsert the archived entity; restructure → set `parentId` on the entity's entity entitlement upsert to move the node (leaf-only)**. Since upsert is idempotent bulk `PUT`, a declarative "push the desired tree each sync" pattern works. Full mapping + the unshipped allocation-return caveat: `references/entity-lifecycle.md`.

## Entity Entitlements — the Limits

An **entity entitlement** sets a usage limit for one `(entity, featureId|currencyId)` pair:

- `entityId` — The entity being limited (any level of the customer's tree).
- `featureId` **or** `currencyId` — What's being limited — exactly one: a **`featureId`** (metered feature) **or** a **`currencyId`** (credit currency). Mutually exclusive.
- `usageLimit` — The limit for the cadence window. **`null` = unlimited**: no ceiling, usage is still tracked.
- `cadence` — A **single-unit** ISO-8601 duration for the reset window, e.g. `'P1M'` (monthly), `'P10D'` (every 10 days) or `'PT1H'` (hourly). It must be one unit only — a composite like `'P1M10D'` (1 month *and* 10 days) is **not** allowed.
- `scopeEntityIds` — Optional — defines a dimension-scoped sub-limit; it narrows **when** the limit applies. `[]` (empty) = a node-wide limit that always applies. Non-empty (e.g. `['model-gpt4o']`) = the limit applies only when every listed entity is present in the request's resolved set (the ancestors + dimension matches from attribution). At most 5 ids, no duplicates. Use it instead of everything under the entity for specific (typically dimensional) entities.

Operations: **list / upsert**, at most 100 rows per upsert. The upsert body key is **`entityEntitlements`** (v1-beta used `assignments`). A row is matched by **`(entityId, featureId|currencyId, scopeEntityIds)`**: a match is updated (omitted `usageLimit` / `cadence` are kept), anything else is created (both are required). So the same feature under two different scopes is two separate limits. A `currencyId` entity entitlement limits *credit consumption* — unrelated to billing currency (USD/EUR). Worked examples: `references/entity-entitlements.md`.

## Reporting Usage — the Meter-Type Fork

Ingest never gates; it records consumption and rolls up the tree. **The endpoint depends on the target feature's meter type:**

- **Calculated / incremental meters** (your app computes the number) → **`client.v2.usage.report`**: synchronous. Its answer lists each measurement's usage and, in `data.entities`, the usage of each entity entitlement it counted towards. Required: `customerId`, `featureId` (a feature id; a credit currency id fails with `FeatureNotFound`), `value`. (`client.v1.usage.report` is deprecated.)
- **Raw-events meters** (Stigg aggregates raw events — count/sum/unique) → **`client.v1.events.report`** (reportEvent): the **required** path for these today (`v2.usage.report` cannot report raw-events–metered features — use `events.report`/reportEvent instead); high-volume, eventually-consistent. reportEvent has **no `featureId`** — the meter is matched by **`eventName`** (+`idempotencyKey`).

Both carry attribution in **`dimensions`** (the entity type's attributionKey values, e.g. `{ "projectId": "proj-atlas", "regionId": "eu-west" }`) → attributes to the matching entity **and every ancestor**.

> **Read the feature's meter to pick the path.** The Feature type exposes `hasMeter` / `meter` / `meterType` — inspect the `meter` config to tell a raw-events meter (→ `events.report`) from a calculated/incremental one (→ `v2.usage.report`). Note `meterType` alone doesn't flag "raw events" (a raw-events feature can read `INCREMENTAL`), so read the `meter` config and confirm fields via `search_docs`. Keep a definitive fallback: on **`CannotReportUsageForEntitlementWithMeterError`** from `v2.usage.report`, it's a raw-events meter — switch to `events.report`. Shapes + examples: `references/enforcement-and-usage.md`.

## Enforcement and Usage Reads

Two distinct reads — do not mix them up. This is the **hot-path (enforcement) vs cold-path (observability)** split; the full framing is in `references/enforcement-vs-observability.md`:

1. **Access (gates requests):** the standard entitlements **check** — `client.v1.customers.checkEntitlement(customerId, { ... })`, passing `featureId` **or** `currencyId`, optional `requestedUsage` (default 1), and attribution `dimensions`. With `dimensions`, the governance limits are checked too. When at least one entity entitlement applies, the response's `data.chains` lists each limit on the path, and a governance denial reads `RequestedUsageExceedingLimit`. **An entity is blocked when ANY ancestor has hit its hard limit** — a child with limit left is still blocked if any of its parents is exhausted. A **feature** (`featureId`) check also needs an entitling active subscription; a **credit** (`currencyId`) check does not (see Prerequisites).
2. **Usage vs limit (observability):** the **governance tree query** — `client.v1Beta.customers.governance.list(customerId, { ... })`, still on v1-beta and paginated — returns the entity nodes; remaining limit = `usageLimit − currentUsage`. **By default it runs tree-only: `usageLimit` / `currentUsage` / `utilization` / `cadence` come back `null`;** pass `featureIds` and/or `currencyIds` (arrays) to hydrate config + usage for those features / credit currencies. Its usage figures **may lag by a few seconds** (more for raw events) — use it for dashboards and alerts, **never** to gate access. Full semantics: `references/enforcement-and-usage.md`.

## Surface Map

Operations run through the MCP `execute` tool as `@stigg/typescript` SDK calls (bulk upserts are idempotent; bulk upserts and archives take at most 100 items per call). Each operation → its SDK call (via MCP `execute`):

- **Entity types — list / upsert (bulk) / archive / checkActiveEntities** — `client.v1.entityTypes.*`
- **Entities — list / retrieve / upsert (bulk, also restores archived) / archive** — `client.v1.customers.entities.*` (`retrieve(entityId, { id: customerId })`)
- **Entity entitlements — list / upsert (body key `entityEntitlements`)** — `client.v1.customers.entities.entitlements.*`
- **Entitlements check (not gated by governance enablement — it still gates access)** — `client.v1.customers.checkEntitlement(customerId, { ... })`
- **Governance tree query (pass `featureIds` / `currencyIds` to hydrate; paginated; v1-beta only)** — `client.v1Beta.customers.governance.list(customerId, { ... })`
- **Ingest — calculated/incremental meter** — `client.v2.usage.report`
- **Ingest — raw-events meter (by `eventName`)** — `client.v1.events.report`

Don't guess operation names or shapes — confirm exact operations via `search_docs`.

**API vs UI — different entity entitlement models, by design.** The API targets **one feature or credit currency per entity entitlement** (`(entity, featureId|currencyId)`). When you create an entity under a parent in the dashboard UI, it writes **one unlimited entity entitlement (`usageLimit: null`) for each feature or credit currency you selected**, to hold the placement (you edit limits per row). A top-level entity gets none. So UI- and API-created entities look different — don't be surprised, and don't "repair" one to match the other.

**Via the Stigg MCP:** the MCP exposes only **`execute`** (sandboxed TypeScript over the SDK) and **`search_docs`** — no one-tool-per-endpoint mapping. Governance ops go through `execute` using the SDK namespaces above. Connection, keys, and environment setup are the **`stigg-mcp`** skill's job — if `execute` or `search_docs` won't connect, that's where the troubleshooting lives, not here.

## Moving from v1-beta

The `client.v1Beta` governance calls are deprecated. Each maps to a v1 call, and the fields inside each row keep their names:

- `v1Beta.entityTypes.list / upsert` → `v1.entityTypes.list / upsert` (v1 adds `archive` and `checkActiveEntities`)
- `v1Beta.customers.entities.list / retrieve / upsert / archive` → the same names on `v1.customers.entities`
- `v1Beta.customers.entities.unarchive` → `v1.customers.entities.upsert` with the archived entity's id
- `v1Beta.customers.assignments.list / upsert` → `v1.customers.entities.entitlements.list / upsert`; the upsert body key changes from `assignments` to `entityEntitlements`
- `v1Beta.customers.entitlements.check` → `v1.customers.checkEntitlement` (same `featureId` / `currencyId` / `requestedUsage` / `dimensions` params)
- `v1Beta.customers.governance.list` → no change (no v1 version)
- `v1.usage.report` (deprecated) → `v2.usage.report`, whose answer adds `data.entities`

Raw REST moves the same way, `/api/v1-beta/...` → `/api/v1/...`, with one rename: `/api/v1-beta/customers/{id}/assignments` → `/api/v1/customers/{id}/entities/entitlements`.

## When NOT to Use This Skill

- Account-level credit currencies, grants, pools, ledger, auto-recharge → `stigg-credits`.
- Feature gating / usage reporting with no per-entity limits → `stigg-entitlements`.
- Modeling features / plans / credit currencies in the catalog → `stigg-pricing-modeling`.
- Provisioning / canceling subscriptions → `stigg-subscriptions`.

## Common Mistakes

Each mistake — the fix:

- Writing new code on `client.v1Beta` — Those governance calls are deprecated. Use the v1 calls (see Moving from v1-beta); only the tree query stays on `client.v1Beta`.
- Sending `assignments` to the v1 upsert — The v1 body key is `entityEntitlements`.
- Calling `unarchive` — v1 has none. Re-upsert the archived entity (same id) to restore it.
- Archiving an entity type that still has active entities — The whole request fails with 400. Check `checkActiveEntities(id)`, archive the type's entities first, then the type.
- Gating access from the tree query — Its usage figures may lag by a few seconds. Gate with the entitlements check only.
- Debugging "child has limit left but is blocked" as a bug — Ancestor semantics: any exhausted ancestor blocks the whole subtree. Check the tree upward.
- Building more than 8 hierarchy levels — A hard limit; a deeper placement is rejected. Redesign the model (e.g. fold a level into `scopeEntityIds`).
- Reporting usage without attribution-key dimensions — Usage won't attribute to any entity — limits never accrue. Design dimensions up front.
- Always calling `v2.usage.report`, or always `events.report` — It's a meter-type fork. Calculated/incremental → `v2.usage.report`; raw-events → `events.report` (reportEvent, by `eventName`). If `v2.usage.report` returns `CannotReportUsageForEntitlementWithMeterError`, switch to reportEvent.
- Passing a credit currency id as the usage report's `featureId` — It takes feature ids only and fails with `FeatureNotFound`. Draw credits directly through the credits API (`stigg-credits`).
- Passing a `featureId` to reportEvent — reportEvent targets the meter by `eventName` (+`idempotencyKey`), not `featureId`; attribution rides in `dimensions`.
- Creating a **feature** (`featureId`) entity entitlement and expecting checks to pass — The upsert does no entitlement validation. The feature must be entitled via an active subscription first, else check denies with `NoActiveSubscription`/`NoFeatureEntitlementInSubscription`.
- Assuming a **credit** (`currencyId`) limit needs a subscription like a feature does — It doesn't — credits are a different path. The check reads the credit pool directly; credits can be granted/consumed with no subscription. It needs a credit currency + funded pool, not a subscription (`stigg-credits`).
- Reading `usageLimit: null` as zero — `null` means unlimited; usage still accrues and rolls up.
- Reading the tree and getting `null` limits/usage — Tree-only mode. Pass `featureIds`/`currencyIds` to hydrate config + usage.
- Treating an MCP connection error as a governance problem — Connection/setup issues (bad key, wrong environment) are `stigg-mcp`'s domain — fix them there, then re-run the governance op.
- Re-parenting a node that still has children — Re-parent is leaf-only — a non-leaf move is rejected. Move the children to another parent, or detach them with `parentId: null`, bottom-up; then move the former parent. Archiving a child does not help: an archived child still counts. The rejection is not transient, so don't retry it.
- Assuming archive is leaves-only — It isn't — archiving a parent orphans its active children. Archive/re-home descendants bottom-up yourself; the platform won't stop you.
- Putting `parentId` on the entity upsert (or in `metadata`) — The entity upsert rejects `parentId`, and `metadata` has no structural role. Set `parentId` on the entity entitlement upsert to place/move the node.
- Treating entities as a one-time import — They're an ongoing mirror of your source of truth — wire upsert/archive/re-parent into your create/update/delete/restructure paths (`references/entity-lifecycle.md`).
- Reading `currencyId` as a billing currency — It's a credit currency (see `stigg-credits`), not USD/EUR.
- Expecting per-endpoint MCP tools for governance — The MCP has only `execute` + `search_docs`. Drive the SDK through `execute`.
