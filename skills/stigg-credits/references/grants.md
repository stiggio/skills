# Credit Grants — Reference

Grants are the individual blocks of credits added to a customer's pool. Five types, three lifecycle moments (create / update / void), one append-only ledger.

## Grant types

| Type | Origin | Cost basis | Notes |
|---|---|---|---|
| **Recurring** | Plan or add-on entitlement | Paid (the plan's price) | Issued automatically each billing cycle. Modeled on the plan/add-on, not granted ad-hoc. |
| **Prepaid** | Customer purchase up front | Paid | Fixed pack ("$10 buys 10,000 credits"). |
| **Promotional** | Manual grant for incentive | **Zero** | Time-bound or lifetime. Never affects revenue rec. |
| **Manual adjustment** | Admin add / remove | Paid or zero | Ad-hoc; keep audit trail in the `reason` field. |
| **Overdraft** | System-generated | n/a | Tracks negative balance. Cannot be created, edited, or voided manually. |

## Grant fields

| Field | Notes |
|---|---|
| `amount` | Credits in the block. |
| `creation_date` | When the grant record was created. |
| `effective_at` | When credits become usable (defaults to creation; can schedule). |
| `expires_at` | When unused credits vanish. Nullable for non-expiring. |
| `category` | `paid` / `promotional`. |
| `cost_basis` | $ per credit, **expressed in full dollars** (Stigg convention — `0.001` means $0.001/credit, not 0.1¢). Used for revenue rec. **Zero for promotional.** |
| `priority` | Lower number = higher priority. Burn order. |
| `status` | `Active` / `Expired` / `Voided` / `Scheduled`. |
| `reason` | Free-text — "Onboarding", "Promo ended", "Renewal bonus", "Payment failed", etc. |
| `actor` | Who created / changed it (System / Admin / API). |

## Granting credits via the Stigg app

**Customers → select a customer → Credits tab → Adjust credits balance:**

1. **Adjustment type** — `Grant credits`.
2. **Resource** — pick the credit currency.
3. **Credit amount** — number of credits.
4. **Schedule** (optional) — `Effective date` and / or `Expiry date`.
5. **Grant method:**
   - **Purchase credits** — sets `Per unit cost basis`, picks a `Payment method`, optional `Reason`.
   - **Promo / Free granted** — optional `Reason`.
6. Review the **Summary** (credits, total, previous / new balance).
7. **Grant Credits** to apply.

> Credits can be granted regardless of whether the customer has an active subscription. The pool persists; the balance updates immediately.

## Granting via API / SDK / MCP

The same operation is exposed via the GraphQL `createCreditGrant` mutation, REST `POST /credit-grants`, the SDK equivalents, and the Stigg MCP. Pass the same fields. **Re-fetch the docs page for your runtime** — the canonical field names live there.

## Modifying / voiding grants

- **Recurring grants** — managed by the plan / subscription. Don't void manually unless you know what you're doing; the next billing cycle will re-issue.
- **Prepaid grants** — can be voided to refund-and-revoke (coordinate with your billing system; Stigg doesn't auto-refund the payment).
- **Promotional grants** — can be revoked at any time.
- **Manual adjustment grants** — editable.
- **Overdraft grants** — read-only. Auto-settled when new credits arrive.

## Granting concurrently with auto-recharge

Auto-recharge issues paid grants automatically when the balance drops below the configured threshold. **It's still a grant** — visible in the table, on the ledger, with `cost_basis` set. Treat it the same as any other paid grant for revenue rec.

## Common mistakes

| Mistake | Fix |
|---|---|
| Granting promotional credits with `cost_basis > 0` | Pollutes revenue rec. Promotional = zero. |
| Trying to grant negative credits to "charge back" | Wrong direction. Use a `revoke grant` op or void a specific grant. |
| Picking arbitrary priority numbers | Establish a convention (e.g. promo=1, recurring=10, prepaid=20) and document it. |
| Issuing a recurring grant outside the plan flow | Recurring grants belong on the plan/add-on as an entitlement. Manual recurring grants drift from the catalog. |
| Granting before a payment is collected (paid case) | Guarantee payment first; otherwise you've issued credits with no money. |
| Granting credits to a customer with an overdraft and not noticing the auto-settlement | Stigg silently transfers the deficit. Surface this in your app's customer-credit UI. |
| Attempting to void an overdraft grant | Can't. Overdraft only auto-settles. |
