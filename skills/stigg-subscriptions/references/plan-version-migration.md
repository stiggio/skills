# Plan-Version Migration — Reference

Moving an **existing Stigg subscription** from plan version *N* to plan version *N+1* after a publish.

> This is **routine** ops — happens whenever the catalog ships a new version.

## Why migration is opt-in

When a plan or add-on is republished, **existing subscribers stay on their current version** by default ("grandfathered"). This protects in-flight customers from surprise changes. To bring them onto the new version, you migrate explicitly — manually per-subscription, or programmatically en masse.

If you don't migrate, you accumulate **SKU sprawl**: customers on dozens of legacy plan versions, each with subtly different entitlements. Migrate periodically.

## Migration mechanism — picked automatically by Stigg

Two mechanisms, chosen by the **plan-type transition**:

| From → To | Mechanism | Why |
|---|---|---|
| Free → Paid | **Provision new subscription** | Paid plans require billable metric values; can't migrate in place. |
| Free → Custom | **Provision new subscription** | Custom plans require variable entitlement values. |
| Paid → Free | **Migrate to latest version** | In-place. |
| Paid → Custom | **Migrate to latest version** | In-place. ⚠ The existing subscription in the **external billing system is not canceled** — only Stigg-side metadata (`stiggEntityId`, `stiggEntityUrl`) is removed. |
| Custom → Paid | **Provision new subscription** | Paid plans require billable metric values. |
| Custom → Free | **Migrate to latest version** | In-place. |

> The "Paid → Custom" caveat is important: your billing-provider subscription stays — that's intentional, since custom plans typically use external billing. Don't expect Stigg to clean up Stripe/Zuora for you.

## How to trigger migration

### UI (single subscription)

1. Open the subscription detail view.
2. `⋮` (dotted menu) → **Migrate to latest**.
3. Choose timing:
   - **Immediately**, or
   - **At the end of the current subscription period.**
4. Confirm.

### Programmatic (bulk / gradual rollout)

Use the **migrate-subscription** mutation / SDK method:

```ts
await stigg.migrateSubscriptionToLatestVersion({
  subscriptionId: 'sub-123',
  // immediate vs end-of-period configurable per the SDK signature
});
```

Common rollout patterns:

- **Cohort-based** — migrate 1% of subs / day, watch metrics, ramp up.
- **Customer-segment-based** — migrate a tier / region first, then the rest.
- **Trigger-based** — migrate when a customer renews, performs a key action, etc.

> Re-fetch the **migrate-subscription-to-latest-plan-version** API page before authoring; the parameter shape (`subscriptionMigrationTime`, etc.) evolves.

## Pricing impact when migrating immediately

If migration is **immediate** and the customer's current price differs from the latest published price, **Stigg auto-charges or credits the prorated amount** for the remainder of the billing period. End-of-period migrations don't trigger immediate financial impact.

## At publish time vs at migration time

Two related but distinct decision points:

- **At publish time** (in `stigg-pricing-modeling`): pick a migration **type** (e.g. `NEW_CUSTOMERS` vs forced) — this is a default policy applied at publish.
- **At migration time** (this file): trigger a specific subscription's migration with `Immediate` vs `End of period`.

Both work together — the publish-time policy controls the *default* behavior; per-subscription migration calls let you override / catch up.

## Don't confuse with…

| Migration | Skill | What it is |
|---|---|---|
| Plan-version migration | **this skill** (`stigg-subscriptions`) | Moving an existing Stigg sub to a new plan version after a publish. Routine. |
| Multi-billing-provider migration | [docs.stigg.io](https://docs.stigg.io) (native-integrations / billing) | Moving a customer between billing providers (e.g. Stripe → Zuora) within Stigg. |

## Common mistakes

| Mistake | Fix |
|---|---|
| Skipping migration after every publish | SKU sprawl. Schedule periodic migrations. |
| Migrating Paid → Custom and expecting Stripe to be canceled | It isn't — only Stigg metadata moves. Coordinate with your billing flow. |
| Migrating immediately on a high-price-delta change without warning customers | Surprise charges. Warn first; consider end-of-period instead. |
| Treating "latest version" as a moving target on a single SDK call | The SDK / API resolves "latest" at call time. If a new version is published mid-rollout, your bulk migration may target it. Lock the target version explicitly when authoring large rollouts. |
