# Charges and Pricing Models — Modeling Reference

A plan or add-on can carry one or more **charges**. Each charge has a **billing model**, a **billing period**, and an **amount** (or amounts, for tiered).

## Billing models

| Model | What it bills | Example |
|---|---|---|
| **Flat fee** | Fixed amount per billing period | $49 / month |
| **Per-unit** | Amount × number of units selected at provisioning | $10 / seat / month |
| **Usage-based — pay-as-you-go** | Amount × consumption of a metered feature, at end of period | $0.001 / API call |
| **Usage-based — in-advance commitment** | Customer commits to a quantity up front; consumes against it | "10k API calls/month included, billed up front" |
| **Minimum spend** | Floor on the bill regardless of usage | "$100/month minimum, on-demand usage applied first" |
| **Credits charge** | Bills the customer for credits that fund their pool. Three common shapes — see "Credits charge — concrete shapes" below for the configuration. | "$10 buys 10,000 credits"; "Pro plan: 50,000 tokens / month included" |

## Credits charge — concrete shapes

A **credits charge** wires a price (in your billing currency) to credits in a custom credit currency. Three common configurations, each modeled differently in the catalog:

### 1. Recurring credit grant on a paid plan

The plan carries a flat fee + a recurring credit entitlement.

- **Plan:** Pro, $20 / month base charge.
- **Credit entitlement on the plan:** 50,000 `ai-tokens` per month (recurring grant; resets each billing period).
- **No separate credits charge** — the credits are *included* in the flat fee.
- Set this up in the Stigg app: **Plans → Pro → Edit → Entitlements → add credit entitlement → Recurring grant, 50,000, monthly cadence.**
- Customers see "50,000 tokens / month" on the pricing table.

### 2. Prepaid credit pack (top-up)

Customer buys credits up front, separate from any subscription.

- Modeled as a **plan or add-on** with a **flat fee** charge that issues a one-time credit grant.
- **Plan:** "10k Token Pack", $10 one-time.
- **Credit entitlement:** 10,000 `ai-tokens`, **one-time grant** (not recurring).
- Customers buy through the Checkout widget or a payment link.
- Pairs well with auto-recharge — see `stigg-credits/references/auto-recharge.md`.

### 3. Pay-per-credit (overage / variable)

Customers consume past their grant; you charge per credit consumed above quota.

- This is **not** a credits-charge primitive directly — it's the **overage** mechanism on a metered feature, layered with the credits consumption mapping.
- Configure: metered feature `tokens_consumed` mapped to `ai-tokens` (1:1 or via custom formula); overage rate of $0.001 per credit consumed beyond the included grant.
- Stigg auto-creates an **overdraft grant** when balance hits zero (per `stigg-credits/references/consumption.md`); your billing-provider invoice picks up the overage at period end.

### Picking between #1 and #2

- **#1 (recurring on plan)** is the default for "subscribe to access N credits / month" — predictable revenue, no top-up flow.
- **#2 (prepaid pack)** is the default for "pay-as-you-go from a balance you fund yourself" (OpenAI-style).
- **#1 + #2 combined** is common: subscription gives a base monthly grant; customers can also top up.
- **#1 + #3 combined** = "subscription with paid overage" — see `stigg-recipes/references/freemium-with-paid-usage.md`.

### Common credits-charge mistakes

| Mistake | Fix |
|---|---|
| Modeling "$0.001 per credit" as a separate charge type | It's not a primitive — it's a metered-feature overage. See `freemium-with-paid-usage.md` Path A. |
| Setting recurring grant cadence ≠ billing period | Customers expect "monthly credits" to align with the monthly bill. |
| Forgetting auto-recharge requires Stripe | See `stigg-credits/references/auto-recharge.md`. Auto-recharge is a customer-level config, not a plan attribute. |
| Promotional credits with non-zero `cost_basis` | Pollutes revenue rec. Promotional must be zero. |

## Tiered pricing variants

For per-unit and usage-based:

- **Volume tiers** — the unit price for the *whole* quantity changes once you cross a threshold ("0–1000 @ $0.001, 1001+ @ $0.0008").
- **Graduated tiers** — different unit prices apply to *units within each tier* ("first 1000 @ $0.001, next 9000 @ $0.0008").
- **Per-unit volume** — the price per unit is set by the volume tier reached, but applied to every unit equally.

Volume vs graduated is a common source of confusion — volume picks **one** price for the whole, graduated layers **multiple** prices.

## Overages

For metered features bundled into a plan (e.g. "10k API calls included"), **overages** define what happens when the customer exceeds the quota:

- **Pay-as-you-go overage** — bill for excess usage at a per-unit rate.
- **Block at quota** — deny further usage (a runtime entitlement decision; modeled in the catalog with no overage).
- **Tiered overages** — same volume / graduated mechanics applied to the overage portion.

The full overage model space (`how-overages-work`, `notifications`, `supported-models`, `defining-overages`) is a section in the docs — re-fetch when authoring.

## Billing periods

A single plan / add-on can carry **multiple billing periods** in parallel — typically monthly and annual — each with its own price. Customers pick the period at provisioning. Annual prices are typically discounted vs the monthly equivalent.

## Multi-currency (price localization)

Each charge can carry prices in multiple billing currencies. The customer's billing country code at provisioning resolves to a currency. **Custom credit currencies are unrelated** — they're a consumption unit, not a billing unit. See `price-localization.md`.

## Combining charges

A single plan often has *several* charges:

```
Pro plan:
  • Flat fee:         $49 / month
  • Per-unit:         $10 / seat / month
  • Usage-based:      $0.001 / API call (pay-as-you-go above 10k included)
  • Minimum spend:    $100 / month floor
```

When combining, the order of operations matters for the bill:

1. Flat + per-unit charges accumulate.
2. Included quotas (in-advance commitment) consume usage first.
3. Minimum spend pulls the bill up to the floor.
4. Overages add on top.

The right docs page (Plans → Defining the plans price → Paid → ...) carries the canonical accounting rules. Re-fetch when modeling combined charges.

## Free trials on charges

Trials apply to paid plans (and custom plans). Trial behavior:

- Trial length (days / weeks).
- What's billed during the trial (typically nothing).
- Conversion: at trial end, convert to paid using the configured charge model.
- Trial extensions and re-trials: admin overrides are supported, self-serve typically isn't.

## Modeling-time vs runtime

This file is about the **catalog shape** — what charges look like in the model. The runtime concerns (proration, billing-cycle anchor, payment method handling) live in `stigg-subscriptions`.

## Common mistakes

| Mistake | Fix |
|---|---|
| Multiplying amounts by 100 because "Stripe convention" | Stigg amounts are in **full dollars**, not cents. `49` = $49.00, not $0.49. Never multiply for Stigg. |
| Confusing volume and graduated tiers | Volume = one price for the whole. Graduated = layered prices. |
| Forgetting the minimum spend floor when modeling pay-as-you-go | Without a floor, low-usage customers pay near $0. Decide explicitly. |
| Modeling "$X included, then per-call" as a separate add-on | Use **in-advance commitment** + **pay-as-you-go overage** on the same plan. |
| Setting the same price across all currencies "for simplicity" | Currency conversion is not a 1:1 multiplier; localize prices to perceived value per region. |
| Annual prices that are exactly 12× monthly | Customers expect a discount for annual commitment — model it explicitly. |
| Putting the billing period selection in your app instead of in Stigg | Stigg models multiple billing periods natively; surface them via the paywall widget. |
