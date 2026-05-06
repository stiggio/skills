# Add-ons — Modeling Reference

An **add-on** extends a plan with extra capability. Customers buy add-ons on top of their plan; entitlements are **additive** by default.

## Core mechanics

- **Additive entitlements.** A customer's effective entitlement for a feature = plan entitlement + sum of add-on entitlements (multiplied by add-on quantity if multi-instance).
- **Lifecycle is tied to the plan.** When a subscription's plan ends, its add-ons are deactivated.
- **Compatibility is per-plan.** An add-on declares which plans it can attach to.

## Two add-on types

| Type | Use for |
|---|---|
| **Paid add-on** | Self-serve, recurring charge, can appear in pricing tables; supports multiple billing intervals and currencies |
| **Custom add-on** | Negotiated / enterprise; per-customer pricing and entitlements; **must** be paired with a custom plan; excluded from automated billing |

## Anatomy of an add-on

- **Add-on ID** — stable, immutable.
- **Display name & description**.
- **Type** — paid / custom.
- **Visibility** — controls appearance in customer-facing pricing tables.
- **Compatibility** — which plans it can attach to.
- **Entitlements** — feature configurations. **Five flavors** of feature entitlements on add-ons:
  - **Boolean** — grant a feature flag.
  - **Configuration** — bump a numeric or enum value.
  - **Metered** — increase a metered quota.
  - **Credits** — grant a credit allocation (see `credits-modeling.md`).
  - **Entitlement behavior** — special modifiers (re-fetch the docs page for current options).
- **Charges** — flat / per-unit / usage-based / etc., same model as plans (see `charges-and-pricing-models.md`).
- **Billing period** — monthly / annual / custom.
- **Multi-instance behavior** — can the add-on be purchased in multiples?

## Versioning

Like plans: **draft → edit → publish**. Existing subscription add-on instances stay on the prior version until migrated.

## Operations

- **Create** (paid / custom).
- **Edit a draft.**
- **Publish.**
- **Edit a published add-on** — same draft / publish loop.
- **Define add-on price** — see `charges-and-pricing-models.md`.
- **Define compatibility** — declare which plans accept this add-on.
- **Control visibility** — hide / show in customer-facing pricing tables.
- **Archive / unarchive.**
- **View list** of add-ons in the catalog.
- **Storing metadata** for integration with custom business logic.

## Add-on entitlement behavior

For metered features, add-on entitlements are typically **additive** to the plan entitlement (Pro plan gives 10k API calls; "+1k API calls" add-on raises it to 11k; buying two of that add-on raises it to 12k).

For boolean features on add-ons, the add-on grants the feature even if the base plan doesn't. Useful for "Premium support" sold as an add-on across all plans.

For credit entitlements (add-ons can grant credit allocations directly), the runtime mechanics live in the `stigg-credits` skill — this skill covers the modeling shape.

## Common mistakes

| Mistake | Fix |
|---|---|
| Putting a custom add-on on a paid plan | Custom add-ons must pair with custom plans by design. |
| Forgetting to declare compatibility for a new plan | The add-on won't be selectable on subscriptions to that plan until you do. |
| Modeling "extra seats" as a separate add-on per quantity | Use a single multi-instance add-on with per-unit pricing. |
| Editing a published add-on directly | Open a draft → publish, like plans. |
| Treating add-on entitlements as a *replacement* for plan entitlements | They're additive, not overrides. For per-customer overrides, use **promotional entitlements** (covered in `stigg-entitlements`). |
