# Provisioning Subscriptions — Reference

Full surface for `provisionSubscription`. Re-fetch the docs page when authoring; field names occasionally evolve.

## Required vs optional — GraphQL mutation surface

The canonical mutation is **`provisionSubscriptionV2`** (the older `provisionSubscription` is still listed in some rate-limit tables but the documented mutation is the V2 form). SDK methods are typically called `provisionSubscription` and translate to V2 internally — **for raw GraphQL queries, target `provisionSubscriptionV2`.**

Input is `ProvisionSubscriptionInput`. SDK abstractions may rename or wrap these for ergonomics — **confirm field names in your SDK's docs**, but expect the underlying mutation to use the names below.

| Field | Required? | Notes |
|---|---|---|
| `customerId` | ✅ | Customer must exist (provision the customer first). |
| `planId` | ✅ | Plan reference ID. |
| `resourceId` | Conditional | **Required** when the product allows multiple active subscriptions. |
| `billingPeriod` | Conditional | `MONTHLY`, `ANNUAL`, etc. — required for paid plans with multiple periods. |
| `addons` | optional | List of `{ addonId, quantity }`. Must be compatible with the plan. |
| `billableFeatures` | optional | Initial quantities for pay-per-unit / in-advance-commitment features. List of `{ featureId, quantity }`. |
| `priceOverrides` | optional | Custom pricing overrides. |
| `startDate` | optional | `DateTime`. Defaults to now. |
| `trialOverrideConfiguration` | optional | `{ isTrial, trialEndDate, trialEndBehavior: 'CANCEL_SUBSCRIPTION' \| 'CONVERT_TO_PAID' }`. Override the plan's default trial. |
| `appliedCoupon` | optional | Apply a coupon. Accepts: `{ couponId }` (catalog coupon), `{ billingCouponId }` (billing-provider coupon ID), `{ discount: { ... } }` (inline ad-hoc one-off — see "Discounts at provisioning" below for fields). The legacy `{ promotionCode }` shape (Stripe-style promo code) was **deprecated in 2024-09 in favor of `billingCouponId`**; still accepted but flag use sites for migration. |
| `entitlements` | optional | For **custom plans** with variable entitlements. Supports both `feature` (`{ featureId, usageLimit, hasUnlimitedUsage }`) and `credit` (`{ customCurrencyId, amount, cadence: 'MONTH' \| 'YEAR' }`) entitlements. |
| `awaitPaymentConfirmation` | optional | Wait for payment before activating. |
| `environmentId` | optional | Environment ID. Usually inferred from the API key. |

### SDK-level options (not on the raw GraphQL mutation)

The Stigg app UI and SDK abstractions also expose:

- **`billingCountryCode`** — required by some flows for price localization.
- **`billingCycleAnchor`** (`SubscriptionBillingCycleAnchor.UNCHANGED` / `NOW`) — controls cycle anchor on the change.
- **`subscriptionProrationBehavior`** (`ProrateNow` / `ProrateOnNextInvoice` / `NoProration`).
- **`paymentBehavior`** — how Stigg handles payment collection on provisioning.
- **`minimumSpend` / `maximumSpend`** — spend management on the resulting subscription.

Where these surface depends on your SDK / runtime. Re-fetch the SDK docs page for your language; they wrap the underlying mutations in different ways.

## Start-date behavior

- **Immediately** — sub becomes active at provisioning. Default.
- **End of current billing period** — when the customer already has an active paid subscription on the same product, the new one starts when the current one ends.
- **Specific date in the past** — back-dated. **The first billing charge happens in the next billing period, not immediately.** Useful when the customer already paid via another channel.
- **Specific date in the future** — sub activates on that date.

## Billing anchor and proration

Supported when Stigg is integrated with Stripe / Zuora / a custom billing solution.

- **Billing anchor:**
  - `UNCHANGED` (default) — keep the current cycle.
  - `NOW` — reset the cycle to the moment of provisioning. Use when an upgrade should start a fresh billing period immediately.
- **Proration behavior:**
  - `ProrateNow` — generate and collect the prorated charge immediately.
  - `ProrateOnNextInvoice` — defer prorations to the next invoice.
  - `NoProration` — disable.

## Trial / paid coexistence rules

Per `(customer, resource)` pair:

- **At most one active trial.** Provisioning a new trial cancels any existing trial for the same product **at the new trial's start date**.
- **At most one active non-trial subscription.** Provisioning a new paid sub cancels the existing non-trial sub at the new sub's start date.
- **Trial → paid upgrade** starts the paid plan at the **end of the trial**.

Self-served limits:

- Customers get only **one** free trial per product.
- Customers who **already paid** for a plan are not eligible for another self-serve free trial.

Admins can override these limitations via the Stigg app.

## Generous-entitlement rule

When effective entitlements come from multiple sources — **active subscription, add-ons, trial, promotional entitlements, additional product subscriptions** — Stigg picks the **most generous** value across all of them. The "trial vs paid" case is one instance; the rule applies any time sources overlap. Verify via the entitlement summary in the Stigg app.

## Multi-instance (multiple subscriptions per customer)

Set on the **product** (not the plan) at product creation; once any subscription is provisioned on the product, the setting is immutable.

```ts
await stigg.provisionSubscription({
  customerId: 'customer-123',
  resourceId: 'workspace-abc',     // required for products with multiple active subscriptions
  planId: 'plan-team',
  billingPeriod: 'MONTHLY',
});
```

**Pass `resourceId` everywhere** — provision, update, entitlement check, usage report, paywall widget. Without it, calls attribute to the customer-level scope, which is usually wrong in multi-tenant setups.

## Discounts at provisioning

`appliedCoupon` on `provisionSubscription` / `previewSubscription` / `updateSubscription` accepts:

- **`{ couponId }`** — apply a reusable Stigg catalog coupon.
- **`{ billingCouponId }`** — apply a coupon defined in your billing provider (Stripe coupon ID, etc.) — useful when finance manages discounts in the billing system. **This is the recommended shape for billing-provider discounts going forward** (it replaced the deprecated `promotionCode` form in 2024-09).
- **`{ discount: { ... } }`** — inline ad-hoc subscription-specific discount that **doesn't appear in the catalog**. The `SubscriptionCouponDiscountInput` fields include `name`, `description`, `percentOff`, `amountsOff`, `durationInMonths` (verify the full set per the SDK changelog). Use for one-off negotiated discounts that you don't want exposed as reusable.
- **`{ promotionCode }`** — *deprecated 2024-09*; still accepted but migrate use sites to `billingCouponId`.

Re-fetch the docs page for the exact shape of each — Stigg's billing-integration setup affects which forms are accepted.

## Payment behavior

- `automaticallyUsePaymentMethodOnFile` — required for paid plans when you expect immediate billing.
- Generate an invoice — useful for wire transfers, payment links, manual collection.
- Don't use Stigg for payments — Stigg manages entitlements only; you handle billing externally.

> **Stigg never stores payment methods.** They live in the integrated billing provider (Stripe / Zuora / etc.).

## Common mistakes

| Mistake | Fix |
|---|---|
| Provisioning before creating the customer | Provision the customer first; or use the upsert endpoint that does both. |
| Setting `startDate` in the past expecting an immediate charge | Back-dated subs charge from the **next** billing period. |
| Passing add-ons that aren't compatible with the plan | Compatibility is declared on the add-on. Confirm the plan is in its compatibility list. |
| Using `freeTrial: true` as a parameter | The GraphQL mutation uses `trialOverrideConfiguration: { isTrial: true, ... }`. SDK abstractions may expose simpler shapes — check your SDK. |
| Self-serve trial for a customer who already paid | Not eligible. Either skip the trial, or use an admin override. |
| Treating a trial sub as cancellable end-of-period | Not applicable — trials cancel immediately or via the trial-expired path. |
| Confusing the "Updating subscriptions" UI capabilities with what `updateSubscription` accepts | The UI surfaces more than the GraphQL `updateSubscription` mutation — see `updates.md`. |
