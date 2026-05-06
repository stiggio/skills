# Products — Modeling Reference

A **product** is the foundational object that anchors plans, add-ons, and the customer journey for one product line. Most accounts have one product; multi-product accounts have one per offered product line.

## What a product controls

- **The plan and add-on lineup** for that product line.
- **Single vs multiple active subscriptions per customer** — products configure whether a customer can hold one subscription on this product, or several in parallel (e.g., one per workspace).
- **Default subscription behaviors** — how new subscriptions start, whether trials are available by default, downgrade/upgrade defaults.
- **Customer journey defaults** — onboarding plan, trial, payment-collection behavior at upgrade.

## When you need multiple products

- Distinct revenue lines that customers can buy independently (e.g. "Analytics" and "Messaging").
- Different teams own different products and want isolated catalogs.
- A customer can subscribe to *both* products in parallel; subscriptions are scoped per product.

For multi-tenancy *within* one product (workspaces / projects / orgs), you don't need multiple products — use **customer resources** instead, with the product configured for multiple active subscriptions.

## Operations

- **Create** a product.
- **Edit details** — name, description, customer-journey defaults.
- **Modeling pricing** — the page that walks through how to lay out plans and add-ons under the product.
- **Storing metadata** — arbitrary key/value, for integration.
- **Archive / unarchive.**
- **Delete** — only when no plans/subs reference it.
- **View list** of products in the catalog.

## Multiple subscriptions per product

If a customer can subscribe to several plans on the same product (e.g., one subscription per workspace), the product must have **multiple active subscriptions** enabled. Subscriptions are then keyed by `resourceId` — the workspace / project / app ID. This is the multi-tenancy lever; flagging it on after launch is harder than picking it up front.

## Common mistakes

| Mistake | Fix |
|---|---|
| Spinning up a new product for every plan tier | Plans, not products, model tiers. One product, many plans. |
| Using multiple products for multi-tenancy | Use **customer resources** within a single product instead. |
| Forgetting to enable multiple active subscriptions before workspaces ship | Set this at product creation; later changes are operationally heavy. |
| Treating product metadata as a config surface for application logic | Use it for integration metadata only; product-level business logic belongs in your app, not in the catalog. |
