# Plans — Modeling Reference

A **plan** is a tier on a product, packaged with feature entitlements, pricing, and optional trial behavior. It's what a customer subscribes to.

## Three plan types

| Plan type | Use | Payment info required? | Trial allowed? | Variable entitlements? |
|---|---|---|---|---|
| **Free** | Freemium tier; usage-limited; drives upgrades | No | n/a | No |
| **Paid** | Self-serve monetized offering, fixed and/or usage-based | Yes (for paid actions) | Yes | No (use add-ons or in-advance commitment) |
| **Custom** | Sales-led, per-customer pricing, negotiated quotas | Varies | Yes | **Yes** — variable entitlement values per customer |

A product can carry all three plan types in parallel.

## Anatomy of a plan

- **Plan ID** — stable, immutable, used in code (`plan-pro`, `plan-enterprise`).
- **Display name & description** — UI surfaces (paywall, customer portal).
- **Visibility** — public (shown in pricing tables) or hidden (custom / sales-led).
- **Compatibility** — which add-ons can be combined with this plan.
- **Entitlements** — which features and at what configuration. Pulled from the feature catalog.
- **Charges** — the price model (flat / per-unit / usage-based / etc.). See `charges-and-pricing-models.md`.
- **Billing period** — monthly / annual / custom; multiple periods can be defined per plan.
- **Free trial** — optional; trial length, what it grants, conversion terms.
- **Metadata** — arbitrary key/value for integration with your business logic.

## Versioning — drafts and publishes

Plan edits **never modify the live version directly**:

```text
Open draft  →  edit (entitlements / prices / metadata)  →  Publish  →  new version is active
                                                          ↳ pick migration policy
                                                            for existing subs
```

Migration types at publish time include:

- **NEW_CUSTOMERS** — only new subscriptions get the new version. Existing subs stay grandfathered.
- **Forced migration** — applies to existing subscribers per the configured rule (immediate or at end of period).

The exact set of supported migration types evolves — re-fetch the publish-plan page in the docs before authoring the rollout. Migration of a *single* subscription to the latest version (after publish) is covered in `stigg-subscriptions/references/plan-version-migration.md`.

## Free plan specifics

- No payment method required.
- Usually carries restrictive entitlements to drive upgrades.
- Trial is irrelevant (it's already free).
- Can be the default plan a customer is provisioned onto when they sign up.

## Paid plan specifics

- Connects to your billing provider (Stripe / Zuora / etc.) for charge collection.
- Supports billing periods (monthly / annual), trial periods, minimum spend, in-advance commitment.
- Multiple billing-period prices per plan are normal (monthly + annual).
- Multiple currency prices per plan via price localization.

## Custom plan specifics

- Built for sales-led / enterprise deals.
- Supports **variable entitlements** — quotas filled in per-customer at provisioning time.
- Excluded from automated billing system integrations by default.
- Usually invisible in self-serve pricing tables.

## Operations on plans

- **Create** — start with the catalog UI/MCP, define entitlements + charges.
- **Edit (draft)** — open a draft, change anything, publish to a new version.
- **Publish** — make a draft the active version. Pick migration policy.
- **View history** — see prior versions and their migrations.
- **View list** — filter by status (draft, published, archived).
- **Archive** — hide from new subscriptions; existing subs keep their plan.
- **Plan visibility** — control whether the plan appears in pricing tables.
- **Variable entitlement values** (custom plans) — per-customer quota inputs at provisioning.

## Common mistakes

| Mistake | Fix |
|---|---|
| Editing a published plan directly | Always: open a draft, edit, publish. Direct edits aren't a supported flow. |
| Publishing without choosing a migration type | The migration type controls existing-subscriber experience. Don't skip it. |
| Designing custom plans without variable entitlements | The whole point of custom plans is per-customer flexibility. Use variable entitlements. |
| Putting "billing currency" inside a custom credit currency | They're orthogonal. Billing currency is price localization. Credit currency is consumption units. |
| Hiding free plans behind visibility = false to "force a paid funnel" | Use entitlements and trial flow to drive upgrades. Hiding free plans usually hurts top-of-funnel. |
