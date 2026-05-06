# Coupons — Modeling Reference

A **coupon** is a discount instrument applied to a customer or to a specific subscription.

## Discount types

- **Percentage-based** — e.g. "20% off".
- **Fixed amount** — e.g. "$10 off / month".

## Duration

- **Limited duration** — applies for N billing cycles, then expires.
- **Open-ended** — applies until removed.

## Scope of application

- **Customer-level** — applies to all active subscriptions for that customer.
- **Subscription-level** — applies only to the specified subscription. Useful for "discount on the Pro upgrade" without affecting a separate add-on subscription.

## Operations

- **Create** a coupon (catalog-wide, reusable).
- **Edit details** — name, percentage / amount, duration, metadata.
- **Apply** to a customer or subscription via API/UI.
- **Storing metadata** — for integration with custom flows.
- **Archive** — hides from new applications; existing applied coupons keep working.
- **View list.**

## Ad-hoc subscription coupons

When provisioning a subscription, you can **customize a coupon inline** — Stigg creates an ad-hoc subscription-specific coupon that does **not** appear in the product catalog. Use this for one-off negotiated discounts that you don't want to expose as reusable.

## Billing-provider integration

When integrated with a billing provider (Stripe / Zuora), coupon names appear on customer invoices. Pick coupon names that are customer-readable.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hard-coding discount logic in your app | Use Stigg coupons instead — they show on invoices and propagate through billing integrations. |
| Reusing one coupon ID for multiple campaigns | Use distinct coupons per campaign so analytics can attribute correctly. |
| Applying a customer-level coupon when only one subscription should be discounted | Use a subscription-level apply. |
| Open-ended coupons forgotten in the catalog | Coupons live for as long as you let them. Audit periodically; archive when retired. |
