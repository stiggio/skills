# Credit Widgets — Reference

Six widgets ship for credit-based pricing. They live under **Widgets → Credits** in the Stigg app and as React components (with vanilla-JS / Vue / Embed equivalents — see docs.stigg.io).

## The six widgets

| Widget | Shows | Customer can… |
|---|---|---|
| **Credits Balance** | Live balance for the selected credit type, plus the auto-recharge control | Top up via "Adjust credit balance" |
| **Credits Utilization** | Used vs. available (progress bar + count) | (read-only — quick "used / total" snapshot) |
| **Credits Usage Chart** | Time-series chart of credit consumption, grouped by feature | Hover for daily breakdown; pick a date range |
| **Credits Grants Table** | All credit blocks with status and remaining amounts | View promotional vs purchased, expirations, adjustments |
| **Credits Auto-Recharge Status** | Current auto-recharge configuration + monthly spend status | (read-only — see the live state) |
| **Credits Auto-Recharge Configuration** | Settings form for auto-recharge | Enable / configure / disable auto-recharge |

## Where to render them

Inside the Customer Portal (most common) — use the modular layout and drop in the credit widgets where they fit:

```tsx
import { CustomerPortalProvider } from '@stigg/react-sdk';
import {
  CreditsBalance,
  CreditsUtilization,
  CreditsUsageChart,
  CreditsGrantsTable,
  CreditsAutoRechargeStatus,
  CreditsAutoRechargeConfiguration,
} from '@stigg/react-sdk';
// (verify component names against the React SDK page — names occasionally evolve)

<CustomerPortalProvider>
  <CreditsBalance creditCurrencyId="ai-tokens" />
  <CreditsUtilization creditCurrencyId="ai-tokens" />
  <CreditsUsageChart creditCurrencyId="ai-tokens" />
</CustomerPortalProvider>
```

## Storybook

Variants live under:

- `widgets-credits-credit-balance--*` (8 stories)
- `widgets-credits-credit-grants--*` (6 stories)
- `widgets-credits-credit-usage-chart--*` (5 stories)
- `widgets-credits-credit-utilization--*` (9 stories)
- `widgets-credits-automatic-recharge--*` (8 stories) — **a single Storybook component** with multiple stories (e.g. `--active-with-spend-limit`, `--error-state`). The "Auto-Recharge Status" / "Auto-Recharge Configuration" split appears on the docs but not as separate Storybook IDs.

Open `https://widgets.stigg.io/index.json` for the live list, then the Storybook UI for prop tables.

## Per-currency rendering

If a customer has multiple credit currencies (e.g. "AI tokens" + "Storage credits"), pass `creditCurrencyId` per widget instance. Each currency has its own pool, balance, and ledger.

## Multi-tenant — `resourceId`

Pass `resourceId` for products with multiple active subscriptions. Each resource has its own pool (when the credit entitlement is scoped per-resource).

## Auto-recharge UI patterns

- **Customers should see** `CreditsAutoRechargeStatus` for transparency — current threshold, target, monthly spend.
- **Customers configure** via `CreditsAutoRechargeConfiguration` — picks payment method, sets threshold / target / monthly limit.
- Listen for `automatic_recharge.configuration.changed` webhooks to keep your in-app UI in sync if the system disables auto-recharge (e.g. on payment failure).

## Common mistakes

| Mistake | Fix |
|---|---|
| Hand-rolling a credit-balance UI | Stigg's widget syncs in real time and includes the auto-recharge control. Don't reinvent. |
| Showing only the balance, hiding overdraft | Customers in deficit deserve to see it. Surface overdraft state explicitly. |
| Forgetting `creditCurrencyId` on multi-currency products | Widgets default to a currency arbitrarily; users see the wrong pool. Always pass it. |
| Building a custom usage chart from raw events | The Credits Usage Chart already groups by feature and respects credits cost basis. Use it. |
| Treating the Grants Table as a writable surface | Most rows are read-only system data. Promotional / manual grants can be voided from the Stigg app's Customers screen — surface a deep-link if customers need to escalate. |
