# Stigg Core Concepts — Glossary

The minimum vocabulary needed to model anything in Stigg. Sourced from [docs.stigg.io / Core concepts](https://docs.stigg.io/documentation/getting-started/core-concepts) — re-fetch for the canonical version.

## Entity relationships (text view)

```
Product           ──has─→ Plan, Add-on
Plan / Add-on     ──grants─→ Entitlement
Entitlement       ──configures─→ Feature
Feature           ──measured-by─→ Meter
Customer          ──holds─→ Subscription      (and optionally many Customer-Resources)
Customer-Resource ──scopes─→ Subscription
Subscription      ──based-on─→ Plan, includes Add-ons, may have Coupons, may be Promotional
Customer          ──owns─→ Credit-Pool ──denominated-in─→ Credit-Currency
Credit-Pool       ──funded-by─→ Credit-Grant
Consumption-Mapping ──deducts-from─→ Credit-Currency
```

## Catalog side (what you model)

### Product

Top-level product or product line. Anchors plans, add-ons, and the customer journey. Controls how many active subscriptions a customer can hold (single vs multiple parallel) and default subscription behaviors.

### Plan

A tier on a product, packaged with pricing, features, and optional trial behavior. Three types:

- **Free** — freemium; no payment info required; usage-limited.
- **Paid** — fixed and/or usage-based pricing, billing periods, trials, optionally connected to a billing provider (Stripe, Zuora, etc.).
- **Custom** — sales-led; per-customer pricing and feature configuration, usually provisioned via API.

### Add-on

Modular extension of a paid or custom plan. Entitlements are **additive** (sum of plan + add-ons; multi-instance multiplies). Two flavors:

- **Paid add-ons** — self-serve, recurring charge, can appear in pricing tables.
- **Custom add-ons** — paired with a custom plan; per-customer pricing.

### Feature

A capability that can be controlled and monetized. Types:

- **Boolean** — on/off (e.g., "Premium support").
- **Configuration** — numeric or enum value (e.g., "Number of seats", "Available templates").
- **Metered** — usage tracked over time, with optional limits and an optional reset cadence (hourly / daily / weekly / monthly).

### Meter

How raw usage data aggregates for a metered feature: aggregation function (sum / count / average / etc.), data field, optional filters. Connects raw events to the feature's effective usage value.

### Entitlement

The combination of a feature and its configuration. Lives at two levels:

- **Package entitlement** — defined on a plan or add-on; every subscriber gets it.
- **Promotional entitlement (override)** — granted directly to a specific customer, independent of their plan. Time-limited or lifetime.

> **Mental model:** features are vocabulary; entitlements are sentences.

### Price

Billing config for a plan or add-on. Models:

- **Flat fee** — fixed recurring (e.g., $49/month).
- **Per-unit** — multiplied by units selected (e.g., $10/seat/month).
- **Usage-based** — calculated from actual consumption.

Tiered pricing (volume / graduated / per-unit volume), minimum spend, and credit-based billing are supported.

### Coupon

Discount instrument applied to a customer or subscription. Percentage or fixed amount, time-bounded or open-ended.

## Customer side (how customers interact)

### Customer

An individual or organization holding subscriptions. Unique immutable ID, optional name/email.

### Customer Resource

A distinct unit inside a customer (workspace, project, organization). Each resource can hold its own subscriptions and entitlements independently — used for multi-tenancy.

### Subscription

The customer's link to a plan + add-ons. Determines effective entitlements, usage limits, billing behavior. Supports trials, discounts, usage-based charges, scheduled changes, upgrades, downgrades.

### Promotional Entitlement

Per-customer entitlement override outside any subscription. Use cases: temporary limit increases, feature trials, enterprise accommodations, goodwill gestures.

### Usage Measurement

Tracks consumption of a metered feature against the customer's entitlement limit. Two ingestion modes:

- **Calculated usage** — your app reports the current value (e.g., "this customer currently has 5 seats").
- **Raw usage events** — your app sends individual events; Stigg aggregates them.

> The distinction between `usage.report` (calculated) and `events.report` (raw) is by **throughput / scale**, not by who counts. Covered in depth in `stigg-entitlements`.

## Credits

For prepaid and credit-based pricing.

- **Credit currency** — the unit (e.g., "AI tokens", "API credits"). Non-interchangeable across currencies.
- **Credit pool** — a customer's live balance of one credit currency.
- **Credit grant** — a block of credits added to a pool (recurring plan credits, prepaid pack, promotional, manual). Carries amount, effective/expiration dates, category, cost basis, consumption priority.
- **Consumption mapping** — converts metered feature usage into credit deductions (e.g., "1 email = 1 AI Token").
- **Credit ledger** — append-only chronological record of every change to a customer's credit balance.

## Operational

### Environment

Isolated workspace inside an account (e.g., development, staging, production). Each has its own configurations, data, and API keys. Production is for real customer interactions and is subject to limits/SLAs; non-production is sandbox.

### API key

Environment-bound credential. Two system-managed types per environment:

- **Full access (server)** — `server-` prefix; backend only.
- **Publishable (client)** — `client-` prefix; frontend only; read-only and immutable scope.

Plus user-created **scoped server keys** (Scale plan only) with explicit, restricted permissions. Covered in depth in `stigg-api`.

## How effective entitlements are computed

A customer's effective entitlements come from up to **five sources**, combined under the **generous-entitlement rule** — Stigg picks the most generous value across all of them:

1. **Plan entitlements** — from the active subscription's plan.
2. **Add-on entitlements** — from any add-ons on the subscription (additive).
3. **Trial subscription entitlements** — during overlapping trials.
4. **Promotional entitlements** — direct, per-customer overrides.
5. **Additional product subscriptions** — when the customer holds subscriptions on other products that grant the same feature.

Stigg computes and caches these into a single effective set per customer. SDKs and APIs read from the cached set.
