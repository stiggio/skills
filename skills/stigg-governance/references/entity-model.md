# Entity Model — Entity Types and Entities

The structural half of governance: what kinds of units exist (entity types) and which units a given customer actually has (entities). Field names and SDK operations below reflect the surface at the time of writing — governance is a private-beta surface whose shape may change, so **confirm shapes via the Stigg MCP's `search_docs` before calling.**

## Entity Types (per environment)

An entity type declares a category of sub-customer unit — whatever the customer defined (see SKILL.md Step 1; the model is arbitrary, not a fixed dept/team/agent hierarchy) — and how usage events map onto it.

- **Scope** — Environment-level — shared across customers.
- **`attributionKeys`** — **Max 2** event-dimension keys. When a usage event carries these dimensions, its usage attributes to the matching entity.
- **Operations** — **List**, **upsert** — upsert is a **bulk `PUT`**: idempotent create-or-update, safe to re-run from provisioning scripts.
- **SDK** — `client.v1Beta.entityTypes.*`

### Designing attribution keys

Attribution keys are the join between your event stream and the governance tree — get them right **before** reporting usage:

- Pick dimensions your app already emits (or can start emitting) on every relevant event — whatever identifies the customer's units (e.g. `projectId`, `regionId`, `tenantId`, `costCenterId`).
- Two keys max per type. If you're tempted to add a third, that's usually a sign one "key" is really a dimensional scope — model it with `scopeEntityIds` on the assignment instead (see `assignments.md`).
- There is no retroactive attribution. Events reported before the dimensions existed never attribute.

## Entities (per customer)

Entities are the instances: *this* customer's own units (whatever they defined in Step 1 — projects, regions, tenants, cost centers, …).

- **Scope** — Per customer.
- **Record shape** — Flat: `id`, `entityTypeId`, `metadata` (free-form key/value; patch semantics — `""` deletes a key, omitted keys preserved). The entity upsert **rejects `parentId`** (`Unrecognized key "parentId"`), and `metadata` has **no structural role** — parenting happens on `upsertAssignment` (see Hierarchy).
- **Hierarchy** — To place/move a node, set **`parentId` on its `upsertAssignment`** (see `assignments.md`). `parentId` is the entity's **single parent** — one position per entity, **up to 4 levels**, in whatever shape the customer defined. **Re-parenting is leaf-only** (see `assignments.md`).
- **Operations** — **List / get / upsert (bulk `PUT`) / archive / unarchive**. Default list excludes archived.
- **Archive** — **Not leaf-gated.** Archiving a parent succeeds even with children — the children stay active but orphaned (their assignments still point at the archived parent). To retire a subtree cleanly, archive/re-home **descendants first** yourself. Unarchive restores an entity. The "archive the leaves first" rule people expect is really the **re-parent** constraint, not archive.
- **SDK** — `client.v1Beta.customers.entities.*`

### Hierarchy guidance

- Usage attributed to an entity also attributes to **every ancestor** — the tree is a roll-up structure, not just an org chart. Design levels around where you want budgets, not around your full org taxonomy.
- The 4-level cap is a hard limit. If your org model is deeper, flatten the levels that don't carry budgets.
- Bulk `PUT` upsert makes the entity tree declarative: your provisioning code can push the desired tree on every sync and let Stigg reconcile.

### Example — upsert entities, then set `parentId` on the assignments

> **This is ONE possible model — ask your customer for theirs; it is not a default.** The `client → project` shape below is illustrative; it teaches the API's shape, not the hierarchy to adopt.

Entities are declared **flat** — `parentId` does **not** belong here. You place each node in the tree when you call `upsertAssignment` (see `assignments.md`), setting `parentId` to the node's parent.

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
// 1) Entities: flat records, no parentId.
await client.v1Beta.customers.entities.upsert('customer-123', {
  entities: [
    { id: 'client-acme',  entityTypeId: 'client' },
    { id: 'proj-atlas',   entityTypeId: 'project' },
    { id: 'proj-borealis', entityTypeId: 'project' },
  ],
});
// 2) parentId is set on the upsertAssignment call (see assignments.md).
```

## Common mistakes

Each mistake — the fix:

- Creating entity types per customer — Types are environment-level; entities are per customer.
- Modeling every org level as an entity level — Only levels that carry budgets or need roll-up belong in the tree — 4 levels max.
- Assuming archive is leaves-only — It isn't — archiving a parent orphans its active children. Archive/re-home descendants bottom-up yourself; the platform won't block you. (Leaf-only is the *re-parent* rule.)
- Re-parenting a node with children — Leaf-only — rejected (opaque 500). Move/archive its children first.
- Putting `parentId` on the entity upsert (or in `metadata`) — The entity upsert rejects `parentId`, and `metadata` has no structural role. Set `parentId` on the `upsertAssignment` operation to place/move the node.
- Renaming attribution keys after events flow — Old events keep the old dimensions and won't re-attribute. Treat keys as append-only contracts.
