# Updating Subscriptions — Reference

Update an active subscription after provisioning. **The GraphQL `updateSubscription` mutation has a focused surface** — it does **not** change plans. For plan changes, call `provisionSubscription` again. For trial / spend / coupon changes, the Stigg app UI uses separate paths under the hood; consult the SDK docs for your runtime to find the right method.

> **Search-first rule applies here especially.** SDK signatures evolve, and the SDK methods sometimes wrap multiple GraphQL mutations into one call. Re-fetch the docs page for your runtime before authoring update code.

## What `updateSubscription` directly supports

Per the GraphQL `updateSubscription` mutation:

| Field | Type | Effect |
|---|---|---|
| `subscriptionId` | required | Which subscription to update. |
| `billingPeriod` | optional | `MONTHLY`, `ANNUAL`, etc. |
| `addons` | optional | **Replaces** existing add-on list — pass the new full list. |
| `billableFeatures` | optional | In-advance commitment quantities, per-unit quantities. |
| `priceOverrides` | optional | Custom pricing overrides. |

**Plan changes are not in this surface.** Re-`provisionSubscription` instead.

## Other updates the Stigg app UI supports

The "Updating subscriptions" page in Stigg's UI lists more capabilities than the GraphQL `updateSubscription` mutation:

- **Trial period** — extend / shorten / remove.
- **Spend management** — `minimumSpend`, `maximumSpend`.
- **Discounts** — add / remove / modify applied coupons.
- **Effective date** — immediately / end-of-billing-period / end-of-billing-month.
- **Billing settings** — reset billing cycle (`UNCHANGED` / `NOW`), proration (`ProrateNow` / `ProrateOnNextInvoice` / `NoProration`).

These flow through **other** mutations / SDK methods (e.g. trial overrides via `provisionSubscription`'s `trialOverrideConfiguration`, billing anchor / proration via SDK-level options on the right call). **Do not assume they're parameters on the GraphQL `updateSubscription` mutation.** Search the SDK docs for your runtime.

## Plan changes — use `provisionSubscription`

```ts
await stigg.provisionSubscription({
  customerId: 'customer-123',
  planId: 'plan-pro',           // new plan
  billingPeriod: 'MONTHLY',
  // ...other provisioning fields
});
```

Stigg routes this through the right path for the plan-type transition (in-place migration vs. new subscription). See the migration matrix in `plan-version-migration.md`.

## Scheduled updates — replace, don't queue

If a subscription already has a scheduled update (a future downgrade, scheduled upgrade, etc.) and you submit a **new** update **in the Stigg app**, the new update overrides the scheduled one and applies immediately. The same behavior is generally expected for SDK-driven updates (same backend), but **the docs explicitly state the rule for the UI path** — confirm in your SDK's docs before relying on it programmatically.

If you need queued updates, model the queue in your app and apply them one at a time.

## Cancel a scheduled update

The docs describe a "cancel scheduled update" operation surfaced both in the UI (open the sub → **Cancel update**) and via API/SDK. The exact SDK method name varies by runtime — search the SDK's README for "cancelScheduledUpdate" or "cancel scheduled update" to confirm the spelling for your language. There's also a separate operation for the specific case of a subscription update that's pending payment confirmation (typically named along the lines of `cancelPendingPaymentUpdate`).

## Common update patterns

### Add or change addon quantity

```ts
await stigg.updateSubscription({
  subscriptionId: 'sub-789',
  addons: [{ addonId: 'addon-seats', quantity: 20 }],
});
```

> **`addons` replaces the entire list** — include all add-ons you want to keep.

### Switch billing period (monthly → annual)

```ts
await stigg.updateSubscription({
  subscriptionId: 'sub-789',
  billingPeriod: 'ANNUAL',
});
```

### Change in-advance commitment quantity

```ts
await stigg.updateSubscription({
  subscriptionId: 'sub-789',
  billableFeatures: [{ featureId: 'feat-seats', quantity: 25 }],
});
```

### Upgrade to a different plan (NOT via `updateSubscription`)

```ts
await stigg.provisionSubscription({
  customerId: 'customer-123',
  planId: 'plan-pro',
  // SDK-level proration / billing-cycle-anchor options if your SDK supports them
});
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Passing `planId` to `updateSubscription` | The GraphQL mutation rejects it. Use `provisionSubscription` for plan changes. |
| Passing only the *new* add-on to `updateSubscription` | `addons` replaces the entire list. Pass the full list you want active. |
| Submitting `updateSubscription` to "queue" another change on top of a scheduled update | It replaces. Cancel the scheduled update first if that's not what you want. |
| Forgetting `resourceId` on multi-active products | The update targets the wrong subscription. Always pass it on multi-active products. |
| Authoring `effectiveDate` / `billingCycleAnchor` / `subscriptionProrationBehavior` on the GraphQL `updateSubscription` mutation directly | They aren't on that mutation. Either configure via the SDK abstraction (if your SDK supports it) or re-fetch the docs page to find the right call. |
| Updating trial length in flight without checking conversion logic | Trial extensions can stack with conversion behavior — re-fetch the trial-conversion docs page first. |
