# Recipe — Configure Hybrid Pricing

**Goal:** a single plan that combines multiple charge models (e.g. flat base + per-seat + usage-based overage). This is the typical shape for B2B SaaS that wants predictable revenue plus scale-with-usage upside.

## A canonical hybrid plan

```text
Pro plan:
  • Flat fee:         $49 / month                ← predictable base
  • Per-unit:         $10 / seat / month         ← scales with team
  • Usage-based:      $0.001 / API call          ← scales with consumption
  • In-advance commitment: 10,000 API calls
  • Pay-as-you-go overage: $0.001 per call above 10k
  • Minimum spend:    $100 / month               ← floor
```

## Steps

### 1. Confirm the model (`stigg-pricing-expert`)

Before authoring config, pass the recommendation through `stigg-pricing-expert` (sanity-check at median + top-decile usage). Hybrid is only worth it when **two value metrics genuinely matter** — don't reach for it because you can.

### 2. Model the catalog (`stigg-pricing-modeling/references/charges-and-pricing-models.md`)

In Stigg:

- **Create the metered feature** (`api_calls`) and the **configuration feature** (`seats`).
- **Create the Pro plan.**
- Add **multiple charges** to the plan:
  - Flat fee charge.
  - Per-unit charge linked to `seats`.
  - Usage-based charge linked to `api_calls`.
- Configure the **in-advance commitment** quantity (10k API calls included).
- Add an **overage** for `api_calls` above the included quota.
- Set a **minimum spend** floor.
- **Publish.**

### 3. Provision subscriptions with the right inputs (`stigg-subscriptions/references/provisioning.md`)

```ts
await stigg.provisionSubscription({
  customerId: 'customer-123',
  planId: 'plan-pro',
  billingPeriod: 'MONTHLY',
  // billableFeatures: provide initial quantities for in-advance commitment / per-unit features
  billableFeatures: [
    { featureId: 'seats', quantity: 5 },
    // api_calls in-advance commitment is on the plan; quantity is implicit unless the customer overrides
  ],
});
```

### 4. Gate at runtime (`stigg-entitlements`)

Both metered features need entitlement checks. Per request:

```ts
const ent = await stigg.getMeteredEntitlement({
  customerId,
  featureId: 'api_calls',
  options: { requestedUsage: 1 },
});
if (!ent.hasAccess) throw new HttpError(402, 'Quota exceeded');
await stigg.reportEvent({ events: [/* api.request event */] });
```

`hasAccess` returns `true` even into the overage range — it's just billed. Block only if you've configured a hard cap.

### 5. Update seat count over time (`stigg-subscriptions/references/updates.md`)

When a customer adds seats:

```ts
await stigg.updateSubscription({
  subscriptionId,
  billableFeatures: [{ featureId: 'seats', quantity: 6 }],
});
```

`updateSubscription` accepts `billableFeatures` directly. Don't try to do this through `provisionSubscription` (that would re-provision, not update).

### 6. Render the customer portal (`stigg-widgets/references/customer-portal.md`)

The portal shows the plan, current seat count, current API-call usage, and the next invoice projection. Customers self-serve seat changes through the Plan Picker / Pricing Table embedded in the portal.

## Gotchas at the seams

- **Overage requires the right charge config.** Setting up an in-advance commitment without an overage means usage above the quota is silently uncounted (or blocked, depending on entitlement settings). Configure the overage explicitly.
- **Minimum spend interaction with overage.** The floor pulls the bill up to the minimum *before* overage; overage piles on top. Re-fetch the docs page on charge composition for the canonical accounting.
- **Volume vs graduated tiers.** Hybrid plans often have tiered overage rates. Volume = one price for the whole; graduated = layered prices per tier. Pick deliberately.
- **Customers can't preview overages from a static price card.** Use `previewSubscription` to project the bill at expected usage and show the customer.

## Verify in production

- Create a customer at `seats=5`, simulate 5,000 API calls (well within commitment) → bill = base + 5×per-seat. No overage.
- Same customer at 15,000 API calls → bill includes 5,000 of overage at the configured rate.
- Same customer at 200 API calls (way under) → bill hits the minimum spend floor.
- Customer adds a seat → next invoice shows the new per-seat charge.
- `previewSubscription` returns numbers consistent with what actually invoices.
