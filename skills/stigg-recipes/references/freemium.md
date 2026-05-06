# Recipe — Implement a Freemium Model

**Goal:** new users land on a Free plan with limited entitlements; they upgrade to Pro by hitting a paywall in-app.

## Ingredients

- A `Free` plan with restrictive entitlements (drives upgrade urgency).
- A `Pro` paid plan with the entitlements people actually want.
- (Optional) A free trial of `Pro` for new signups.
- The `Paywall` widget (or `Pricing Table`) for the upgrade prompt.
- The `Checkout` widget (or your own checkout flow) to collect payment.
- Backend gating that blocks excessive usage on Free and reports usage to Stigg.

## Steps

### 1. Model the catalog (`stigg-pricing-modeling`)

- **Create features** — at least the metered features you'll gate on Free (e.g. `api_calls`, `projects`).
- **Create the Free plan** with restrictive entitlements ("100 API calls / month").
- **Create the Pro plan** with generous entitlements + the charge model from `stigg-pricing-expert` (often flat fee or per-seat). Optionally add a free trial.
- **Publish.**

### 2. Provision new customers onto Free (`stigg-subscriptions`)

In your sign-up handler, call `provisionCustomer` then `provisionSubscription` with the Free plan ID:

```ts
await stigg.provisionCustomer({ customerId, name, email });
await stigg.provisionSubscription({ customerId, planId: 'plan-free' });
```

For self-serve free signups, this is the entire backend cost of "free tier exists".

### 3. Gate features in the app (`stigg-entitlements`)

For every metered feature on the Free → Pro boundary, add a backend gate:

```ts
const ent = await stigg.getMeteredEntitlement({
  customerId,
  featureId: 'api_calls',
  options: { requestedUsage: 1 },
});
if (!ent.hasAccess) throw new HttpError(402, 'Upgrade required');
await stigg.reportEvent({ events: [/* … */] });
```

Make sure cache + fallback are configured for production.

### 4. Render the paywall (`stigg-widgets`)

When the customer hits a limit (or wants to view plans):

```tsx
<Paywall
  onPlanSelected={({ plan, intentionType }) => {
    if (intentionType === SubscribeIntentionType.UPGRADE_PLAN) {
      // open checkout
    }
  }}
/>
```

### 5. Add checkout (`stigg-widgets/references/checkout.md`)

When the user picks a plan from the Paywall, route to:

```tsx
<Checkout
  planId="plan-pro"
  onCheckoutCompleted={({ success }) => {
    if (success) {
      // refresh entitlements; show "welcome to Pro" UX
    }
  }}
/>
```

### 6. (Optional) Customer Portal for self-service downgrade / management

```tsx
<CustomerPortal paywallComponent={<Paywall />} />
```

## Gotchas at the seams

- **Provisioning Free at signup is required.** Without an active subscription, gating uses fallback rules — likely wrong.
- **Cache refresh after upgrade.** When the customer goes from Free → Pro via Checkout, the JS client cache may briefly hold the Free entitlements. Use the cache-refresh hook.
- **Free plans don't need payment.** Make sure the Paywall doesn't push the customer through Checkout for "Get started" on the Free plan — handle that as a no-op or a sign-up prompt.
- **Don't gate the Free signup itself.** New users on Free should be able to explore; gating bites only when they try to *exceed* the Free entitlement.
- **Trial overlap.** If you offer a Pro trial, customers on Free during their trial get the **generous-entitlement** treatment — Pro values win during the trial. Don't try to re-implement this in your code.

## Verify in production

- New signup → customer + Free subscription in Stigg.
- Excessive usage on Free → entitlement check returns `hasAccess: false`.
- Paywall renders the right plans, in the right order.
- Checkout completes → Pro plan active → entitlement check now returns `hasAccess: true`.
- Customer Portal shows the correct plan, usage, and upgrade options.

If any step fails, **don't ship.** Fix in staging first.
