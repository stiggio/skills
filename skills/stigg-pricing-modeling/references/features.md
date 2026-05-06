# Features — Modeling Reference

Features are the building blocks. They define **what** can be controlled and monetized, before any plan or add-on assigns *how much* of them a customer gets (which is the entitlement).

## Feature types

| Type | Use for | Entitlement form |
|---|---|---|
| **Boolean** | "Has access to X" / "Doesn't" — premium support, SSO, custom branding | granted (true) or denied (false) |
| **Configuration — numeric** | A bound value: max projects, max seats, max file size | a numeric value (`5`, `100`, `unlimited`) |
| **Configuration — enum** | One or more from a predefined list: supported regions, available templates, support tier | a single or multi-value selection |
| **Metered** | Consumption tracked over time: API calls, messages sent, tokens used | a value + optional usage reset cadence |

## Metered features — meters and aggregation

A metered feature is paired with a **meter** that defines how raw events become a usage value.

| Aggregation | What it does | Example |
|---|---|---|
| `count` | counts events | "API calls / month" |
| `count_unique` | unique values for a dimension | "Active users / month" via `customerId` |
| `sum` | sums a numeric dimension | "Bandwidth used" via `bytes` |
| `min` / `max` | min/max of a dimension over the window | "Peak concurrent connections" |
| `average` | mean of a dimension | "Average response size" |

Aggregations other than `count` require you to specify the dimension to aggregate.

> Re-fetch the live page when modeling — supported aggregations are an evolving list.

## Usage reset cadence

For metered features:

- `hourly` / `daily` / `weekly` / `monthly` — usage resets at each cadence boundary
- `lifetime` — never resets; the value is cumulative for the lifetime of the subscription

Pick the cadence that matches how the customer sees their bill — typically monthly resets for monthly billing periods.

## Feature ID conventions

- Use stable, descriptive IDs (`api_calls`, `active_seats`, `feature.premium_support`).
- IDs are referenced from your application code on every entitlement check — treat them as part of your public API.
- Don't change a feature ID after launch unless you're ready to update every check site.

## Feature groups

Group features when several plans share the same set of entitlements. Editing the group propagates to all plans that use it. Useful for "Pro tier perks" that span the lineup.

## Operations on features (modeling-time)

- **Create** — boolean / configuration / metered.
- **Edit details** — name, description, metadata. The feature *type* is fixed at creation.
- **Archive** — hides from new plans; existing subs keep current entitlement values.
- **Unarchive** — restore.
- **Revoke access (per customer)** — covered in `stigg-entitlements` (it's a runtime/promotional concern, not a catalog edit).
- **View usage across plans/add-ons** — useful when refactoring the catalog.

## Common mistakes

| Mistake | Fix |
|---|---|
| Modeling "seats" as a metered feature | Use a **configuration** feature with a per-unit charge. Metered is for time-series consumption. |
| Feature IDs in user-facing copy | Keep IDs stable and machine-readable; show the **name** in UI. |
| Adding boolean features for tiered values | Use a configuration feature with the value, not five booleans. |
| Mismatching reset cadence and billing period | Daily resets on a monthly bill confuse customers — align them. |
| Designing aggregation around current product behavior, not future | Add as many event dimensions as you can up front; backfilling them later is painful. |
