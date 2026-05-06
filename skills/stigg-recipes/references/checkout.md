# Recipe — Add an In-App Checkout Experience

**Goal:** when a customer picks a paid plan (or upgrades), they pay for it inside your app via Stigg's `Checkout` widget without redirecting to an external billing portal.

## Ingredients

- An integrated billing provider (typically Stripe).
- The `Paywall` widget to surface plans.
- The `Checkout` widget to collect payment + complete provisioning.
- A post-checkout state in your app (welcome / refresh entitlements).

## Steps

### 1. Integrate Stripe with Stigg

Set up the Stripe integration via **Integrations → Stripe** in the Stigg app. Verify it's healthy before authoring code.

### 2. Render the Paywall (`stigg-widgets/references/paywall.md`)

```tsx
<Paywall
  onPlanSelected={({ plan, intentionType }) => {
    switch (intentionType) {
      case SubscribeIntentionType.UPGRADE_PLAN:
      case SubscribeIntentionType.DOWNGRADE_PLAN:
        // open <Checkout planId={plan.id} />
        break;
      case SubscribeIntentionType.START_TRIAL:
        // provision a trial via your backend (no checkout needed)
        break;
      case SubscribeIntentionType.REQUEST_CUSTOM_PLAN_ACCESS:
        // route to a "Contact us" form
        break;
    }
  }}
/>
```

### 3. Render Checkout when the customer commits (`stigg-widgets/references/checkout.md`)

```tsx
<Checkout
  planId="<TARGET_PLAN_ID>"
  resourceId="workspace-abc"          // for multi-active products
  onCheckoutCompleted={({ success, error }) => {
    if (success) {
      // refresh customer cache, redirect to "welcome to <plan>" state
    } else {
      // surface the error to the user
    }
  }}
/>
```

### 4. Handle promotion codes (optional)

If your business uses coupons, use the `widgets-checkout--with-promotion-code` Storybook variant.

### 5. Refresh entitlements post-checkout

After `onCheckoutCompleted({ success: true })`, the JS client cache may briefly hold pre-checkout entitlements. Use the SDK's cache-refresh hook before re-rendering parts of the app that depend on entitlements.

## Gotchas at the seams

- **Don't re-call `provisionSubscription` from your backend post-checkout.** Checkout already did it. A second call can duplicate or conflict.
- **`resourceId` is required on multi-active products** — pass it to both the Paywall and the Checkout.
- **Custom plans don't go through Checkout.** They use sales-led / invoice flows. The Paywall fires `REQUEST_CUSTOM_PLAN_ACCESS` for these.
- **Free plans bypass Checkout.** Provision Free directly in your sign-up handler.
- **Failed checkout** (card declined, etc.) — surface the error and let the customer retry. Don't auto-fall-back to a different plan.

## Verify in production

- Paywall → user picks Pro → Checkout opens with the right plan and price.
- Checkout collects payment → Stripe charges → `onCheckoutCompleted({ success: true })`.
- Customer entitlement check returns Pro values (cache refreshed).
- Customer Portal shows the new plan and the most recent invoice.
- Failed-card flow surfaces the right error in the UI.
