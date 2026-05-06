# Recipe — Trial Add-Ons While Keeping a Paid Plan Active

**Goal:** customer is on a paid plan and you want to let them try one or more **add-ons** for free for a limited time, without disturbing the base subscription.

## When this is the right shape

- "Try Premium Support free for 14 days" — same plan, sold-on add-on.
- "Trial Advanced Analytics on top of Pro" — opt-in add-on, paid customers only.
- "Free 30-day trial of the AI add-on for existing customers" — boost adoption of a new module.

If you want a trial of a different *plan*, that's a different flow (use plan-level `trialOverrideConfiguration` on `provisionSubscription`).

## Steps

### 1. Model the add-on with a trial-friendly shape (`stigg-pricing-modeling/references/addons.md`)

In Stigg:

- The add-on is a **paid add-on** (recurring charge after the trial converts).
- It declares **compatibility** with the paid plans that should be eligible for the trial.

### 2. Provision the trial add-on (`stigg-subscriptions/references/trials.md`)

The exact API path varies by SDK / runtime — re-fetch the canonical guide at `guides/i-want-to/configure-trial-add-on.mdx` via `query_docs_filesystem_stigg`. The general shape:

- Provision a **trial subscription scoped to just the add-on**, alongside the customer's existing paid plan.
- The trial has its own duration; the base paid plan keeps running as-is.
- Per the **generous-entitlement rule**, the customer gets the most generous of (paid plan entitlements) and (paid plan + trial add-on entitlements) — typically the latter, so the add-on's entitlements apply during the trial.

### 3. Convert the trial to paid (or expire)

When the trial ends, the customer either:

- **Converts** — the add-on becomes a paid add-on on their subscription, billed per the recurring charge.
- **Expires without conversion** — the add-on entitlements drop; the base paid plan continues.

Listen for `trial.ends_soon` and `trial.expired` webhooks to drive in-app prompts.

### 4. Surface the trial state in the customer portal (`stigg-widgets/references/customer-portal.md`)

The portal's Subscription Overview shows active subscriptions including trial subs. Customers can see the trial end date and convert from there (via the Plan Picker / Pricing Table in the portal).

### 5. Gate the add-on's entitlements (`stigg-entitlements`)

The add-on may grant new metered features or boost existing ones. Update entitlement checks to read from the cached effective set — Stigg already merged plan + add-on + trial-add-on entitlements per the generous-entitlement rule, so your check code doesn't need new branches.

## Gotchas at the seams

- **The trial is a subscription scoped to the add-on, not a flag on the plan.** Don't confuse this with plan-level trials.
- **Generous-entitlement rule applies.** During the trial, the add-on's values win; after expiry, they drop. Your code reads the merged effective set.
- **Eligibility:** customers who already paid for the add-on aren't eligible for a self-serve trial of the same add-on. Admin overrides exist.
- **Self-serve vs admin-grant.** A first-time add-on trial can be self-serve (driven from the customer portal); ad-hoc trials for specific customers are typically admin-driven.
- **Compatibility matters.** The add-on's compatibility list determines which plans can host the trial. Cover all paid plans you want to enable.

## Verify in production

- Customer on Pro plan → starts trial of Premium Support add-on.
- Effective entitlements include the add-on's.
- 7 days before expiry → `trial.ends_soon` fires; in-app banner surfaces.
- Customer converts → add-on becomes paid on the subscription; billing reflects.
- Alternatively: trial expires → `trial.expired` fires; add-on entitlements drop; base plan continues unchanged.
