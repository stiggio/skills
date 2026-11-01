# Entity Model — Entity Types and Entities

The structural half of governance: what kinds of units exist (entity types) and which units a given customer actually has (entities). Field names and SDK operations below reflect the surface at the time of writing — **confirm shapes via the Stigg MCP's `search_docs` before calling.**

## Entity Types (per environment)

An entity type declares a category of sub-customer unit — whatever the customer defined (see SKILL.md Step 1; the model is arbitrary, not a fixed dept/team/agent hierarchy) — and how usage events map onto it.

- **Scope** — Environment-level — shared across customers.
- **Record shape** — `id`, `displayName` (required), optional `description`, `attributionKeys`.
- **`attributionKeys`** — The event-dimension keys. When a usage event carries these dimensions, its usage attributes to the matching entity.
- **Operations** — **List**, **upsert**, **archive**, **checkActiveEntities**. Upsert is a **bulk `PUT`**: idempotent create-or-update, safe to re-run from provisioning scripts, at most 100 types per call. An upsert of an archived type restores it.
- **Archive** — `archive({ ids })`. If any listed type still has an active entity (in any customer), the whole request fails with 400. `checkActiveEntities(id)` returns whether such an entity exists (an unknown type returns `false`). Archive the entities first, then the type. List hides archived types unless you pass `includeArchived: 'true'`.
- **SDK** — `client.v1.entityTypes.*`

### Designing attribution keys

Attribution keys are the join between your event stream and the governance tree — get them right **before** reporting usage:

- Pick dimensions your app already emits (or can start emitting) on every relevant event — whatever identifies the customer's units (e.g. `projectId`, `regionId`, `tenantId`, `costCenterId`).
- There is no retroactive attribution. Events reported before the dimensions existed never attribute.

## Entities (per customer)

Entities are the instances: *this* customer's own units (whatever they defined in Step 1 — projects, regions, tenants, cost centers, …).

- **Scope** — Per customer.
- **Record shape** — Flat: `id`, `entityTypeId`, optional `displayName`, `metadata` (free-form key/value; patch semantics — `""` deletes a key, omitted keys preserved). The entity upsert **rejects `parentId`** (400, unknown key), and `metadata` has **no structural role** — parenting happens on the entity entitlement upsert (see Hierarchy).
- **Hierarchy** — To place/move a node, set **`parentId` on its entity entitlement upsert** (see `entity-entitlements.md`). `parentId` is the entity's **single parent** — one position per entity, **up to 8 levels**, in whatever shape the customer defined. **Re-parenting is leaf-only** (see `entity-entitlements.md`).
- **Operations** — **List / get / upsert (bulk `PUT`) / archive**, at most 100 entities per call. There is no unarchive: **an upsert of an archived entity restores it**. The default list excludes archived entities; pass `includeArchived: 'true'` to include them, or `entityTypeId` to filter by type. An upsert of an entity whose entity type is archived fails with 400.
- **Archive** — **Not leaf-gated.** Archiving a parent succeeds even with children — the children stay active but orphaned (their entity entitlements still point at the archived parent). To retire a subtree cleanly, archive/re-home **descendants first** yourself. An archived child still counts as a child, so it still blocks re-parenting its parent. Re-upserting an entity restores it. The "archive the leaves first" rule people expect is really the **re-parent** constraint, not archive.
- **SDK** — `client.v1.customers.entities.*`

### Hierarchy guidance

- Usage attributed to an entity also attributes to **every ancestor** — the tree is a roll-up structure, not just an org chart. Design levels around where you want limits, not around your full org taxonomy.
- The 8-level cap is a hard limit; a deeper placement is rejected. If your org model is deeper, flatten the levels that don't carry limits.
- Bulk `PUT` upsert makes the entity tree declarative: your provisioning code can push the desired tree on every sync and let Stigg reconcile.

### Example — upsert entities, then set `parentId` on the entity entitlements

> **This is ONE possible model — ask your customer for theirs; it is not a default.** The `client → project` shape below is illustrative; it teaches the API's shape, not the hierarchy to adopt.

Entities are declared **flat** — `parentId` does **not** belong here. You place each node in the tree when you upsert its entity entitlements (see `entity-entitlements.md`), setting `parentId` to the node's parent.

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
// 1) Entities: flat records, no parentId.
await client.v1.customers.entities.upsert('customer-123', {
  entities: [
    { id: 'client-acme',  entityTypeId: 'client' },
    { id: 'proj-atlas',   entityTypeId: 'project' },
    { id: 'proj-borealis', entityTypeId: 'project' },
  ],
});
// 2) parentId is set on the entity entitlement upsert (see entity-entitlements.md).
```

## Common mistakes

Each mistake — the fix:

- Creating entity types per customer — Types are environment-level; entities are per customer.
- Modeling every org level as an entity level — Only levels that carry limits or need roll-up belong in the tree — 8 levels max.
- Archiving an entity type that still has active entities — The request fails with 400. Check `checkActiveEntities(id)`, archive the entities, then the type.
- Calling `unarchive` — There is none in v1. Re-upsert the archived entity (same id) to restore it.
- Assuming archive is leaves-only — It isn't — archiving a parent orphans its active children. Archive/re-home descendants bottom-up yourself; the platform won't block you. (Leaf-only is the *re-parent* rule.)
- Re-parenting a node with children — Leaf-only — rejected. Move its children to another parent, or detach them with `parentId: null`, first. Archiving them does not help: an archived child still counts.
- Putting `parentId` on the entity upsert (or in `metadata`) — The entity upsert rejects `parentId`, and `metadata` has no structural role. Set `parentId` on the entity entitlement upsert to place/move the node.
- Renaming attribution keys after events flow — Old events keep the old dimensions and won't re-attribute. Treat keys as append-only contracts.
