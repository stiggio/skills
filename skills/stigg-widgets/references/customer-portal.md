# Customer Portal Widget — Reference

The `CustomerPortal` widget gives customers self-service for their subscription, usage, and billing — drop-in component, opinionated layout, fully themable.

## What's inside

The portal renders four core sections (each is also exposed as a standalone component for modular layouts):

1. **Subscription Overview** — current plan, purchased add-ons, granted promotional entitlements.
2. **Usage** — current usage of subscription's metered features.
3. **Plan Picker** — uses the Pricing Table widget under the hood; auto-reflects catalog changes; lets customers self-serve upgrades / downgrades.
4. **Payment Details** — view + (with Stripe integration) update billing details and payment method via Stripe Billing Portal; view past invoices.

## Minimal usage (React)

```tsx
import { StiggProvider, CustomerPortal, Paywall } from '@stigg/react-sdk';

<StiggProvider apiKey="<STIGG_CLIENT_API_KEY>" customerId="<LOGGED_IN_CUSTOMER_ID>">
  <CustomerPortal
    paywallComponent={<Paywall />}
    theme={{ /* … */ }}
    textOverrides={{ /* … */ }}
    onContactSupport={() => { /* open your support flow */ }}
  />
</StiggProvider>
```

## Modular layout — when the default doesn't fit

If you want to render the sections individually (e.g. you have your own dashboard layout), wrap them in `CustomerPortalProvider`:

```tsx
import {
  CustomerPortalProvider,
  CustomerUsageData,
  PaymentDetailsSection,
  // SubscriptionsOverview, AddonsList, Promotions, InvoicesSection, ...
} from '@stigg/react-sdk';

<CustomerPortalProvider theme={{ /* … */ }} textOverrides={{ /* … */ }}>
  <CustomerUsageData />
  <PaymentDetailsSection />
</CustomerPortalProvider>
```

Storybook variant: `widgets-customer-portal--modular-layout`.

## Stripe-only sections

When Stigg is integrated with Stripe:

- Customers can update billing details + payment method through the Stripe Billing Portal (linked from the Customer Portal).
- Customers can view past invoices through the Stripe Billing Portal.

**Stigg does not store payment methods.** Configure Stripe Billing Portal correctly to avoid surfaces colliding with Stigg's own.

## Custom-priced subscriptions

Customers on **custom plans** can't self-serve upgrade / downgrade through the portal. Handle the `onContactSupport` callback to route them to a sales rep / support channel.

## Multi-tenant — `resourceId`

For products with multiple active subscriptions, pass `resourceId`:

```tsx
<CustomerPortal
  resourceId="workspace-abc"
  paywallComponent={<Paywall resourceId="workspace-abc" />}
/>
```

Pass it to **both** `CustomerPortal` and the `Paywall` slot. Each workspace renders its own portal.

## Combining single-instance + multi-instance products

If your customer has subscriptions on both a single-instance product AND a multi-instance product (e.g. account + per-site), render two portals — one without `resourceId` for the account-level sub, one per resource for the per-site subs.

## Theming & copy

- **Designer:** Stigg app → no-code Widget Designer.
- **Code-level themes:** `theme={...}` prop.
- **Custom CSS classes** for fine targeting:
  - `stigg-payment-details-section-layout` / `-header` / `-title`
  - `stigg-customer-name` / `stigg-customer-email`
  - `stigg-credit-card` / `stigg-credit-card-expiration`
  - `stigg-invoices-section-layout` / `-header` / `-title`
  - `stigg-view-invoice-history-button` / `stigg-edit-payment-details-button`
  - `stigg-user-information-layout`
  - `stigg-customer-portal-usage-section-title`
  - `stigg-entitlement-usage-{featureID}` / `stigg-entitlement-feature-{feature.id}`
  - `stigg-{plan.id}` / `stigg-compatible-addon-{addon.id}`
- **Text overrides** via `textOverrides={...}` — keys include `manageSubscription`, `usageTabTitle`, `addonsTabTitle`, `promotionsTabTitle`, `editBilling`, `invoicesTitle`, `viewInvoiceHistory`, `editPaymentDetails`, `paywallSectionTitle`, `cancelScheduledUpdatesButtonTitle`, etc.

Always re-fetch the docs for the current full key list — it grows.

## Cache refresh

After your code mutates the customer (provisioning, updating, granting promotional entitlements), the portal may briefly show stale data. Use the SDK's cache-refresh hook — see the React SDK page on cache refresh.

## Common mistakes

| Mistake | Fix |
|---|---|
| Wiring custom UI for plan / sub display alongside the portal | Defeats the auto-sync. Use the modular layout if the opinionated one doesn't fit. |
| Forgetting `paywallComponent` | The Plan Picker section won't render. Pass it explicitly. |
| Skipping `onContactSupport` for custom-plan customers | They click and nothing happens. Always handle it for accounts that may have custom plans. |
| Writing CSS targeting `<div>` selectors | Use the `stigg-*` namespaced classes; they're stable. |
| Not configuring Stripe Billing Portal | Customers see broken or missing billing-detail links. Configure it before going live. |
| Rendering the portal at the wrong scope on multi-active products | Always pass `resourceId` to both the portal and the embedded paywall. |
