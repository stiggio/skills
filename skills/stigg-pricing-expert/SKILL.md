---
name: stigg-pricing-expert
description: Use when the user needs help PICKING a monetization model — not implementing one. Triggers include "what pricing should I use", "how should I monetize this", "which pricing model fits", "should I do per-seat or usage-based", "should I use credits", "design my pricing", "I'm pricing an AI product", "freemium vs trial", "hybrid pricing advice", "value metric". Skip for catalog *implementation* (use stigg-pricing-modeling) and runtime gating (use stigg-entitlements).
---

# Stigg Pricing Expert — Pick the Right Monetization Model

This skill helps the user choose the right monetization model **before** they start modeling it in Stigg. It's advisory — it asks diagnostic questions, maps answers to Stigg's supported pricing models, and then hands off to `stigg-pricing-modeling` for execution.

> **What this skill does NOT do:** model plans, create features, configure charges, or write code. It surfaces the right *shape* of the pricing model and points the user at the implementation skill.

## Before You Start

Per the umbrella `stigg` skill: **search first.** Stigg's supported pricing models evolve (recent additions include credit-based monetization with auto-recharge, custom credit consumption formulas, seat-based credit pools). Confirm what's currently supported before recommending.

## The Process

### 1 — Clarify what's being priced

Ask if not stated:

- **What does the product *do* for the customer?** (Outcome they pay for.)
- **Who is the buyer?** (End user, team admin, procurement?)
- **What's the granularity of value?** (Per request, per seat, per project, flat access?)
- **Is the product already in market, or pre-launch?** (Pricing-changes velocity differs.)

### 2 — Identify the value metric

The **value metric** is the thing that, when more of it happens, the customer gets more value. **Picking the right value metric is the highest-leverage decision in this whole process.** Discovery questions, common metric patterns by domain, and how to handle multiple competing metrics: `references/value-metric-discovery.md`.

Common value metrics by domain:

| Domain | Likely value metric |
|---|---|
| Collaboration tool | Active users / seats |
| Storage / cloud infra | GB stored, GB transferred, compute-hours |
| AI / LLM apps | Tokens, generations, model calls |
| Marketing / outreach | Emails sent, contacts, conversions |
| Analytics | Events ingested, dashboards, queries |
| API products | API calls, requests, throughput |
| Vertical SaaS | Domain-specific (transactions processed, employees managed, properties listed) |

Ask:

- **Where does usage scale with value to the customer?** That's a candidate metric.
- **What would the customer pay more / less for if it grew?** Validates the metric.
- **Is the metric measurable in real time?** (If not, switch to a proxy or aggregate-after-the-fact model.)

If multiple metrics surface, you can model a **hybrid** (see below). Don't try to monetize on every dimension — pick one or two primary, the rest as soft caps.

### 3 — Map onto Stigg's supported models

Once the value metric is clear, map onto a Stigg pricing model:

| If… | Use… | Implementation skill |
|---|---|---|
| Predictable monthly fee, one user paying | **Flat fee** | `stigg-pricing-modeling` (Plan + flat charge) |
| Pay per resource selected (seats, environments, projects) | **Per-unit** | `stigg-pricing-modeling` (Per-unit charge) |
| Usage scales unpredictably; bill what was used | **Pay-as-you-go** | `stigg-pricing-modeling` (Usage-based charge) |
| Customer commits to N units up front, true-up after | **In-advance commitment** | `stigg-pricing-modeling` |
| Floor on revenue regardless of usage | **Minimum spend** | `stigg-pricing-modeling` |
| Usage-based with included quota | **Quota + overages** | `stigg-pricing-modeling` (overages section) |
| Variable-cost workloads (multi-model AI, mixed inputs) | **Credits with custom formula** | `stigg-credits` (formulas) |
| Prepaid / pay-before-you-use (OpenAI-style) | **Credits with auto-recharge** | `stigg-credits` (auto-recharge) |
| Team product where credits should grow with seats | **Seat-based credit pools** | `stigg-credits` (seat pools) |
| Multiple revenue streams in one offering (e.g., flat base + usage) | **Hybrid** — combine flat / per-unit / usage charges on one plan | `stigg-pricing-modeling` (combining charges) |
| Free tier to drive top-of-funnel | **Freemium plan** | `stigg-pricing-modeling` (Free plan) |
| Time-bound preview of paid features | **Free trial** | `stigg-pricing-modeling` + `stigg-subscriptions` (trials) |
| Negotiated, sales-led pricing per customer | **Custom plan** | `stigg-pricing-modeling` (Custom plan) |
| Multi-region pricing | Layer **price localization** on top of any model | `stigg-pricing-modeling/references/price-localization.md` |

The decision tree with edge cases lives in `references/pricing-model-decision-tree.md`.

### 4 — Sanity-check, then hand off

Sketch revenue at median and top-decile usage; confirm the bill is predictable enough for the buyer's segment (self-serve customers can't reason about graduated overages on top of credits); confirm an expansion path exists (Free → Pro → Enterprise should feel inevitable); flag Finance pitfalls (non-zero cost basis on promotional grants, hand-rolled discounts outside Stigg coupons, per-customer pricing without custom plans). The "Done" checklist below names the seven things the user should know before you route to implementation.

### 5 — Hand off to implementation

Once a model is chosen, route the user explicitly:

- **Catalog modeling** (features, plans, addons, charges) → `stigg-pricing-modeling`.
- **Credits-based monetization** → `stigg-credits`.
- **Subscription provisioning / lifecycle** → `stigg-subscriptions`.
- **Drop-in UI** → `stigg-widgets`.
- **End-to-end recipes** (freemium / hybrid / credits) → `stigg-recipes`.

Do **not** start authoring catalog config in this skill — that's the implementation skill's job. The full hand-off checklist (what counts as a "done" recommendation, which target skill to route to per concern, how to phrase the hand-off message): `references/handoff-to-modeling.md`.

## Heuristics — When to Pick What

### Per-seat vs usage-based

- **Per-seat** when value scales linearly with users; admin buys for the team; predictable budgeting matters.
- **Usage-based** when value scales with consumption regardless of who's consuming; customers expect to pay only for what they use.
- **Both** (hybrid) when both axes matter — e.g., per-seat base + per-call usage on top.

### Credits vs raw usage charges

- **Credits** when:
  - Costs vary by workload (LLM models, file types, regions).
  - Customers benefit from flexibility (one balance for many features).
  - You want prepaid / commitment / promotional mechanics.
- **Raw usage** when:
  - Pricing is uniform per unit.
  - Customers think of the metric directly (API calls, GB).
  - Simplicity wins over flexibility.

### Freemium vs free trial

- **Freemium** — perpetual free tier with limits. Drives top-of-funnel; customers convert when they hit limits.
- **Free trial** — time-bound preview of paid features. Drives conversion through urgency and exposure to value.
- **Both** — freemium for low-end discovery; trial for upgrading freemium users to paid.

### Self-serve vs sales-led

- **Self-serve** — published plans, paywall, checkout. Best for SMB / individual users.
- **Sales-led** — custom plans, sales reps configuring pricing per deal. Best for enterprise.
- **Both** — single product, multiple GTM motions. Stigg supports this — public plans + a custom plan tier.

## What Counts as a "Done" Recommendation

Before handing off, the user should know:

1. The **value metric** they'll meter on.
2. The **plan structure** — free / paid / custom; how many tiers.
3. The **charge model** per plan (flat / per-unit / usage / credits).
4. Whether **add-ons** extend plans, and which plans they're compatible with.
5. Whether **credits** are involved and which currencies.
6. Whether **trials** apply and for how long.
7. Whether **price localization** applies and which markets.

If any of these are still ambiguous, ask one more question before handing off.

## When NOT to Use This Skill

- The user has already decided on a model and wants to *implement* it → go directly to `stigg-pricing-modeling` (or `stigg-credits` for credits).
- The user is asking about subscription lifecycle (provisioning / cancellation) → `stigg-subscriptions`.
- The user is asking about runtime entitlement enforcement → `stigg-entitlements`.
- The user wants a market-specific pricing study (competitive analysis, willingness-to-pay surveys) → that's outside Stigg; this skill is about mapping intent onto Stigg's supported models.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Recommending a model without surfacing the value metric | Every tier becomes arbitrary. Surface it first. |
| Mapping competitor pricing 1:1 | Anchor on the customer's value metric, not benchmarks. |
| Authoring catalog config in this skill | Hand off — modeling lives in `stigg-pricing-modeling`. |
| Recommending credits because "the user mentioned AI" | Credits fit when costs vary by workload. Uniform-cost AI works fine on raw usage charges. |
| Recommending hybrid by default | Hybrid adds complexity. Default to one primary model unless two metrics genuinely matter. |
| Skipping the median + top-decile sanity check | Pricing that breaks at scale gets discovered by your highest-value customers. Sketch the math. |
| Treating Stigg as the source of pricing strategy | Stigg implements pricing models; it doesn't pick the right one. That's this skill's job. |
