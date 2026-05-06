# Recipe — Monetize an AI Product with Credits End-to-End

**Goal:** customers pay for AI-token consumption via credits, with a recurring monthly grant on the Pro plan, optional self-serve top-ups, auto-recharge for prepaid users, and customer-facing credit widgets. This is the canonical "OpenAI-style" monetization pattern.

## Ingredients

- A **credit currency** (e.g. `ai-tokens`).
- A **metered feature** (e.g. `tokens_consumed`) with a meter that aggregates from your event stream.
- A **consumption mapping** (or **custom formula**) that converts feature usage to credit deductions.
- **Plans** that include **recurring credit grants** as entitlements.
- **Auto-recharge** for prepaid customers.
- **Credit widgets** in the customer portal.
- **Webhook handlers** for credit lifecycle events.

## Steps

### 1. Define the credit currency (`stigg-credits`)

In Stigg: **Product Catalog → Credits → + Add credit type.** Pick a stable ID (`ai-tokens`), display names ("token" / "tokens").

### 2. Model the metered feature

- Create the metered feature `tokens_consumed`.
- Configure its meter (aggregation, dimensions).

Note: the feature itself only describes what's measured. The conversion to credits (consumption mapping or custom formula) is **set on each plan** — see step 3.

### 3. Add credit entitlements + consumption mapping per plan (`stigg-credits/references/custom-formulas.md` for non-uniform pricing)

For **each plan** that should consume the metered feature for credits:

- Add a credit entitlement (recurring grant of N `ai-tokens` per period, if the plan includes credits).
- Configure the consumption mapping under **Plan → Edit price → Set price → Credit consumption**:
  - Uniform: `1 tokens_consumed → 1 ai-token`.
  - Variable / multi-model: a custom formula like `credits_used = (1.1 × gpt4o_input) + (5 × gpt4o_output)`.

Different plans can use different rates / formulas for the same feature — common when a higher-tier plan unlocks cheaper unit pricing.

For seat-based AI pricing (credits scale with seats), see `stigg-credits/references/seat-pools.md`. For multi-tier plans where a higher tier should also offer additional credits, model an **add-on** with its own credit entitlement and compatibility against the Pro plan.

### 4. Wire usage reporting (`stigg-entitlements/references/usage-reporting.md`)

In your AI-call code path:

```ts
await stigg.reportEvent({
  events: [{
    customerId,
    eventName: 'tokens_consumed',
    idempotencyKey: `${requestId}`,
    dimensions: {
      model: 'gpt-4o',
      input_tokens: 1500,
      output_tokens: 250,
    },
    timestamp: new Date().toISOString(),
  }],
});
```

If using a custom formula, ensure the dimensions referenced in the formula are sent.

### 5. Gate before the AI call (`stigg-entitlements`)

```ts
const ent = await stigg.getMeteredEntitlement({
  customerId,
  featureId: 'tokens_consumed',
  options: { requestedUsage: estimatedTokens },
});
if (!ent.hasAccess) {
  // overdraft is auto-created — but you can still surface "consider topping up"
}
```

> **Don't block on overdraft** unless you mean to. Stigg auto-creates an overdraft grant rather than rejecting the event. Decide in your UX whether to warn or block.

### 6. Configure auto-recharge for prepaid customers (`stigg-credits/references/auto-recharge.md`)

For customers who don't have a recurring grant on a plan (pay-as-you-go), expose the **Credits Auto-Recharge Configuration** widget so they can self-configure threshold + target + monthly spend cap.

### 7. Render credit widgets (`stigg-widgets/references/credit-widgets.md`)

In the customer portal:

- **Credits Balance** — live balance + adjust-credits.
- **Credits Utilization** — used vs available.
- **Credits Usage Chart** — time-series, grouped by feature.
- **Credits Grants Table** — all grants (recurring, prepaid, promotional, overdraft).
- **Credits Auto-Recharge Status / Configuration** — for prepaid users.

### 8. Wire webhook handlers (`stigg-credits` — webhooks section)

At minimum:

- `credit.balance.depleted` → notify the customer + offer top-up.
- `credit.balance.low` → preemptive notification at a configured threshold.
- `automatic_recharge.operation.attempted` → log every auto-recharge.
- `credits.automatic_recharge_limit_exceeded` → notify customer they hit 80/90/100% of cap.
- `automatic_recharge.configuration.changed` → if disabled (e.g. payment failure), surface immediately.

### 9. Connect billing for paid grants (`stigg-credits/references/billing-integration.md`)

For prepaid top-ups and recurring grants on paid plans, connect Stripe (or your billing provider). When money moves, Stigg creates the corresponding paid grant with `cost_basis` set.

### 10. Set up revenue rec exports (`stigg-credits/references/revenue-recognition.md`)

Connect a warehouse via **Integrations → Data warehouse** and use the `credit_grants` and `credit_ledger` exports for Finance.

## Gotchas at the seams

- **Recurring grant cadence vs billing period.** If the plan bills monthly but credits expire weekly, customers get whiplash. Match cadences.
- **Overdraft is always on.** Plan UX around it. Don't try to disable.
- **Custom formulas depend on event dimensions.** Renaming a dimension breaks the formula silently. Coordinate dimension changes carefully.
- **Auto-recharge requires a payment method.** Check + surface failures via the `automatic_recharge.configuration.changed` webhook.
- **Promotional credits have zero `cost_basis`.** Setting it nonzero pollutes revenue rec.
- **Multi-currency products** (e.g. `ai-tokens` + `storage-credits`): each widget needs `creditCurrencyId` passed; multi-active products also need `resourceId`.

## Verify in production

- New Pro customer → recurring grant appears in the credit pool.
- AI call → usage event ingested → ledger updated → balance decremented.
- Customer hits zero → overdraft grant auto-created → next AI call still goes through.
- Customer purchases credits → overdraft auto-settled → new grant has the carried-over consumption.
- Auto-recharge customer crosses threshold → recharge attempted → grant created → webhook fires.
- Finance exports show `cost_basis × consumed` matches recognized revenue period-end.
