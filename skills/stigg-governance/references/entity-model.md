# Entity Model — Entity Types and Entities

The structural half of governance: what kinds of units exist (entity types) and which units a given customer actually has (entities). Field names and route paths below reflect the surface at the time of writing — the v1-beta API moves, so **confirm shapes via the Stigg MCP's `search_docs` before calling.**

## Entity Types (per environment)

An entity type declares a category of sub-customer unit and how usage events map onto it.

| Aspect | Rule |
|---|---|
| Scope | Environment-level — shared across customers. |
| `attributionKeys` | **Max 2** event-dimension keys. When a usage event carries these dimensions, its usage attributes to the matching entity. |
| Operations | **List**, **upsert** — upsert is a **bulk `PUT`**: idempotent create-or-update, safe to re-run from provisioning scripts. |
| SDK | `client.v1Beta.entityTypes.*` |

### Designing attribution keys

Attribution keys are the join between your event stream and the governance tree — get them right **before** reporting usage:

- Pick dimensions your app already emits (or can start emitting) on every relevant event: `departmentId`, `teamId`, `agentId`, `workspaceId`.
- Two keys max per type. If you're tempted to add a third, that's usually a sign one "key" is really a dimensional scope — model it with `scopeEntityIds` on the assignment instead (see `assignments.md`).
- There is no retroactive attribution. Events reported before the dimensions existed never attribute.

## Entities (per customer)

Entities are the instances: *this* customer's departments, teams, and agents.

| Aspect | Rule |
|---|---|
| Scope | Per customer. |
| Hierarchy | `parentId` links entities into a tree — **up to 4 levels** (e.g. org → department → team → agent). |
| Operations | **List / get / upsert (bulk `PUT`) / archive / unarchive**. |
| Archive | **Leaves only.** An entity with children cannot be archived — re-parent or archive the children first. Unarchive restores it. |
| SDK | `client.v1Beta.customers.entities.*` |

### Hierarchy guidance

- Usage attributed to an entity also attributes to **every ancestor** — the tree is a roll-up structure, not just an org chart. Design levels around where you want budgets, not around your full org taxonomy.
- The 4-level cap is a hard limit. If your org model is deeper, flatten the levels that don't carry budgets.
- Bulk `PUT` upsert makes the entity tree declarative: your provisioning code can push the desired tree on every sync and let Stigg reconcile.

### Example — upsert a small tree (via MCP `execute`, which runs SDK code)

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
await client.v1Beta.customers.entities.upsert('customer-123', {
  entities: [
    { id: 'dept-legal',  entityTypeId: 'department' },
    { id: 'team-ip',     entityTypeId: 'team', parentId: 'dept-legal' },
    { id: 'agent-7',     entityTypeId: 'agent', parentId: 'team-ip' },
  ],
});
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Creating entity types per customer | Types are environment-level; entities are per customer. |
| Modeling every org level as an entity level | Only levels that carry budgets or need roll-up belong in the tree — 4 levels max. |
| Archiving a parent to "clean up" a subtree | Archive walks bottom-up, leaves only. |
| Renaming attribution keys after events flow | Old events keep the old dimensions and won't re-attribute. Treat keys as append-only contracts. |
