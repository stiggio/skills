# Credit Consumption Logic — Reference

How Stigg picks which credits to burn, what happens when a single event spans multiple grants, and how overdraft works.

## Burn order — the prioritization rules

When multiple grants can apply to a usage event or invoice, Stigg picks them in this order:

1. **Priority** — lower numeric value = higher priority.
2. **Expiration date (`expires_at`)** — sooner-expiring before later or non-expiring.
3. **Category** — among same-expiration, **promotional before paid**.
4. **Effective date (`effective_at`)** — earlier first.
5. **Created date (`created_at`)** — older first if all else ties.

## Multi-grant deductions

If a single usage event needs more credits than the highest-ranked grant has:

- Stigg deducts from the top-ranked grant until its balance is zero.
- Continues with the next eligible grant by the same prioritization.
- **One ledger entry per grant consumed.**

### Worked example

A customer has these grants:

| Grant | Priority | Expires | Category | Balance |
|---|---|---|---|---|
| A | 1 | Sep 1 | paid | 50 |
| B | 1 | Sep 1 | promotional | 20 |
| C | 2 | Aug 15 | promotional | 100 |

A usage event needs **60 credits**.

Apply the rules:

1. **Priority** → only A and B are priority 1; C is priority 2 and skipped (despite expiring sooner — priority wins).
2. **Expiration** → A and B both expire Sep 1, tie.
3. **Category** → promotional before paid → B first, then A.

**Deduction:**

- 20 credits from B (B → 0)
- 40 credits from A (A → 10)
- C is untouched

Two ledger entries created (one per consumed grant).

## Overdraft — when credits are depleted

When a usage event needs more credits than are available across all active grants, Stigg does **not** reject the event. It creates an **overdraft grant** to track the deficit.

### Worked example

- Customer has one active grant with 10 credits remaining.
- A usage event costs 25 credits.
- Stigg deducts 10 from the existing grant (fully consumed).
- The remaining 15 are tracked in a new overdraft grant.
- Effective balance: **−15 credits.**
- Overdraft grant appears in the credit grants table and ledger.

### Overdraft grant properties

- `display_name`: "Overdraft"
- `grant_type`: `OVERDRAFT`
- `amount`: 0 (the overdraft has no credits to give — it tracks what's owed)
- `consumed_amount`: the deficit
- `payment_collection`: `NOT_REQUIRED`
- `status`: `Active` until settled, then `Voided`

### Settlement when new credits arrive

Stigg auto-settles overdraft when new credits are granted:

- **Full settlement:** the new grant has enough capacity to cover the entire overdraft. Consumed amount is transferred to the new grant; overdraft is voided.
- **Partial settlement:** the new grant doesn't cover the whole overdraft. Available capacity transfers; overdraft stays active with a reduced deficit.

### Worked example (settlement)

- Customer has an overdraft of 15 credits.
- Customer purchases a 50-credit grant.
- 15 credits of consumption transfer from overdraft → new grant.
- Overdraft voided.
- New grant: `total: 50, consumed: 15, remaining: 35`.

### Constraints

- **One overdraft per `(customer, currency)`** (and per resource, if resource-scoped).
- Overdraft grants are **excluded from billing integration** — no Stripe invoice / Zuora order is created for them. Billing happens when the customer purchases new credits to settle.

## Common mistakes

| Mistake | Fix |
|---|---|
| Expecting Stigg to reject usage on zero balance | It won't. Stigg always tracks deficit via overdraft. Plan around this. |
| Picking priority by gut feel | Document a convention. Stigg's defaults are reasonable but you'll want explicit policy when you start mixing recurring + prepaid + promo. |
| Burning paid before promo because "promo is free" | Wrong direction. Stigg burns promotional first when other attributes match — preserves paid balance. |
| Treating overdraft as a regular paid grant for billing | It isn't billed separately. Bill on the *settling* purchase. |
| Trying to void an overdraft to "reset the deficit" | Can't. Settlement happens by granting new credits. |
| Designing UI to hide overdraft | Surface it. Customers should know they're in deficit; ops should see it for support. |
