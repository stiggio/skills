# Stigg Webhook Event Catalog

Curated map of webhook events by domain. **Names evolve** — always re-confirm against the live catalog at `/documentation/native-integrations/webhooks/events.mdx` (and the per-event subpages for payload schemas) via the Mintlify Stigg docs MCP.

## Customer lifecycle

| Event | When it fires |
|---|---|
| `customer.created` | New customer provisioned |
| `customer.updated` | Customer fields changed (name, email, billingId, metadata) |
| `customer.deleted` | Customer archived |
| `customer.payment_failed` | Billing provider reports a payment failure |
| `payment_method.attached` | Customer added a payment method |
| `payment_method.detached` | Payment method removed |

## Subscription lifecycle

| Event | When it fires |
|---|---|
| `subscription.created` | New subscription provisioned |
| `subscription.updated` | Subscription changed (plan, addons, period, etc.) |
| `subscription.canceled` | Subscription canceled (immediate or scheduled time reached) |
| `subscription.creation_failed` | Provisioning failed (often payment-method or billing-provider issue) |
| `subscription.expired_non_recurring` | Non-recurring subscription reached end |
| `billing.month_ends_soon` | 1 hour before billing month ends — useful for last-minute usage prompts |
| `subscription.spend_limit_exceeded` | `maximumSpend` ceiling crossed |

## Trial lifecycle

| Event | When it fires | Use for |
|---|---|---|
| `trial.started` | Trial subscription provisioned | Analytics tagging, welcome email |
| `trial.ends_soon` | 7 days before expiry | Win-back campaign trigger (most effective when sent before expiry) |
| `trial.expired` | Trial ended without conversion | Win-back UX, churn analysis |
| `trial.converted` | Customer converted trial → paid | Post-conversion onboarding |

## Entitlements

| Event | When it fires |
|---|---|
| `entitlement.usage_exceeded` | Customer crossed an entitlement quota — useful for in-app warnings, sales triggers |
| `entitlements.updated` | Effective entitlements recomputed (after sub change, promotional grant, plan publish, etc.) |

## Credits — balance

| Event | When it fires |
|---|---|
| `credit.balance.low` | Balance drops below configured low-threshold |
| `credit.balance.depleted` | Balance reaches zero |

## Credits — grants

| Event | When it fires |
|---|---|
| `credit.grant.created` | New grant added to a pool |
| `credit.grant.updated` | Grant details changed |
| `credit.grant.depleted` | Grant fully consumed |
| `credit.grant.expired` | Grant reached `expires_at` with remaining balance |
| `credit.grant.balance_low` | Specific grant approaching depletion (vs. pool-level low) |

## Credits — auto-recharge

| Event | When it fires |
|---|---|
| `automatic_recharge.operation.attempted` | Every auto-recharge attempt (success and failure) — log for ops |
| `credits.automatic_recharge_limit_exceeded` | Customer crossed 80%, 90%, or 100% of monthly auto-recharge spend cap |
| `automatic_recharge.configuration.changed` | Enabled / disabled (manual or system, e.g. on payment failure) — surface in UI immediately |

## Promotional entitlements

| Event | When it fires |
|---|---|
| `promotional_entitlement.granted` | Promo grant created on a customer |
| `promotional_entitlement.updated` | Promo grant changed |
| `promotional_entitlement.revoked` | Promo manually revoked |
| `promotional_entitlement.expired` | Time-bounded promo reached end date |
| `promotional_entitlement.ends_soon` | Promo approaching expiry |

## Catalog (publish notifications)

| Event | When it fires |
|---|---|
| `plan.created` | New plan published for the first time |
| `plan.updated_new_version_published` | Plan republished — new version live |
| `plan.deleted` | Plan archived |
| `addon.created` / `addon.updated_new_version_published` / `addon.deleted` | Add-on lifecycle |
| `feature.created` / `feature.updated` / `feature.deleted` | Feature lifecycle |
| `coupon.created` / `coupon.updated` / `coupon.archived` | Coupon lifecycle |

## Third-party

| Event | When it fires |
|---|---|
| `third_party_sync_failed` | Sync to a connected billing provider (Stripe, Zuora, etc.) failed — investigate the integration |

## Production oncall watch list

A pragmatic subset to alert on:

- **`automatic_recharge.configuration.changed`** with `disabled: true` → customer's auto-recharge is off; their balance will deplete silently.
- **`customer.payment_failed`** → revenue at risk.
- **`subscription.creation_failed`** → broken signup flow.
- **`third_party_sync_failed`** → billing-integration health.
- **`entitlement.usage_exceeded`** spike → either a customer abusing a feature or a misconfigured quota.
- **Webhook-delivery error rate** (your own metric, not a Stigg event) → your endpoint is failing.
