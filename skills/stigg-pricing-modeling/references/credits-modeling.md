# Credits — Modeling Reference (Catalog Side)

This file covers the *catalog* shape of credits — recurring grants attached to plans / add-ons as entitlements. **The canonical home for credit-currency definition, runtime ops, ledger, auto-recharge, billing integration, and revenue recognition is `stigg-credits`.** Read that skill first if you're touching credits at all; come back here when you're attaching a recurring grant to a plan or add-on.

## Where credit currencies appear in the catalog

A credit currency (e.g. "AI tokens") is **defined in the catalog** under Product Catalog → Credits — see `stigg-credits` for the definition surface. Within plan / add-on configs, you reference an existing credit currency in two places:

- **Consumption mappings** are **configured on the plan** (and on each add-on) — not on the metered feature, not on the credit currency. Each plan that uses a metered feature decides how that feature consumes credits ("1 API call = 2 AI tokens", or a custom formula). The same metered feature can have different consumption rules across different plans. See `stigg-credits/references/consumption.md` for burn-order rules and `stigg-credits/references/custom-formulas.md` for non-uniform pricing.
- **Credit charges** on plans / add-ons — see `charges-and-pricing-models.md` (the credits charge type).

## Credit grants on plans / add-ons (catalog side)

Plans and add-ons can carry **credit entitlements** that grant a credit allocation to subscribers:

- **Recurring grant** — N credits on each billing period (e.g. "10,000 tokens / month").
- **One-time grant** — N credits at subscription start, never refilled by this entitlement.
- **Add-on credit entitlement** — additive; an add-on of "+5,000 tokens / month" stacks on the plan grant.

## Consumption mapping

**On each plan**, for every metered feature that should deduct credits, define a **consumption mapping**:

- The metered feature.
- The credit currency to deduct from.
- The conversion rate (`1 unit of the feature → N credits`).
- Optional formula for advanced cases (e.g. token-cost differs by model: see custom credit consumption formulas in the docs).

When usage is reported (covered in `stigg-entitlements`), Stigg applies the mapping in real time and deducts from the customer's pool.

## Where the runtime stuff lives

This file is the *catalog* shape. The `stigg-credits` skill covers credit-currency definition, runtime grants, the ledger, consumption logic, custom formulas, auto-recharge, seat-based pools, billing integration, and revenue recognition.

## Common mistakes

| Mistake | Fix |
|---|---|
| Confusing credit currency with billing currency | Credit currency = consumption unit. Billing currency = USD/EUR. They're different. |
| Forgetting consumption mappings on plans | The mapping lives on the **plan** (per metered feature), not on the feature definition. Adding the feature to the catalog isn't enough — every plan that should consume it for credits needs a mapping. |
| Configuring the formula on the metered feature instead of the plan | The metered feature defines what's measured; the **plan** decides how that measurement converts into credits. Same feature, different rates per plan, is supported. |
| Putting "1 credit = $0.001" math in your app | Credit ↔ cost conversion belongs in Stigg (credits charge + consumption mapping), not in your code. |
| Modeling a per-token-model price by piling features | Use **custom credit consumption formulas** to express variable token cost — see the docs page. |
| Treating recurring credit grants on a plan as carry-over | They reset per billing period unless explicitly modeled otherwise. |
