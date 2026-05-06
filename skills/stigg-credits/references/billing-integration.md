# Billing Integration — Reference

Stigg sits **alongside** your billing stack — no rip-and-replace. Your provider (Stripe / Zuora / custom) keeps owning payments, taxes, invoices, and payment methods. Stigg owns credit pools, grants, consumption, and enforcement.

## System boundaries

| Domain | Owner |
|---|---|
| Product catalog and prices | Your billing system |
| Taxes, invoicing | Your billing system |
| Payments, refunds, credit memos | Your billing system |
| Collections | Your billing system |
| **Credit currencies and pools** | **Stigg** |
| **Grants / top-ups** | **Stigg** |
| **Real-time consumption and enforcement** | **Stigg** |
| **Ledger, exports** | **Stigg** |
| **Customer-facing credit widgets** | **Stigg** |

## How money becomes a grant

| Event | What Stigg does |
|---|---|
| **Customer pays** (card or invoice) | Creates a **paid grant** with `amount`, `cost_basis`, `priority`, `effective_at`, `expires_at`. Updates the pool. Writes a ledger entry. |
| **Promotional credits issued** | Creates a grant with **zero `cost_basis`** (no revenue impact). Tracked separately in the ledger. |
| **Refund** | You revoke the corresponding grant (or its remainder) in Stigg to keep deferred / recognized aligned. The ledger records the revocation. |
| **Real-time burn** | As usage events arrive, Stigg prices via the feature → credits mapping (or custom formula) and burns immediately. |

## Three common flows

### 1. Self-serve top-ups

Customer buys more credits in-app. After a successful charge in your billing provider:

- Stigg issues a **paid grant** with `cost_basis` set to the per-credit price.
- Pool balance updates.
- Customer sees the new balance in the Credits Balance widget.

You can define **min / max purchase rules** and **default pricing** in the plan's credit configuration.

### 2. Sales-led purchases (invoice)

Common for B2B / enterprise:

- Issue an invoice via your billing provider.
- **Per your policy:** grant credits when the invoice is **issued** (optimistic) or **paid** (conservative). Most teams pick paid.
- `cost_basis` captured on the grant for revenue rec.

### 3. Automatic top-ups

See `auto-recharge.md`. Configure a low-balance trigger, top-up amount, and spend cap. When threshold is crossed, Stigg initiates a purchase via your chosen flow and creates a new grant on success.

## Overdraft is excluded from billing

Overdraft grants represent a **deficit**, not a purchase:

- `payment_collection: NOT_REQUIRED`
- `payment_collection_method: NONE`
- **Stripe:** no invoice is created.
- **Zuora:** no order or invoice is created.

Billing happens when the customer purchases new credits that **settle** the overdraft. See `consumption.md`.

## Reconciliation

Use the **ledger** and **grants** exports (Integrations → Data warehouse) to tie purchases, consumption, expirations, and revocations back to invoices and journals in Finance.

See `revenue-recognition.md` for the journal-entry model.

## Go-live checklist

1. Connect your billing provider in **Integrations**.
2. Create a credit currency (the Stigg app's catalog UI calls it a "credit type") and a prepaid plan with pricing, min / max rules.
3. Wire checkout / top-up (self-serve) **or** invoice (sales-led) purchase flows.
4. Map features → credits and send event-based usage.
5. **Test end-to-end:** purchase → grant → burn → auto top-up → refund / revoke → export.

## Common mistakes

| Mistake | Fix |
|---|---|
| Refunding a payment but not revoking the grant | Stigg's deferred revenue stays inflated. Always pair refund with revocation. |
| Issuing an invoice and granting credits before payment is collected | If the invoice goes uncollected, you've issued credits with no money. Pick "grant on paid" unless you have a strong reason otherwise. |
| Trying to migrate billing into Stigg | Stigg doesn't replace your billing system. Keep payments / invoicing where they are. |
| Charging through Stripe directly without telling Stigg | The grant won't appear in the pool. Either use Stigg's auto-recharge / top-up API, or your post-payment hook must create the grant in Stigg. |
| Mismatched `cost_basis` between billing system price and Stigg grant | Revenue rec breaks. Source `cost_basis` from the billing system's actual collected amount, not a hard-coded constant. |
