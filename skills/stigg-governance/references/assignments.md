# Assignments — Budgets per (Entity, Capability)

An assignment is the budget itself: a usage limit for one entity on one capability. Field names below reflect the surface at the time of writing — **confirm shapes via the Stigg MCP's `search_docs` before calling.**

## Shape

| Field | Meaning |
|---|---|
| entity | The entity being budgeted (any level of the tree — org, department, team, agent). |
| `capability` | What's limited: a **`featureId`** (metered feature) **or** a **`currencyId`** (credit currency). Exactly one. |
| `usageLimit` | The budget for one cadence window. |
| `cadence` | ISO-8601 duration for the reset window — `'P1M'` (monthly), `'P1D'` (daily), `'P1W'` (weekly), `'P1Y'` (yearly). |
| `scopeEntityIds` | Optional — narrows which attributed usage counts against this limit (see below). |

Operations: **list / upsert**. SDK: `client.v1Beta.customers.assignments.*`.

## Feature vs credit-currency capability

- **`featureId`** — budgets a metered feature directly: "team-ip may make 10k `api_calls` per month."
- **`currencyId`** — budgets **credit consumption** in that currency: "dept-legal may burn 50k `ai_tokens` credits per month." This is a Stigg credit currency (see `stigg-credits`), **not** a billing currency — there is no USD/EUR budgeting here.

Pick the credit-currency form when multiple features drain one pool and you want a single budget over all of them; pick the feature form for per-feature caps.

## `scopeEntityIds` — dimensional / high-cardinality scope

By default an assignment covers **all** usage attributed to the entity (including roll-up from descendants). `scopeEntityIds` restricts the counted usage to specific entities — the tool for **high-cardinality or dimensional** cases where creating a hierarchy level per value would blow the 4-level / tree-size budget:

> "Limit dept-legal's usage of *model-gpt5* to 10k credits/month" — model the models as entities of a dimensional entity type, then scope the department's assignment with `scopeEntityIds: ['model-gpt5']` instead of adding a hierarchy level under every department.

## Cadence semantics

- The limit applies per cadence window (`'P1M'` = per month). Usage resets with the window.
- Different assignments on the same entity can carry different cadences (daily cap + monthly cap on the same capability is a valid belt-and-suspenders pattern).

## Example — upsert assignments (via MCP `execute`, which runs SDK code)

```ts
// Shapes are illustrative — confirm exact fields via search_docs first.
await client.v1Beta.customers.assignments.upsert('customer-123', {
  assignments: [
    { entityId: 'dept-legal', currencyId: 'ai_tokens', usageLimit: 50000, cadence: 'P1M' },
    { entityId: 'team-ip',    featureId: 'api_calls',  usageLimit: 10000, cadence: 'P1M' },
  ],
});
```

## Interaction with the hierarchy

Assignments at different levels stack as independent gates: usage attributed to `agent-7` counts against `agent-7`'s, `team-ip`'s, and `dept-legal`'s budgets simultaneously, and **any** exhausted level blocks the leaf (see `enforcement-and-usage.md`).

## Common mistakes

| Mistake | Fix |
|---|---|
| Passing both `featureId` and `currencyId` | A capability is one or the other. |
| Using `scopeEntityIds` for a natural hierarchy level | If the values are few and stable, model them as a tree level; scope is for high-cardinality dimensions. |
| Assuming a child's budget caps the parent | Roll-up goes upward only — a parent limit constrains all descendants, not vice versa. |
| Writing `cadence: 'monthly'` | ISO-8601 durations only — `'P1M'`. |
| Budgeting in USD via `currencyId` | Credit currencies only. Money budgets go through credit cost modeling (`stigg-credits`). |
