# Custom Credit Consumption Formulas — Reference

When usage is **not uniform** — different workloads consume credits at different rates — define a custom formula instead of a fixed conversion rate.

## When to use

- Multi-model LLM workflows (cheap model + expensive model in one call).
- Multi-input processing (one event represents documents + emails + images).
- AI / media / data transforms with variable intensity.
- Any time you need more accuracy than fixed per-unit pricing.

If your usage is uniform (`1 API call = 1 credit`), use a regular **consumption mapping** instead — see `stigg-pricing-modeling/references/credits-modeling.md`.

## How it works

- Formulas are **configured on the plan**, per metered feature — different plans can use different formulas (or no formula / a flat conversion rate) for the same feature.
- The formula is a **mathematical expression** referencing the event's **dimensions** (its metadata fields).
- Stigg evaluates the formula at event ingestion and deducts the result in real time.
- Every calculation is captured in the credit ledger (parameter values + resulting burn).

## Three example shapes

### Multi-input processing

```
credits_used = (a × documents) + (b × emails) + (c × images)
```

Use when one event represents multiple workload types you bill differently.

### Multi-model LLM operations

```
credits_used = (1.1 × model1_tokens) + (1.5 × model2_tokens) + (5 × model3_tokens)
```

Use when a single agent run touches multiple models with different cost weights.

### Uniform token-based deduction

```
credits_used = agent1_tokens + agent2_tokens + agent3_tokens
```

Use when each sub-agent contributes separate token counts but they all cost the same.

## Configuration

In the Stigg app:

1. **Product catalog → Plans** → select a plan → **Edit**.
2. Scroll to the **Price** section → **Edit**.
3. **Set price → Credit consumption.**
4. Pick the metered feature configuration (e.g. "Prototype generations").
5. **Calculation mode → Advanced formula.**
6. Enter the formula in the formula field.
7. **Save changes** → **Review and publish** → set what the changes affect → **Publish changes**.

The formula references the **dimensions on the event payload** — anything you pass on `reportEvent({ dimensions: { ... } })` is in scope.

## Designing dimensions for formulas

Because the formula reads dimension fields, **dimension design now matters for billing accuracy**. Best practices:

- Send all variables you might price on, even if you don't use them today (`model_tokens`, `image_count`, `document_pages`).
- Use stable, descriptive names — they're hard to rename without a migration.
- Numeric fields stay numeric (don't pass `"3"` as a string when you'll multiply by it).
- Document the formula's dependency on each dimension so Finance + Eng can reason about pricing changes.

## Common mistakes

| Mistake | Fix |
|---|---|
| Reaching for custom formulas when a flat rate would do | Custom formulas add cognitive load. Use them only when usage is genuinely non-uniform. |
| Sending dimensions as strings | Numeric inputs stay numeric. `tokens: 1500`, not `tokens: "1500"`. |
| Renaming a dimension referenced by a formula | Breaks the formula silently. Rename in two phases (add new, deprecate old). |
| Hard-coding model rates in your app | Keep rates in the formula, not in code — pricing changes don't require deploys then. |
| Designing the formula in code review without Finance | Formulas drive revenue; Finance should sign off on the math. |
| Treating formula changes as silent | Republishing a plan with a new formula migrates affected subscriptions per the publish migration policy. Coordinate the rollout. |
