# Entity Entitlements — Limits per (Entity, Feature or Credit Currency)

An entity entitlement is the limit itself: a usage limit for one entity on one feature or credit currency. v1-beta called it an *assignment*. Field names below reflect the surface at the time of writing — **confirm shapes via the Stigg MCP's `search_docs` before calling.**

## Shape

- `entityId` — The entity being limited (any level of the customer's tree).
- `parentId` — The entity's **single parent** — one position per entity, set here to place/move the node (not per feature/currency). **Tri-state:** *omit* = leave the parent unchanged (a new node defaults to root); `null` = make it a root; an *entity id* = set/change the parent. This payload is flat (one row per entity × feature-or-currency × scope), so the same entity across rows carries the same `parentId`. **Re-parenting is leaf-only** — moving a node that still has children is rejected; move its children to another parent, or detach them with `parentId: null`, first. An archived child still counts.
- `featureId` **or** `currencyId` — What's limited — exactly one on the wire: a **`featureId`** (metered feature) **or** a **`currencyId`** (credit currency).
- `usageLimit` — The limit for one cadence window. **`null` = unlimited**: no ceiling, usage is still tracked and rolls up.
- `cadence` — A **single-unit** ISO-8601 duration for the reset window — `'P1M'` (monthly), `'P1D'` (daily), `'P1W'` (weekly), `'P1Y'` (yearly), `'PT1H'` (hourly), or a single count like `'P10D'` or `'PT30M'`. It must be one unit only — a composite like `'P1M10D'` (1 month *and* 10 days) is **not** allowed.
- `scopeEntityIds` — Optional — narrows which attributed usage counts against this limit (see below). At most 5 ids, no duplicates, order does not matter.

Operations: **list / upsert**. SDK: `client.v1.customers.entities.entitlements.*`. The upsert body key is **`entityEntitlements`**, at most 100 rows per call. List filters by `entityId` and by `featureId` **or** `currencyId`, and returns pages.

## How an upsert matches a row

A row is identified by **`(entityId, featureId|currencyId, scopeEntityIds)`**:

- A match is **updated**. Fields you omit (`usageLimit`, `cadence`) keep their current values.
- No match is **created**. `usageLimit` and `cadence` are both required (`usageLimit: null` counts — it creates an unlimited row).
- The scope is part of the key, so a node-wide limit (`[]`) and a scoped limit (`['model-gpt5']`) on the same feature are two separate rows.
- The cadence is **not** part of the key. Upserting the same key with another cadence changes that row's cadence; it does not add a second window.
- Two rows with the same key in one request are rejected (400).

## Feature vs credit currency

- **`featureId`** — limits a metered feature directly: "`proj-atlas` may make 10k `api_calls` per month."
- **`currencyId`** — limits **credit consumption** in that currency: "`client-acme` may burn 50k `ai_tokens` credits per month." This is a Stigg credit currency (see `stigg-credits`), **not** a billing currency — there are no USD/EUR limits here.

Pick the credit-currency form when multiple features drain one pool and you want a single limit over all of them; pick the feature form for per-feature caps.

## `scopeEntityIds` — dimensional / high-cardinality scope

`scopeEntityIds` defines a **dimension-scoped sub-limit** — it narrows **when** the limit applies:

- `[]` (empty) — a node-wide limit that always applies.
- Non-empty (e.g. `['model-gpt4o']`) — the limit applies only when **every** listed entity is present in the request's resolved set (the ancestors + dimension matches produced by attribution — see `enforcement-and-usage.md`).

By default (empty) an entity entitlement covers **all** usage attributed to the entity (including roll-up from descendants). A non-empty `scopeEntityIds` restricts the counted usage to specific entities — the tool for **high-cardinality or dimensional** cases where creating a hierarchy level per value would blow the 8-level / tree-size limit:

> "Limit `proj-atlas`'s usage of *model-gpt5* to 10k credits/month" — model the models as entities of a dimensional entity type, then scope the project's entity entitlement with `scopeEntityIds: ['model-gpt5']` instead of adding a hierarchy level under every project.

## Cadence semantics

- The limit applies per cadence window (`'P1M'` = per month). Usage resets with the window.
- The cadence must be a **single-unit** duration (`'P1M'`, `'P10D'`, …); a composite like `'P1M10D'` is rejected.
- One row carries one cadence. For a daily cap **and** a monthly cap, put them on rows with different keys — for example the monthly cap on the parent and the daily cap on the leaf. A second upsert of the same key only replaces the cadence.

## Example — upsert entity entitlements (via MCP `execute`, which runs SDK code)

Continuing the **one possible model** (`client → project`) from `entity-model.md` — ask your customer for theirs, this is not a default:

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
// Set parentId here to place each node in the tree (root = parentId: null).
await client.v1.customers.entities.entitlements.upsert('customer-123', {
  entityEntitlements: [
    { entityId: 'client-acme', parentId: null,          currencyId: 'ai_tokens', usageLimit: 50000, cadence: 'P1M' },
    { entityId: 'proj-atlas',  parentId: 'client-acme',  featureId: 'api_calls',  usageLimit: 10000, cadence: 'P1M' },
  ],
});
```

## Re-parenting — set a new `parentId` on the entity entitlement upsert

To move a node in the tree, upsert an entity entitlement for the entity with a new `parentId`. `parentId` is the entity's single parent, so this moves the node itself. Omitted fields (`usageLimit`, `cadence`) are preserved:

```ts
// Move proj-atlas from client-acme to client-globex — usageLimit/cadence preserved.
await client.v1.customers.entities.entitlements.upsert('customer-123', {
  entityEntitlements: [{ entityId: 'proj-atlas', parentId: 'client-globex', featureId: 'api_calls' }],
});
```

Leaf-only: if `proj-atlas` itself has children, this is rejected — move those children to another parent, or detach them with `parentId: null`, first. Archiving them does not help. Wiring this into your restructure code path: `entity-lifecycle.md`.

## Interaction with the hierarchy

Entity entitlements at different levels stack as independent gates: usage attributed to a leaf counts against the leaf's limit and every ancestor's limit simultaneously, and **any** exhausted level blocks the leaf (see `enforcement-and-usage.md`).

## Common mistakes

Each mistake — the fix:

- Sending `assignments` as the body key — v1 expects `entityEntitlements`.
- Passing both `featureId` and `currencyId` — An entity entitlement limits one or the other, never both.
- Using `scopeEntityIds` for a natural hierarchy level — If the values are few and stable, model them as a tree level; scope is for high-cardinality dimensions.
- Assuming a child's limit caps the parent — Roll-up goes upward only — a parent limit constrains all descendants, not vice versa.
- Writing `cadence: 'monthly'` — ISO-8601 durations only — `'P1M'`.
- Writing a composite cadence like `'P1M10D'` — Single-unit durations only; use `'P1M'` **or** `'P10D'`, not both in one cadence.
- Upserting the same key twice to get two cadences — The key has no cadence; the second upsert replaces the first cadence. Put the second cap on a different key.
- Reading `usageLimit: null` as zero — `null` means unlimited.
- Limiting in USD via `currencyId` — Credit currencies only. Money limits go through credit cost modeling (`stigg-credits`).
- Re-parenting a node with children — Leaf-only — rejected. Move the children to another parent, or detach them with `parentId: null`, first; an archived child still counts.
- Expecting the upsert to validate entitlement — No validation at upsert time. The feature must be entitled by an active subscription first, or the later check denies (`NoActiveSubscription`/`NoFeatureEntitlementInSubscription`).
