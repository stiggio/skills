# Checkout Widget — Reference

Cart-like in-app payment experience that lets you accept payments without re-implementing pricing logic or migrating between billing providers.

## When to use

- The customer picked a plan from the `Paywall` (intent: `UPGRADE_PLAN` / `DOWNGRADE_PLAN`).
- You want to collect payment + complete the subscription provisioning in one in-app surface.
- You're integrated with Stripe Elements (or a comparable supported provider).

## Minimal usage (React)

```tsx
import { Checkout } from '@stigg/react-sdk';

<Checkout
  planId="<TARGET_PLAN_ID>"
  resourceId="workspace-abc"        // for multi-active products
  onCheckoutCompleted={({ success, error }) => {
    if (success) {
      // route to post-checkout state — refresh entitlements / redirect
    } else {
      // surface the error
    }
  }}
/>
```

## Variants worth knowing (Storybook)

- `widgets-checkout--playground`
- `widgets-checkout--with-promotion-code` — collect a coupon at checkout
- `widgets-checkout--collect-phone-number`
- `widgets-checkout--collect-tax-id`

Re-fetch the docs and the Storybook for the current full prop surface and the latest variants.

## Where Checkout sits in the flow

```text
Paywall  →  onPlanSelected(intent: UPGRADE_PLAN)
            ↓
      <Checkout planId="…" />
            ↓
      onCheckoutCompleted(success: true)
            ↓
   Refresh customer cache, render the new entitlements
```

Don't bypass this flow with hand-rolled Stripe forms unless you have a strong reason — Checkout handles plan changes, proration, and the right Stigg-side calls atomically.

## Payment-method storage

Stigg never stores payment methods. They live in your billing provider (Stripe, etc.). Checkout collects them and hands them to the provider; Stigg references the provider's payment-method ID.

## Common mistakes

| Mistake | Fix |
|---|---|
| Implementing custom Stripe Elements forms in parallel with Checkout | Mix-and-match leads to mismatched state. Use Checkout, or fully hand-roll. |
| Not handling `onCheckoutCompleted({ success: false })` | Customer sees a silent failure. Always surface errors. |
| Forgetting `resourceId` on multi-active products | Subscription provisions to the wrong scope. Always pass it. |
| Trying to use Checkout for free-plan provisioning | Checkout is for paid plans. For free, use `provisionSubscription` directly (no UI needed). |
| Calling `provisionSubscription` from your backend after `onCheckoutCompleted(true)` | Checkout already did it. Calling again will duplicate or conflict. |
| Skipping promotion codes when your business uses them | Use the `with-promotion-code` variant — it surfaces a coupon input customers can use. |
