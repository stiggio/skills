# Paywall / Pricing Table — Reference

The `Paywall` (also referenced as the **Pricing Table** widget) lets customers pick a plan to subscribe to. Use it on public pricing pages, in-app upgrade prompts, or inside the Customer Portal.

## What it auto-handles

- **Plan list** — pulled from your Stigg catalog. Add a plan in Stigg, it shows up. No code change needed.
- **Add-ons** — compatible add-ons appear per plan, ordered by configured visibility.
- **Localization** — picks the right billing currency from `billingCountryCode` / customer context.
- **CTAs** — labeled per intent (start trial / upgrade / downgrade / contact us for custom).

## Minimal usage (React)

```tsx
import { StiggProvider, Paywall, SubscribeIntentionType } from '@stigg/react-sdk';

<StiggProvider apiKey="<STIGG_CLIENT_API_KEY>">
  <Paywall
    onPlanSelected={({ plan, intentionType }) => {
      switch (intentionType) {
        case SubscribeIntentionType.START_TRIAL:
          // provision a trial subscription via your backend
          break;
        case SubscribeIntentionType.REQUEST_CUSTOM_PLAN_ACCESS:
          // redirect to a "Contact us" form
          break;
        case SubscribeIntentionType.UPGRADE_PLAN:
        case SubscribeIntentionType.DOWNGRADE_PLAN:
          // show a Checkout widget or your custom flow
          break;
      }
    }}
  />
</StiggProvider>
```

## Variants worth knowing (from the Storybook index)

- `playground` — the canonical example.
- `with-custom-theme` — theming via the `theme={...}` prop.
- `authenticated-customer` — paywall pre-scoped to a logged-in customer (the `<StiggProvider>` already has `customerId`).
- `upgrade-only` — hides plans below the customer's current tier.
- `async-plan-selection` — `onPlanSelected` returns a promise; useful if your provisioning flow is async.
- `resource-based` — pass `resourceId` for multi-active products.
- `with-addons` — addons rendered inline per plan.
- `loading-state` — what to render while data loads.

Open the Storybook entry for code samples: `https://widgets.stigg.io/?path=/story/widgets-paywall--<variant>`.

## Public pricing page vs in-app paywall

| Use case | Provider config |
|---|---|
| Public pricing page (anonymous visitors) | `<StiggProvider apiKey="<CLIENT_KEY>">` — no `customerId` |
| In-app upgrade prompt (signed-in) | `<StiggProvider apiKey="<CLIENT_KEY>" customerId="<CUSTOMER_ID>">` — paywall scopes to the customer |

## Composition with Customer Portal

```tsx
<CustomerPortal paywallComponent={<Paywall />} />
```

The portal uses the same paywall in its "Plan picker" section. Pass any custom props via the same `Paywall` element you'd use standalone.

## Theming and copy

- **Designer:** Stigg app → no-code Widget Designer.
- **Code:** `theme={...}` prop + custom CSS targeting documented `stigg-*` classes (e.g. `stigg-{plan.id}`, `stigg-compatible-addon-{addon.id}`).

## Common mistakes

| Mistake | Fix |
|---|---|
| Hard-coding plan cards alongside the `Paywall` "to be safe" | Don't — the paywall auto-syncs to your catalog. Mixing breaks consistency. |
| Treating `onPlanSelected` as a navigation hook | It carries `intentionType` — use it to branch. Trial vs upgrade vs custom-plan request are different flows. |
| Skipping `resourceId` on multi-active products | Paywall renders the wrong context. Always pass it. |
| Using the publishable key + a non-public customer ID | Without client-side security hardening, the customer ID is forgeable. Enable HMAC signed customer tokens for production. |
| Not handling `REQUEST_CUSTOM_PLAN_ACCESS` | Customers click "Contact us" and your handler does nothing. Route to a sales-led flow. |
