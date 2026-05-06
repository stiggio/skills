# Recipe — Freemium With Paid Usage After Quota (the "OpenAI API" Archetype)

**Goal:** customers get a small monthly free quota of a metered resource (API calls, tokens, etc.). When they exceed it, they pay per-unit for additional usage — no subscription tier change, no hard cap. This is the canonical pricing for AI APIs (1k free tokens/mo + $0.001/token after) and many usage-based SaaS.

> **Why this needs its own recipe.** Stigg supports two competing primitives for "free quota + paid above": (1) **credits** (recurring grant + paid credits charge for top-ups), and (2) **metered feature with overages** (included quota + per-unit overage rate). Both work. They have different operational shapes; this recipe documents the choice.

## Pick the primitive

| Pick credits when | Pick metered + overages when |
|---|---|
| Customers will buy more usage in advance (prepaid packs) | Customers pay arrears — bill at end of period |
| Cost-per-unit varies by workload (per-model AI, per-region, etc.) — needs a custom formula | Cost is uniform per unit |
| You want auto-recharge on low balance | Auto-charge above quota is fine; no prepaid-balance UX needed |
| Customers should see a real-time balance widget | Customers see usage on the invoice |
| Stigg + Stripe integration is set up | Either; Stigg + Stripe still needed for paid plans |

For the OpenAI-style "1,000 free tokens/mo + $0.001 per token after" — **either works**. This recipe walks both paths so you can pick.

## Path A — Credits (recommended for AI / variable-cost workloads)

### Steps

1. **Define a credit currency** (`stigg-credits/SKILL.md` "Credit Currency"). Example ID: `ai-tokens`. Singular/plural display: `token` / `tokens`.
2. **Create a metered feature** for the consumption signal (e.g. `tokens_consumed`).
3. **Define the consumption mapping on each plan that should consume the feature for credits** (`stigg-credits/references/custom-formulas.md`). Path: **Plan → Edit price → Set price → Credit consumption**.
   - **Uniform cost:** simple mapping `1 tokens_consumed → 1 ai-token`.
   - **Variable cost (multi-model):** custom formula like `credits_used = (1.1 * gpt4o_input) + (5 * gpt4o_output)`.
   - The same metered feature can be mapped differently on different plans — Pro might offer 2:1 efficiency vs Free's 1:1.
4. **Create a Free plan** with a recurring credit entitlement of 1,000 `ai-tokens` / month.
5. **Configure paid credit purchase** — see `stigg-pricing-modeling/references/charges-and-pricing-models.md` "Credits charge". Customers buy more credits in-app. Auto-recharge keeps balance topped up (`stigg-credits/references/auto-recharge.md`).
6. **Wire usage reporting** in your API path:
   ```ts
   await stigg.reportEvent({
     events: [{
       customerId,
       eventName: 'tokens_consumed',
       idempotencyKey: requestId,
       dimensions: { model, gpt4o_input, gpt4o_output },
     }],
   });
   ```
7. **Don't pre-block on overdraft** — Stigg auto-creates an overdraft grant when balance hits zero rather than rejecting the call (per `stigg-credits/references/consumption.md`). Surface a warning in your UI; let the call through. Charge or auto-recharge settles the deficit.
8. **Render `CreditsBalance` and `CreditsAutoRechargeConfiguration`** in the customer dashboard (`stigg-widgets/references/credit-widgets.md`).
9. **Wire webhooks** for `credit.balance.low`, `credit.balance.depleted`, `automatic_recharge.*` (`stigg-webhooks`).

**Strengths:** real-time balance, multi-model pricing via formulas, prepaid + auto-recharge mechanics, finance gets cost-basis revenue rec.

**Cost:** more catalog setup; customers see "credits" abstraction (which they may or may not love).

## Path B — Metered feature with overages (recommended for uniform-cost APIs)

### Steps

1. **Create a metered feature** (`api_calls` with `count` aggregation).
2. **Create a Free plan** with the metered feature entitlement set to 1,000 `api_calls / month`.
3. **Configure overages** on the Free plan: pay-as-you-go above quota at $0.001 per unit (`stigg-pricing-modeling/references/charges-and-pricing-models.md` "Overages").
4. **Wire usage reporting** the same way:
   ```ts
   await stigg.reportEvent({
     events: [{ customerId, eventName: 'api.request', idempotencyKey: requestId }],
   });
   ```
5. **At billing period end**, Stigg's billing-provider integration (Stripe) generates the invoice with the overage line items. No real-time balance UI needed.
6. **Wire webhooks** for `entitlement.usage_exceeded` (notify the customer they're now in paid-usage territory) and `subscription.spend_limit_exceeded` if you set a `maximumSpend` (`stigg-webhooks`).

**Strengths:** simpler model; customers see usage on the invoice; no prepaid abstraction.

**Cost:** no real-time balance widget; multi-model variable pricing is awkward; no auto-recharge.

## Common to both paths — backend integration

```ts
import Stigg from '@stigg/node-server-sdk';

const stigg = await Stigg.initialize({ apiKey: process.env.STIGG_SERVER_API_KEY! });

// On signup — provision Free
await stigg.provisionCustomer({ customerId, email, name });
await stigg.provisionSubscription({ customerId, planId: 'plan-free' });

// On each request — read effective entitlement (advisory only) + report usage
const ent = await stigg.getMeteredEntitlement({
  customerId,
  featureId: 'api_calls', // or 'tokens_consumed' for the credits path
  options: { requestedUsage: 1 },
});
// Don't block on !hasAccess unless your business model demands a hard stop.

await doTheThing();

await stigg.reportEvent({
  events: [{ customerId, eventName: 'api.request', idempotencyKey }],
});
```

## Common to both — frontend

For the credits path, render the credit-balance widgets in the customer dashboard. For the overages path, surface the customer's current usage via `CustomerUsageData` (from `@stigg/react-sdk`) and let invoices speak for the dollars.

## Gotchas at the seams

- **The Free plan is still a real subscription.** Provision the customer onto `plan-free` at signup; without an active subscription, gating returns fallback values.
- **`requestedUsage` on `getMeteredEntitlement` doesn't enforce** — it just reports `hasAccess` based on a hypothetical. Real enforcement happens at usage-report time (or, for credits, never — overdraft auto-creates).
- **Credits + overages combined is unusual** — pick one. If you've modeled both, you're doubling up.
- **Auto-recharge only fires on credit-balance crossings** — if you're on Path B, auto-recharge doesn't apply; Stripe's invoicing handles paid usage.
- **Migration between paths is non-trivial.** Picking wrong and switching means re-modeling the catalog and re-provisioning. Decide deliberately.

## Verify in production

- **Path A:** new customer → Free subscription → recurring grant appears in pool → AI call → balance decremented → at zero, overdraft grant auto-created → next AI call still works → top-up purchase settles overdraft → balance restored.
- **Path B:** new customer → Free subscription → 1,000 calls succeed → 1,001st call still succeeds (no hard block) → end of period, Stripe invoice includes overage line items at $0.001/call.

If either flow doesn't end-to-end, **don't ship**. Fix in staging.
