# Credits Auto-Recharge — Reference

Auto-recharge keeps a customer's prepaid balance from depleting by automatically purchasing more credits when the balance drops below a threshold.

## The rule

> When the customer's credit balance goes **below** a threshold, Stigg triggers an automatic purchase / grant of credits, bringing the balance up to a target amount.

Recharges stop when the configured monthly spend limit is reached, or when auto-recharge is disabled (manually or by the system, e.g. on payment failure).

## Configuration — six settings

| Setting | What it does |
|---|---|
| **When credit balance goes below** | Trigger threshold (e.g. 20 credits). |
| **Then bring credit balance back up to** | Target balance after the recharge (e.g. 50 credits). |
| **Limit total monthly spend to** | Hard cap on auto-recharge spend per spend-limit period. |
| **Spend limit resets every** | How often the spend-limit counter is reset. |
| **Grant expiration** | How long auto-purchased credits stay valid before expiring. |
| **Payment method** | The card / payment method used for all auto-recharge transactions. |

## Worked example

- Threshold: 20 credits
- Target: 50 credits
- When balance drops below 20, Stigg charges the configured payment method enough to bring the pool back up to 50.

## Limits and behavior

If the **monthly spend limit** is reached (or set below the customer's current spend for the period):

- **Auto-recharge is disabled** automatically.
- **No more credits will be recharged** until the limit is increased or the period resets.
- Stigg fires `automatic_recharge.configuration.changed`.

## Webhook events to handle

| Event | Description |
|---|---|
| `automatic_recharge.operation.attempted` | Every attempt — both success and failure. Use to log + alert. |
| `credits.automatic_recharge_limit_exceeded` | Fired when the customer crosses 80%, 90%, 100% of the spend-limit threshold. |
| `automatic_recharge.configuration.changed` | Enabled / disabled, including system-driven (e.g. due to payment failure). |

> **Always re-fetch the live event names** before wiring webhooks — names occasionally evolve.

## When to enable

- **Pay-before-you-use credit models** (similar to OpenAI's prepaid usage).
- Customers should keep flowing without manual top-ups.
- You want a hard spending guardrail (the monthly spend limit).

## When NOT to enable

- Recurring-credits-on-a-plan model — the plan auto-issues the recurring grant; auto-recharge isn't needed.
- B2B custom-priced deals where each top-up should be a sales conversation.
- Untrusted payment methods or fraud-suspected accounts.

## Customer-facing widgets

Two widgets ship out of the box (see `stigg-widgets`):

- **Credits Auto-Recharge Status** — current configuration + spend used in the period.
- **Credits Auto-Recharge Configuration** — settings form for the customer to enable / configure / disable.

## Common mistakes

| Mistake | Fix |
|---|---|
| Setting threshold = target | Stigg recharges immediately every time the threshold is crossed, ad infinitum. Always `target > threshold`. |
| No spend limit | A bug elsewhere in your app can drain the customer's payment method. Always set the limit. |
| Ignoring `automatic_recharge.configuration.changed` | If the system disables auto-recharge (payment failure), the customer's pool starts depleting silently. Surface the change in your UI immediately. |
| Charging via your billing provider directly without telling Stigg | Stigg won't create the grant. Use Stigg's auto-recharge or its top-up API; don't hand-roll. |
| Auto-recharging promotional credit currencies | Promotional should never be auto-purchased. Auto-recharge is for paid currencies. |
