# Bulk Import Subscriptions — Reference

Imports active subscriptions from your legacy system into Stigg.

## Endpoint

- **REST:** `POST /api/v1/subscriptions/import`
- **GraphQL:** `importSubscriptionsBulk` mutation
- **SDK:** bulk-import method per language

> **Re-fetch the docs page** before authoring — input shape and response envelope evolve.

## Prerequisites — order matters

1. **Modeled your pricing** in Stigg (plans, addons, features, charges) — see `stigg-pricing-modeling`.
2. **Imported customers** first via `bulk-import-customers` — subscriptions reference `customerId`.
3. **Have a dedicated import API key** (contact Stigg support).

## The key trick: backdate the subscription start date

> **Set `startDate` in the past** so Stigg doesn't try to collect a payment immediately.

When `startDate` is in the past:

- The subscription is **active in Stigg from the import**.
- The **first billing charge happens in the next billing period**, not immediately.
- Customers are not double-charged for the current period.

If you don't backdate, Stigg attempts to collect payment via the configured payment method — which is wrong during a migration.

## Per-subscription fields

Typical input (verify exact field names per docs):

| Field | Notes |
|---|---|
| `customerId` | Must already exist in Stigg. |
| `planId` | Stigg plan reference ID. **Use new SKUs in Stigg, not legacy IDs.** |
| `billingPeriod` | `MONTHLY` / `ANNUAL` / etc. |
| `startDate` | **Backdate this** to prevent immediate charging. |
| `addons` | Active addons + quantities. |
| `billableFeatures` | Initial quantities for in-advance commitment / per-unit charges. |
| `priceOverrides` | Custom pricing (rare). |
| `appliedCoupon` | Pre-applied coupons. |
| `resourceId` | Required for multi-active products. |
| `metadata` | Arbitrary integration metadata. |

## Don't try to map legacy SKU IDs

The migration guide is explicit: **use new SKUs (defined in your Stigg catalog) and leave the legacy SKUs in the legacy system.** Trying to match legacy IDs introduces complex translation logic and edge-case bugs. Once a customer's Stigg subscription becomes active, the migration flow can cancel the legacy subscription.

## Async semantics

Like customer import, this is async:

- Submit batch → get job ID.
- Poll status.
- Inspect per-row failures.
- Retry only failed rows.

## What happens to the legacy subscription

Per the migration guide:

> "Once the new subscriptions become active in Stigg, the system will automatically cancel the legacy subscriptions."

Verify this on a small test cohort first — the exact behavior depends on your billing-provider integration setup.

## Importing recent usage

After subscriptions are imported, replay recent usage so customers' running totals are correct:

- Use `reportUsage` (calculated style) for authoritative current values, **or**
- Use `reportEvent` if you have a backlog of raw events to send.
- **Stigg overwrites existing usage values only if the imported timestamp is more recent.** This prevents duplication during retries.

## Common mistakes

| Mistake | Fix |
|---|---|
| Forgetting to backdate `startDate` | Customers get charged immediately. Always backdate. |
| Mapping legacy SKU IDs into Stigg | Use new Stigg SKUs. Leave legacy in the legacy system. |
| Importing subscriptions before customers | Foreign-key fail. Order matters: customers first. |
| Running without a dedicated import key | Production rate limits throttle the job. Get a dedicated key from support. |
| Skipping `resourceId` on multi-active products | Subscriptions land at the wrong scope. |
| Re-running on partial failure | Retry only the failed rows. The successful rows have already created subs. |
| Forgetting to import recent usage after subscriptions | Customer balances and entitlement counters are wrong post-import. Replay. |
| Decommissioning the legacy billing source before validating Stigg revenue lines up | Catch reconciliation issues before turnoff, not after. |
