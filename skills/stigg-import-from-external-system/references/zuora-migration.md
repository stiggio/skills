# Migrating from Zuora — Reference

Zuora has a richer object model than most SaaS billing systems (account, contact, subscription, rate plan, charge, etc.). Stigg ships specific tooling for the Zuora migration:

- **Zuora catalog importer** — imports Zuora products / rate plans into a Stigg catalog.
- **Zuora termed subscriptions** — Stigg understands Zuora's subscription term semantics.
- **Zuora payment form integration** — collect payment via Zuora's hosted form from inside Stigg flows.
- **Zuora bill runs** — Stigg coordinates with Zuora's bill run schedule.

Re-fetch the canonical guides under `documentation/native-integrations/billing/zuora/` via `query_docs_filesystem_stigg`.

## High-level approach

Same Strangler Fig pattern as the general migration playbook — dual-write, gradual rollout, shadow-read, switch and sunset. Zuora-specific notes:

### Catalog migration

Use the **Zuora catalog importer** to bring Zuora products and rate plans into Stigg as plans / addons. Validate the imported catalog before going further:

- Confirm rate plans map to the right Stigg plan / charge model.
- Spot-check a few customer subscriptions against the imported catalog.
- Re-run the importer if Zuora catalog changes during your migration window.

### Subscription migration — termed subs

Zuora's "termed" subscriptions (with explicit term start / end / renewal) require care:

- Stigg's subscription model has its own term semantics. Map Zuora term → Stigg `currentBillingPeriodEnd` / renewal config carefully.
- Verify renewal behavior matches in shadow-read before flipping.

### Payment form

If you use Zuora's hosted payment form, integrate it via Stigg's Zuora payment form integration so customers collect cards through Zuora's PCI surface but provision through Stigg's flows.

### Bill runs

Zuora bill runs are scheduled. Coordinate cutover with bill-run cycles:

- Don't cut over mid-bill-run.
- Verify Stigg-driven invoicing aligns with Zuora's bill run cadence post-cutover.

### APIs

Stigg uses Zuora's **Orders API** for managing subscriptions. Older accounts on the Subscribe API may need to migrate to Orders API first. The Stigg docs page on Zuora APIs lists what's used and any minimum-version requirements.

## What's different from a Stripe migration

| Concern | Stripe-only migration | Zuora migration |
|---|---|---|
| Object model complexity | Customers + subscriptions + invoices | Accounts + contacts + subscriptions + rate plans + bill runs + invoices |
| Catalog import | Manual (Stigg's plan model is straightforward) | Use the Zuora catalog importer |
| Payment surface | Stripe Elements / Checkout | Zuora hosted payment form |
| Term semantics | Implicit renewal | Explicit term + renewal config |
| Bill scheduling | Per-subscription invoice on billing date | Coordinated bill runs |

## When to escalate to Stigg support

- Custom Zuora setups (non-standard term lengths, custom Apex, complex rate plans).
- Catalog migration produces unexpected results.
- Customers on legacy Zuora APIs (pre-Orders API).
- Reconciliation failures during cutover.

Stigg has guided multiple Zuora migrations and has playbooks for the gnarly cases.

## Common Zuora-specific mistakes

| Mistake | Fix |
|---|---|
| Skipping the catalog importer and modeling plans by hand | The importer captures rate plan structure; manual modeling drifts. |
| Mid-bill-run cutover | Coordinate with the Zuora bill run schedule. Don't cut over mid-cycle. |
| Forgetting term semantics | Termed subs renew on their term boundary, not arbitrarily. Validate renewal behavior in shadow-read. |
| Trying to migrate Zuora payment methods directly | Use the payment form integration; don't try to re-tokenize cards into Stripe. |
| Not testing bill runs post-cutover | Bills are how revenue lands. Verify the first post-cutover bill run end-to-end. |
