# Price Localization — Modeling Reference

**Price localization** = charging different prices in different billing currencies / regions. Independent of the pricing model on a plan.

> Distinct from **custom credit currency** — that's a consumption unit, not a billing unit. See `credits-modeling.md`.

## How it works

1. Define localized prices on a plan / add-on charge: `$49 USD`, `€45 EUR`, `¥6800 JPY`.
2. At provisioning, the customer's **billing country code** resolves to a currency.
3. Pricing tables (the paywall widget) show the localized price.
4. Invoices and billing-provider charges are in the localized currency.

## Two configuration paths

- **No-code configuration in the Stigg app** — UI-driven; useful for non-engineering owners.
- **Codebase integration** — programmatic via SDK / REST.

Both produce the same outcome. Pick by who owns pricing (product/RevOps vs engineering).

## What localization changes — and what it doesn't

| Changes | Doesn't change |
|---|---|
| The price the customer sees on the pricing table | The plan's structure (features / entitlements) |
| The currency on the invoice | The catalog ID of the plan |
| The amount the billing provider charges | Custom credit currency (orthogonal) |

Same plan, same entitlements, different price per currency.

## Edge cases worth knowing

- **Customer changes country.** Localized price applies at provisioning; subsequent country changes don't auto-reprice an existing subscription. Re-provision or migrate to handle this.
- **Pricing table doesn't reflect a localization change.** A frequent FAQ — usually a caching / refresh issue. Re-fetch the FAQ page on price localization for current diagnosis.
- **Custom plans with localized prices.** Supported, but the per-customer pricing model can override the localized list price.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating EUR as "USD × 0.92" | Localize to perceived value per region, not to FX rates. |
| Using credit currency as a substitute for billing localization | Different mechanisms — use **price localization** for billing currency, **credit currency** for consumption units. |
| Forgetting to localize add-on prices | Localize on every charge (plans + add-ons + overage rates). Mismatched localization confuses customers. |
| Hard-coding the displayed price in your app | Use the paywall widget — it picks the right localized price automatically. |
