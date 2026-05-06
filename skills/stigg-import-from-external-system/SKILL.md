---
name: stigg-import-from-external-system
description: Use ONLY for one-off migrations bringing customers and subscriptions into Stigg from an external billing or entitlement system (Stripe alone, Recurly, Chargebee, Zuora, custom in-house). Triggers include "migrate from Stripe to Stigg", "migrate from Recurly to Stigg", "import customers from Zuora", "Stigg bulk import", "Strangler Fig migration", "dual-write to Stigg", "shadow-read", "cutover playbook". Skip for routine plan-version migrations within Stigg (use stigg-subscriptions/references/plan-version-migration.md).
---

# Stigg External-System Migration — One-Off Playbook

This skill is for the **one-time migration** from an external system (Stripe-only billing, Recurly, Chargebee, Zuora, a home-grown entitlements service, etc.) into Stigg. **It is not for routine ops.** Day-to-day work belongs to `stigg-subscriptions`, `stigg-entitlements`, `stigg-pricing-modeling`, and `stigg-credits`.

## Before You Start

Per the umbrella `stigg` skill: **search first.** The bulk-import endpoints (`POST /api/v1/customers/import`, `POST /api/v1/subscriptions/import`, and the `importSubscriptionsBulk` GraphQL mutation) evolve. Re-fetch the doc page before authoring import scripts. The full canonical migration guide is `/guides/i-want-to/migrate-from-legacy-systems.mdx` — open it via `query_docs_filesystem_stigg`.

## The Strangler Fig Pattern

Stigg recommends migrating using the [Strangler Fig pattern](https://martinfowler.com/bliki/StranglerFigApplication.html): wrap the legacy system with new Stigg capabilities, then incrementally shift functionality until the legacy is no longer needed. **Do not attempt a "big bang" cutover.**

Four stages:

1. **Implement dual-writes and backfill** — write new data to both systems; backfill historical.
2. **Roll out gradually** — feature-flag specific customers / cohorts onto Stigg.
3. **Monitor with shadow-reads** — verify Stigg behaves like the legacy before expanding rollout.
4. **Switch and sunset** — fully cut over and decommission the legacy system.

Each stage has its own reference file in this skill — see `references/`.

## Stage 1 — Dual-Writes and Backfill

### Dual-writes

Whenever a record is created or updated in the legacy system, **also call Stigg**. The APIs you'll dual-write through:

- `provisionCustomer` / `updateCustomer` — for customer events.
- `provisionSubscription` / `updateSubscription` / `cancelSubscription` — for subscription events.
- `reportUsage` (or `reportEvent`) — for usage events.

> Method names below are the `@stigg/node-server-sdk` (Node SDK) shapes. Migrators on Python / Go / Java / Ruby / .NET — see your SDK's reference at [docs.stigg.io](https://docs.stigg.io) for the equivalent calls; the patterns and Strangler Fig stages are the same.

Sketch:

```ts
export async function registerCustomer(customer) {
  await createCustomerLegacy(customer);
  await ensureCustomerInStigg(customer);
}

async function ensureCustomerInStigg(customer) {
  await stigg.provisionCustomer({ customerId: customer.id, /* ... */ });
  customer.enrolled = true;       // track which customers were already provisioned
}

export async function subscribeToPlan(customer, plan) {
  await createSubscriptionLegacy(customer, plan);
  if (!customer.enrolled) await ensureCustomerInStigg(customer);
  await stigg.provisionSubscription({ customerId: customer.id, planId: plan.id, /* ... */ });
}
```

### Backfill historical data

A background job that:

- **Imports customers** into Stigg (with payment-method identifiers from your billing provider where needed).
- **Skips already-enrolled** customers (`customer.enrolled === true` check).
- **Imports active subscriptions** — for each active sub in the legacy system, create a corresponding Stigg subscription.
  - Use **newly created SKUs** (your new Stigg catalog), not legacy SKU IDs.
  - **Backdate the subscription start date** so no payment is collected during the import.
  - Leave the legacy SKUs in the legacy system; don't try to map IDs across.
  - Once the new Stigg subscription becomes active, the system can automatically cancel the legacy subscription (per the migration guide).
- **Imports recent usage** — report the most recent usage records per customer. Stigg overwrites existing usage values **only if the imported timestamp is more recent**, preventing duplication.

> **Use a dedicated import API key.** Production rate limits on a single key can throttle large imports. **Contact Stigg support** to issue a separate key scoped to the import.

> **Test the import script in a sandbox / dev environment first.** The bulk-import APIs are async and accept large payloads — a botched run is hard to undo cleanly.

Bulk endpoints:

- **REST:** `POST /api/v1/customers/import` and `POST /api/v1/subscriptions/import`.
- **GraphQL:** `importSubscriptionsBulk` mutation.
- **SDK:** the bulk-import method exposed by your SDK.

Both run **asynchronously** — submit a batch, poll for completion, inspect failures, retry.

Detailed call shapes + params: `references/bulk-import-customers.md`, `references/bulk-import-subscriptions.md`.

## Stage 2 — Gradual Rollout

Introduce a customer-level feature flag (e.g. `use-stigg`) that determines which system manages each customer. Roll out by cohort:

- **Internal users** first.
- **Early adopters** / volunteers.
- **New signups** (clean slate, lowest risk).
- **Selected segments** (geo, plan, usage profile).
- **General availability** last.

### Implement a routing abstraction

Don't scatter `if (useStigg) { ... } else { ... }` across your codebase. Wrap behind a **billing-service interface**:

```ts
interface BillingService {
  provisionSubscription(...): Promise<...>;
  getEntitlement(...): Promise<...>;
  reportUsage(...): Promise<...>;
}

class StiggBilling implements BillingService { /* uses Stigg SDK */ }
class LegacyBilling implements BillingService { /* uses legacy APIs */ }

function getBilling(customer): BillingService {
  return featureFlag('use-stigg', customer) ? new StiggBilling() : new LegacyBilling();
}
```

Webflow's documented case used the Strategy Pattern here for zero-downtime switching.

### Verify enrollment before flipping

Before turning on `use-stigg` for a customer, **check that the customer + subscription already exist in Stigg** (created via dual-write or backfill). If not, trigger backfill before flipping. Otherwise the customer hits Stigg and finds nothing.

## Stage 3 — Shadow-Reads

For customers routed to Stigg, **silently call the legacy system in the background** and compare results. Surface mismatches.

### Compare in the abstraction layer

Have the routing wrapper call both, return the Stigg result to the caller, **but log a comparison**:

- Log `customerId`, `featureId`, both responses, and the delta.
- If discrepancies could affect the customer experience (missing features, misaligned limits), trigger an alert.
- **Until mismatches are resolved**, fall back to legacy responses in production. Correctness wins over migration speed.

### Track migration metrics

Build a dashboard:

- How many customers are running through Stigg?
- How many are still in shadow-read-only mode?
- How often are mismatches detected?
- What's the mismatch *rate* per cohort?

This dashboard answers "are we ready to flip more customers?" objectively.

## Stage 4 — Switch and Sunset

Once Stigg has reached parity through shadow-reads and the rollout has been stable:

1. **Flip `use-stigg` globally.** All subscription / entitlement flows now go through Stigg.
2. **Cancel legacy subscriptions gracefully.** For customers whose subscriptions are managed by Stigg, schedule cancellations in the legacy system to avoid double-charging. If Stigg created backdated subs through its native billing connector, the legacy sub can typically be cancelled once the Stigg sub confirms active.
3. **Keep monitoring.** Discrepancies / regressions can still surface post-cutover. Log errors and mismatches; retain validation logic temporarily.
4. **Sunset in phases:**
   - Turn off **writes** to the legacy system.
   - Eventually turn off **reads**.
   - **Clean up infrastructure / code** only after you're confident nothing is calling it anymore.
5. **Tell the team.** Share outcomes — performance wins, operational improvements, rollout stats. Builds confidence in the new system.

## Special cases

### Migrating from Stripe-only billing

You're moving entitlement / subscription state into Stigg while keeping Stripe as the billing provider. The Stripe integration handles this — you don't need to migrate Stripe customers; you migrate the subscription / entitlement state. See `references/cutover-playbook.md`.

### Migrating from Zuora

Zuora has a richer object model (account, contact, subscription, rate plan, etc.). Stigg has a Zuora catalog importer for the catalog side. For subscriptions, use bulk import after mapping rate plans to Stigg plans. See `references/zuora-migration.md` (sketch — re-fetch the official guide page).

### Migrating from a custom in-house system

Same pattern — you're the billing provider too in this case. The rollout is harder because you may not have well-defined APIs to wrap. Spend more time in Stage 1 (dual-write) before flipping anyone.

## When NOT to Use This Skill

- Routine subscription operations (provision, update, cancel, plan-version migration after a publish) → `stigg-subscriptions`.
- Modeling pricing in Stigg → `stigg-pricing-modeling`.
- Setting up the Stigg MCP → `stigg-mcp`.
- Picking the right pricing model for a new product → `stigg-pricing-expert`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Big-bang cutover | Don't. Use the four-stage Strangler Fig pattern. |
| Skipping shadow-reads to "save time" | The discrepancies you'd find during shadow-read become production incidents post-cutover. Don't skip. |
| Mapping legacy SKU IDs into Stigg | Use new SKUs in Stigg's catalog. Mapping legacy IDs introduces complex translation logic and bugs. |
| Importing without backdating subscription start | Stigg attempts to collect a payment immediately. Always backdate. |
| Hitting prod rate limits during a large backfill | Get a dedicated import API key from Stigg support. |
| Routing all customers through `use-stigg` at once | Cohort-by-cohort. Roll back is much harder once 100% are on Stigg. |
| Re-running the import script twice without idempotency | Stigg dedupes by `customerId`, but bulk imports create new subs per call. Use idempotency keys / enrollment flags to prevent re-imports. |
| Decommissioning the legacy system before validation logic has been stable for weeks | Premature shutdown means lost ground if something surfaces. Wait. |
| Forgetting to coordinate billing-system cancellations after Stigg cutover | Customers get double-billed. Schedule legacy cancellations as part of the cutover plan. |
