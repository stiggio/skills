# Seat-Based Credit Pools — Reference

Link a recurring credit grant to a metered seat feature so the customer's credit balance scales with team size.

## The pattern

> "1,000 AI credits per seat per month."
>
> A team of 5 seats gets 5,000 credits / month. When they add a 6th seat, Stigg automatically provisions another 1,000 credits to the **shared pool**.

Credits are **not siloed per user** — the entire org draws from one unified balance. This maximizes utilization across the team.

## Requirements

- **Stripe integration.** Seat-based pools currently require Stigg's Stripe integration.
- The plan or add-on must include **both**:
  - A **credit entitlement** (recurring grant).
  - A **metered seat feature** (or another unit-of-growth feature).

## Configuration

In the Stigg app:

1. **Product Catalog → Plans** (or **Add-ons**).
2. Open or create the plan / add-on.
3. **Entitlements** section → add your credit feature as an entitlement.
4. Click the **Link icon** next to the credit grant to associate it with another entitlement.
5. Select the **metered feature representing seats** (or another unit of growth).
6. Set the **credits per unit** — e.g. `1,000 credits per seat`.
7. Set the **grant cadence** — e.g. `monthly`.
8. **Add**, then **Publish** the plan / add-on.

Once a customer subscribes, their shared pool updates automatically as seats are added or removed.

## What changes when seats change

| Change | Pool effect |
|---|---|
| Seat added mid-period | Shared pool grants the per-seat amount immediately. |
| Seat removed mid-period | Pool does **not** decrease retroactively — credits already granted stay; no further per-seat grants for that seat. |
| Seats stay constant across billing periods | Standard recurring grant cadence. |

(Verify exact pro-ration / removal behavior in the docs for your scenario — Stigg's behavior here may evolve.)

## Why a shared pool, not per-seat pools

- **Utilization.** A heavy user on a 10-seat team can use the team's whole balance without waste; a per-seat model leaves stranded credits.
- **Simpler ops.** One ledger per `(customer, currency)` to reason about.
- **Scales naturally with team growth** — same pool, just bigger.

## Combining with auto-recharge

Auto-recharge can be enabled on top of a seat-based pool — when the shared balance drops below a threshold, Stigg auto-purchases more credits using the customer's configured payment method, regardless of seat count.

## Common mistakes

| Mistake | Fix |
|---|---|
| Trying to enforce per-seat consumption caps | Seat-based pools are **shared by design**. If you need per-seat caps, model individual user-level consumption with a different feature, not credits. |
| Removing seats and expecting an immediate refund | Stigg doesn't claw back already-granted credits when a seat is removed. Check the docs for the canonical behavior. |
| Setting the per-seat amount too high relative to expected usage | Customers stockpile credits; revenue recognition skews toward deferred. Pick numbers based on expected median consumption per seat. |
| Treating seat-based as a substitute for usage-based pricing | It's predictable per-seat; it doesn't scale with usage *intensity* per seat. For variable workloads, layer on a custom formula or pay-as-you-go overage. |
| Forgetting Stripe is required | Seat-based pools won't appear as an option without the Stripe integration. |
