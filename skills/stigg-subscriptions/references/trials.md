# Trials — Reference

How free trials work in Stigg. Re-fetch the docs page when authoring; trial mechanics evolve and have a few non-obvious edges.

## Trial types (terminology)

- **Free trial without payment** — customer starts trialing without entering a credit card.
- **Free trial with payment** — customer enters a card; charged at trial end (the classic SaaS pattern).

## Where trial behavior is configured

1. **On the plan** — defines the *default* trial: length, what's granted, conversion behavior.
2. **At provisioning** — override the plan default for a specific customer via `trialOverrideConfiguration` on `provisionSubscription`.
3. **On update** — extend, shorten, or remove the trial mid-flight (via re-`provisionSubscription`, the Stigg app UI, or an SDK wrapper helper — see below).

## Provision a trial

The canonical GraphQL input field is `trialOverrideConfiguration`. SDK abstractions may rename for ergonomics — confirm the field name in your SDK's docs before authoring.

```ts
await stigg.provisionSubscription({
  customerId: 'customer-123',
  planId: 'plan-pro',
  trialOverrideConfiguration: {
    isTrial: true,
    trialEndDate: '2026-06-01T00:00:00Z',     // optional; otherwise plan default applies
    trialEndBehavior: 'CONVERT_TO_PAID',      // or 'CANCEL_SUBSCRIPTION'
  },
});
```

> **Self-serve eligibility:**
>
> - One free trial per customer per product.
> - Customers who already paid for a plan are **not** eligible for another self-serve trial.
> - Admins can override these limitations.

## Convert a trial to paid

Provision a paid subscription on top of the trial; Stigg starts the paid plan **at the end of the trial**:

```ts
await stigg.provisionSubscription({
  customerId: 'customer-123',
  planId: 'plan-pro',
  // payment method must be on file with the integrated billing provider
});
```

The trial subscription remains active until its scheduled end; the new paid sub takes over the moment it ends. The `Trial converted` event fires.

## Trial expired without conversion

Stigg fires the `trial.expired` event. Your app handles win-back UX:

- Send a campaign.
- Surface an in-app paywall.
- Offer a discount via a coupon.

## Trial + active paid subscription on the same product

Yes, supported — under the **generous-entitlement rule**:

> When a customer's effective entitlements come from multiple sources — active subscription, add-ons, trial, promotional entitlements, additional product subscriptions — Stigg picks the **most generous** value across all of them.

The "trial vs paid" case is one instance of this rule. Useful pattern: a customer on Starter trials Enterprise; during the trial, they get Enterprise quotas. After the trial, they revert to Starter unless they convert.

## Trial with add-ons (alongside a paid plan)

You can run a trial on add-ons while the customer keeps their current paid plan. Useful for "try Premium support free for 14 days":

- Provision the trial subscription targeting just the add-ons.
- The base paid plan keeps running.
- At trial end, the add-ons' trial expires; the customer either converts (continues paying) or loses the add-on entitlements.

The `i-want-to/configure-trial-add-on` guide on docs.stigg.io is the canonical write-up.

## Update a trial mid-flight

The GraphQL `updateSubscription` mutation does **not** carry trial fields directly. To extend / shorten / remove a trial, the canonical paths are:

- **Re-call `provisionSubscription`** with a new `trialOverrideConfiguration` (Stigg routes the change through the right mechanism).
- **Use the Stigg app UI**, which surfaces "Trial period — extend, shorten, or remove" on the subscription update screen.
- **SDK wrapper methods** — some SDKs expose a higher-level trial-update helper. Check your SDK's docs for the exact method name.

Don't pass `trial: {...}` on the GraphQL `updateSubscription` mutation directly — that field doesn't exist there.

## Cancellation semantics for trials

- **Cancel-at-end-of-period** is **not** supported on trials. Use **cancel immediately** or **cancel on a specific date** (where allowed).
- The trial cancellation modal pre-selects the right options based on the product's policy.

## Trial events to handle

- `trial.started` — provisioning, analytics tagging.
- `trial.ends_soon` — heads-up campaign trigger.
- `trial.converted` — post-conversion onboarding.
- `trial.expired` — win-back UX.

These are webhook events. Subscribe via **Integrations → Webhooks** in the Stigg app.

## Common mistakes

| Mistake | Fix |
|---|---|
| Letting a customer who already paid start another self-serve trial | Block in your app, or use an admin override. |
| Charging immediately when the customer "converted from trial" | Stigg starts the paid plan at trial end. The first charge is at trial end, not conversion. |
| Treating "active sub + trial" as an error | It's by design. Generous-entitlement rule applies; check the entitlement summary. |
| Forgetting payment method when converting a no-card trial | Conversion fails. Collect a payment method during conversion UX. |
| Using `cancel at end of period` on a trial sub | Not supported. Cancel immediately or use a specific date. |
| Not surfacing `trial.ends_soon` to your CRM | Win-back works much better when timed before expiry, not after. |
| Passing `freeTrial: true` or `trial: { durationDays }` to provision/update | Canonical GraphQL field is `trialOverrideConfiguration: { isTrial, trialEndDate, trialEndBehavior }`. SDKs may rename — verify per runtime. |
