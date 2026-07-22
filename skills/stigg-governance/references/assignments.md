# Assignments — Budgets per (Entity, Capability)

An assignment is the budget itself: a usage limit for one entity on one capability. Field names below reflect the surface at the time of writing — **confirm shapes via the Stigg MCP's `search_docs` before calling.**

## Shape

- `entityId` — The entity being budgeted (any level of the customer's tree).
- `parentId` — The entity's **single parent** — one position per entity, set here to place/move the node (not a per-capability value). **Tri-state:** *omit* = leave the parent unchanged (a new node defaults to root); `null` = make it a root; an *entity id* = set/change the parent. This payload is flat (one row per entity × capability × scope), so the same entity across rows carries the same `parentId`. **Re-parenting is leaf-only** — moving a node that still has children is rejected (surfaces as an opaque 500, not transient; move/archive children first).
- `featureId` **or** `currencyId` — What's limited — exactly one on the wire: a **`featureId`** (metered feature) **or** a **`currencyId`** (credit currency). There is no `capability` field; "capability" is just the shorthand for whichever of the two you set.
- `usageLimit` — The budget for one cadence window.
- `cadence` — ISO-8601 duration for the reset window — `'P1M'` (monthly), `'P1D'` (daily), `'P1W'` (weekly), `'P1Y'` (yearly).
- `scopeEntityIds` — Optional — narrows which attributed usage counts against this limit (see below).

Operations: **list / upsert**. SDK: `client.v1Beta.customers.assignments.*`.

## Feature vs credit-currency capability

- **`featureId`** — budgets a metered feature directly: "`proj-atlas` may make 10k `api_calls` per month."
- **`currencyId`** — budgets **credit consumption** in that currency: "`client-acme` may burn 50k `ai_tokens` credits per month." This is a Stigg credit currency (see `stigg-credits`), **not** a billing currency — there is no USD/EUR budgeting here.

Pick the credit-currency form when multiple features drain one pool and you want a single budget over all of them; pick the feature form for per-feature caps.

## `scopeEntityIds` — dimensional / high-cardinality scope

`scopeEntityIds` defines a **dimension-scoped sub-budget** — it narrows **when** the budget applies:

- `[]` (empty) — a node-wide budget that always applies.
- Non-empty (e.g. `['model-gpt4o']`) — the budget applies only when **every** listed entity is present in the request's resolved set (the ancestors + dimension matches produced by attribution — see `enforcement-and-usage.md`).

By default (empty) an assignment covers **all** usage attributed to the entity (including roll-up from descendants). A non-empty `scopeEntityIds` restricts the counted usage to specific entities — the tool for **high-cardinality or dimensional** cases where creating a hierarchy level per value would blow the 4-level / tree-size budget:

> "Limit `proj-atlas`'s usage of *model-gpt5* to 10k credits/month" — model the models as entities of a dimensional entity type, then scope the project's assignment with `scopeEntityIds: ['model-gpt5']` instead of adding a hierarchy level under every project.

## Cadence semantics

- The limit applies per cadence window (`'P1M'` = per month). Usage resets with the window.
- Different assignments on the same entity can carry different cadences (daily cap + monthly cap on the same capability is a valid belt-and-suspenders pattern).

## Example — upsert assignments (via MCP `execute`, which runs SDK code)

Continuing the **one possible model** (`client → project`) from `entity-model.md` — ask your customer for theirs, this is not a default:

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
// Set parentId here to place each node in the tree (root = parentId: null).
await client.v1Beta.customers.assignments.upsert('customer-123', {
  assignments: [
    { entityId: 'client-acme', parentId: null,          currencyId: 'ai_tokens', usageLimit: 50000, cadence: 'P1M' },
    { entityId: 'proj-atlas',  parentId: 'client-acme',  featureId: 'api_calls',  usageLimit: 10000, cadence: 'P1M' },
  ],
});
```

## Re-parenting — set a new `parentId` on `upsertAssignment`

To move a node in the tree, call `upsertAssignment` for the entity with a new `parentId`. `parentId` is the entity's single parent, so this moves the node itself. Omitted fields (`usageLimit`, `cadence`) are preserved:

```ts
// Move proj-atlas from client-acme to client-globex — usageLimit/cadence preserved.
await client.v1Beta.customers.assignments.upsert('customer-123', {
  assignments: [{ entityId: 'proj-atlas', parentId: 'client-globex', featureId: 'api_calls' }],
});
```

Leaf-only: if `proj-atlas` itself has children, this is rejected (opaque 500) — re-parent or archive those children first. Wiring this into your restructure code path: `entity-lifecycle.md`.

## Interaction with the hierarchy

Assignments at different levels stack as independent gates: usage attributed to a leaf counts against the leaf's budget and every ancestor's budget simultaneously, and **any** exhausted level blocks the leaf (see `enforcement-and-usage.md`).

## Common mistakes

Each mistake — the fix:

- Passing both `featureId` and `currencyId` — A capability is one or the other.
- Using `scopeEntityIds` for a natural hierarchy level — If the values are few and stable, model them as a tree level; scope is for high-cardinality dimensions.
- Assuming a child's budget caps the parent — Roll-up goes upward only — a parent limit constrains all descendants, not vice versa.
- Writing `cadence: 'monthly'` — ISO-8601 durations only — `'P1M'`.
- Budgeting in USD via `currencyId` — Credit currencies only. Money budgets go through credit cost modeling (`stigg-credits`).
- Re-parenting a node with children — Leaf-only — rejected (opaque 500). Move/archive children first.
- Expecting assignment upsert to validate entitlement — No validation at assignment time. The feature must be entitled by an active subscription first, or the later check denies (`NoActiveSubscription`/`NoFeatureEntitlementInSubscription`).
