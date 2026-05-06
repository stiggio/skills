# Hand-Off to Implementation — Reference

Once a pricing model is chosen, route to the right implementation skill. **Do not start authoring catalog config in this skill.**

## Hand-off checklist

The recommendation is "done" when the user knows:

1. **Value metric** they meter on.
2. **Plan structure** — free / paid / custom; how many tiers.
3. **Charge model** per plan (flat / per-unit / usage / credits).
4. **Add-ons** (if any) and which plans they extend.
5. **Credits** (if any) and the credit currency.
6. **Trials** (if any) and length.
7. **Price localization** (if any) and which markets.
8. **Self-serve vs sales-led** and any custom-plan tier.

Anything ambiguous: ask one more diagnostic question before handing off.

## Hand-off targets

| What's next | Skill | Why |
|---|---|---|
| Modeling features, plans, addons, products, charges, coupons | `stigg-pricing-modeling` | Catalog implementation |
| Credit currencies, grants, auto-recharge, custom formulas, seat pools | `stigg-credits` | Credit-specific implementation |
| Provisioning subscriptions, trials, lifecycle | `stigg-subscriptions` | Subscription operations |
| Drop-in UI (paywall, customer portal, checkout, credit widgets) | `stigg-widgets` | UI layer |
| Composed end-to-end workflows (freemium / hybrid / credits-monetized AI) | `stigg-recipes` | Multi-step flows |
| Authentication, REST vs GraphQL, SDK selection | `stigg-api` | Integration plumbing |
| Setting up the Stigg MCP for AI-assisted dev | `stigg-mcp` | Agent-driven catalog ops |

## Hand-off pattern

End the advisory turn with a clear directive:

> "Recommended: a freemium plan + a Pro tier with per-seat charge + a Team add-on, with a free credit currency for AI generations metered by tokens with a custom formula across two models.
>
> **Next step:** open `stigg-pricing-modeling` to model the catalog (plans, addons, features) and `stigg-credits` to define the credit currency + formula. When you're ready to drop UI, `stigg-widgets` covers the paywall and credit widgets."

Not:

> "Let me go ahead and create the plan now…" *(implementation belongs to the other skill)*

## Don't accidentally re-do discovery

If the user already gave you the value metric, plan structure, and charge model in their initial prompt, don't run them through diagnostics. Confirm the inferred shape, sanity-check, then hand off.

## Maintain the boundary

This skill has **no implementation steps**. Its terminal action is to point at the right next skill. If you find yourself authoring SDK code or catalog YAML, you've drifted out of scope — stop, summarize the recommendation, and route to the implementation skill.
