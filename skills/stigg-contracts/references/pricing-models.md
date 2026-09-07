# Pricing models — which one a deal shape maps to

Every contract line item is priced by **one of the environment's existing pricing models**, and there is no
way to define pricing inline. `billing-and-invoices.md` covers the calls, the row keys and the charge type;
this file covers the choice itself — which model a deal shape maps to, and how to defend the choice when
the user pushes back.

Picking wrong here is silent. Every model accepts the same row shape and returns 200; the difference shows
up as an amount on an invoice. So each pattern below carries **the nearest wrong neighbour and the symptom
of picking it** — that is what makes the choice arguable in both directions.

---

## Resolve a model by `isPayAsYouGo` + shape, never by name

Five of the ten selectable models **share a name with their opposite-category twin**. There is a Fixed
"Flat rate" and a Usage "Flat rate", a Fixed "Tiered" and a Usage "Tiered", and so on:

| Shape | Fixed (`isPayAsYouGo: false`) | Usage (`isPayAsYouGo: true`) |
|---|---|---|
| Flat rate | ✓ | ✓ |
| Package | ✓ | ✓ |
| Tiered | ✓ | ✓ |
| Volume | ✓ | ✓ |
| Stairstep | ✓ | ✓ |

The name in the `GET /contracts/pricing-models` payload does not tell you which twin you're holding — only
`isPayAsYouGo` does. **Fixed** bills a committed quantity you state; **Usage** resolves quantity from
reported usage and requires a metered item. Matching on name alone is how a commitment gets priced as
consumption.

The list is the *environment's* models, which can be renamed or cloned, so treat the names below as
illustrative and the **shapes** as the contract. `services` models (Installation Fee, Platform Fee, Base
charge) are excluded from contract line items — the dashboard's own model picker filters them out too. The
Blank / Credit Blank / Payout Blank templates are for authoring new models in the dashboard, not for
pricing a line.

---

## The five shapes, on one set of numbers

Same rate table for all three tier shapes — **0–50 at $10, 50+ at $9** — and the same consumption,
**60 units**. What changes is the shape, not the data:

| Shape | How quantity resolves | 60 units | Row keys |
|---|---|---|---|
| **Flat rate** | every unit at one rate | 60 × $10 = **$600** | `price`, `quantity`¹ |
| **Package** | rounds **up** to whole packages | packageSize 50 → 2 packs × $10 = **$20** | `price`, `packageSize`, `quantity`¹ |
| **Tiered** | graduated — each tier bills only the units inside it | 50 × $10 + 10 × $9 = **$590** | `from`, `to`, `price`, `quantity`¹ |
| **Volume** | retroactive — one row fires and prices **every** unit | 60 × $9 = **$540** | same |
| **Stairstep** | a flat fee for the band; quantity is always 1 | 1 × $9 = **$9** | same |

¹ Fixed models only. A usage model **rejects** `quantity` — usage supplies it.

Stairstep is the odd one: its `price` is a band fee, not a unit rate, so a real stairstep table reads
$500 / $900, not $10 / $9. Reading a stairstep price as a unit rate is the largest-magnitude mistake on
this page.

The formulas each model computes, so any claim above is checkable:

```
Flat rate   quantity = usage
Package     amount   = price × CEILING(quantity / packageSize, 1)
Tiered      quantity = IF(usage >= from, IF(NOT(to), usage, MIN(usage, to)) - from, 0)
Volume      quantity = IF(AND(usage > from, OR(NOT(to), usage <= to)), usage, 0)
Stairstep   quantity = IF(AND(usage > 0, usage > from, OR(NOT(to), usage <= to)), 1, 0)
```

Two things that follow from them:

- **An open-ended top tier is `to: null`.** Every tier formula tests `NOT(to)`, so a missing upper bound is
  what makes the last tier unbounded — not a large number.
- **A Fixed tier model takes the same `quantity` on every row.** It is the committed total, and the formula
  splits it across the tiers; that is what the dashboard does when you type a quantity onto a fixed tiered
  table. Send it identically on each row, not divided between them.

---

## Fixed patterns

### 1. Flat platform / subscription fee
*aka platform fee, base fee, license fee*
- **Buying:** the right to use the product for a period, at one price, however much they use.
- **Tells:** one amount against a period — "$12,000 / year", "Platform fee", "Subscription".
- **Model:** Fixed · Flat rate — one row, `{ quantity: 1, price: 12000 }`.
- **Worked:** $12,000/yr on a yearly cycle in advance → one invoice per year.
- **Not Services · Platform Fee:** that model exists and is the obvious name match, but services models
  aren't selectable on a contract line item.
- **Ask:** is the fee flat for the whole term, or does it change year to year? (Changes → see Gaps.)

### 2. Per-seat / per-user
- **Buying:** a contracted number of users at a fixed rate each.
- **Tells:** "25 seats × $30/mo", "per user per month", a quantity column beside a unit price.
- **Model:** Fixed · Flat rate — `{ quantity: 25, price: 30 }`.
- **Worked:** 25 × $30 = $750/mo.
- **Not Usage · Flat rate:** seats sold are a commitment, not a measurement. A usage model would bill
  whatever seat usage was reported and ignore the 25 they bought.
- **Ask:** billed for seats purchased, or seats actually active? Active seats need a metered seat item and a
  usage model.

### 3. Included / bundled at $0
- **Buying:** something sold as part of the deal at no separate charge.
- **Tells:** the amount column reads "Included", "Bundled", "N/C", or a dash.
- **Model:** Fixed · Flat rate — `{ quantity: n, price: 0 }`.
- **Worked:** "Priority Support — Included" → one row at price 0, which appears on the invoice at $0.
- **Not dropping the line:** the invoice then carries no record that it was in scope, and the entitlement
  behind it doesn't exist. An "included" support tier with nothing behind it surfaces later as a dispute.
- **Ask:** nothing — but confirm each included line has an *entitlement*, not just a $0 charge.

### 4. Seat volume discount (retroactive)
- **Buying:** a per-seat rate that drops for **everyone** once they cross a threshold.
- **Tells:** "1–50 seats: $30; 51+: $25", with wording like "all seats at the applicable rate" or "blended".
- **Model:** Fixed · Volume — `{ from: 0, to: 50, price: 30 }`, `{ from: 50, to: null, price: 25 }`, the
  same `quantity` on both rows.
- **Worked:** 60 seats → only the second row fires → 60 × $25 = **$1,500**.
- **Not Tiered:** Tiered bills 50 × $30 + 10 × $25 = $1,750. Both are real deals; the gap is $250/mo and
  there is no error either way.
- **Ask:** at 60 seats, do all 60 get $25, or do the first 50 stay at $30?

### 5. Graduated seats
- **Buying:** band-by-band rates — earlier seats keep their higher price.
- **Tells:** "first 50 at $30, next 50 at $25", "incremental", "marginal rate".
- **Model:** Fixed · Tiered — the same rows as #4.
- **Worked:** 60 seats → 50 × $30 + 10 × $25 = **$1,750**.
- **Not Volume:** the mirror image of #4.
- **Ask:** the same single question as #4 — it settles both.

### 6. Plan band by size
*aka tiered plan, size-based plan*
- **Buying:** a flat price for a band; the price is identical anywhere inside it.
- **Tells:** "0–50 users: $500/mo; 51–200: $900/mo" — the figures are totals, not rates.
- **Model:** Fixed · Stairstep — `{ from: 0, to: 50, price: 500 }`, `{ from: 50, to: null, price: 900 }`.
- **Worked:** 60 users → second band → **$900/mo**, flat.
- **Not Volume:** Volume multiplies — 60 × $900 = $54,000 instead of $900.
- **Ask:** is $900 the price for the whole band, or per user within it?

### 7. Block / bundle purchase
- **Buying:** units in fixed-size packs, where part of a pack costs a whole pack.
- **Tells:** "per 1,000 credits", "blocks of 10", "packs of".
- **Model:** Fixed · Package — `{ packageSize: 1000, price: 44, quantity: 2500 }`.
- **Worked:** 2,500 units in packs of 1,000 → CEILING(2.5) = 3 packs × $44 = **$132**.
- **Not Flat rate:** with the pack price it bills 2,500 × $44; with a per-unit price it loses the rounding
  and bills 2.5 packs.
- **Ask:** does a partial pack bill as a full pack? If not, this is per-unit, not packaged.

### 8. Annual prepaid commitment
- **Buying:** a year or more paid up front rather than monthly.
- **Tells:** "paid annually in advance", "prepaid", "annual commitment"; one amount covering 12 months.
- **Model:** any Fixed shape, plus `billingCycleUnit: "yearly"` and the in-advance charge type.
- **Worked:** a 2-year contract on a yearly cycle produces two invoices, one per year.
- **Not a monthly cycle:** if the form promises two invoices, monthly gives twenty-four. Check the cycle
  against the payment schedule on the form, not against the term.
- **Ask:** how many invoices does the customer expect over the term?

### 9. One-time implementation / onboarding fee
- **Buying:** a single non-recurring charge.
- **Tells:** "one-time", "setup fee", "implementation", "professional services", no recurrence stated.
- **Model:** Fixed · Flat rate with `billingCycleUnit: "one_time"`.
- **Worked:** $40,000 once, alongside a recurring platform fee on the same contract.
- **Not Services · Installation Fee:** excluded from contract line items despite being the name match.
- **Ask:** invoiced at contract start, or on a milestone? Milestones aren't modellable — see Gaps.

---

## Usage patterns

### 10. Pure pay-as-you-go per unit
- **Buying:** nothing up front — they pay for what they consume.
- **Tells:** "as incurred", "in arrears", "$0.002 per request", and the giveaway: **no amount in the
  total column**.
- **Model:** Usage · Flat rate — `{ price: 0.002 }`. No `quantity`.
- **Worked:** 3.2M requests reported → 3,200,000 × $0.002 = $6,400 on that period's invoice.
  A rate quoted *per 1K* of something is packaged, not flat — that's #15.
- **Not omitting the line:** it contributes nothing to the contract subtotal, so it is invisible if you work
  from the totals. It is the line most often dropped — and the usage then never bills at all.
- **Ask:** is there any allowance before this rate starts? If yes, this is #11, not #10.

### 11. Committed volume + overage ★
*aka commit-and-burst, prepaid allowance with true-up*
- **Buying:** a block of usage paid up front, plus the right to keep going past it at an agreed rate.
- **Tells:** two lines for the same item — one with an amount, one reading "As incurred" / "In arrears";
  "additional X once the committed allowance is exhausted"; "overage rate".
- **Model:** Usage · Tiered, two rows — `{ from: 0, to: 600000, price: 0 }`,
  `{ from: 600000, to: null, price: 0.065 }`, in arrears. The commitment itself is a **separate Fixed
  line**, in advance.
- **Worked:** 750,000 consumed → tier 1 = 600,000 × $0 = $0; tier 2 = 150,000 × $0.065 = **$9,750**.
- **Not Usage · Flat rate at $0.065:** `quantity = usage`, so it bills all 750,000 — including the 600,000
  the customer already prepaid. $48,750 instead of $9,750. The invoice looks plausible; nobody catches it
  until the customer does.
- **Not Usage · Volume:** Volume reprices every unit at the reached tier's rate — all 750,000 at $0.065.
  The same failure by a different formula.
- **Ask:** is the committed volume already paid for on another line? Does exceeding it reprice everything,
  or only the excess?

### 12. Declining usage rates (graduated)
- **Buying:** cheaper units as they scale, with earlier units keeping their price.
- **Tells:** a rate card with several bands and no commitment — "first 1M at $0.01, next 4M at $0.008,
  thereafter $0.006".
- **Model:** Usage · Tiered, one row per band, `to: null` on the last.
- **Worked:** 6M units → 1M × $0.01 + 4M × $0.008 + 1M × $0.006 = **$48,000**.
- **Not Volume:** 6M × $0.006 = $36,000 — $12,000 apart, both plausible on their face.
- **Ask:** do earlier units keep their higher rate, or does the whole volume reprice at the rate reached?

### 13. Retroactive volume discount on usage
- **Buying:** one blended rate for everything, set by the total volume reached.
- **Tells:** "all units priced at the applicable tier", "blended rate", "retroactive discount".
- **Model:** Usage · Volume, one row per band.
- **Worked:** 6M units → 6M × $0.006 = **$36,000**.
- **Not Tiered:** the mirror image of #12.
- **Ask:** the same question as #12.

### 14. Usage buckets
*aka stairstep usage, request bands*
- **Buying:** a flat fee determined by which usage band they land in.
- **Tells:** "up to 10K requests: $99/mo; 10K–50K: $399/mo" — totals, not rates.
- **Model:** Usage · Stairstep.
- **Worked:** 25K requests → **$399** flat, wherever in the band they land.
- **Not Volume:** 25,000 × $399 — off by four orders of magnitude.
- **Ask:** is $399 the whole charge for the band?

### 15. Packaged usage
- **Buying:** consumption metered but priced in blocks, rounded up.
- **Tells:** "per 1,000 tokens", "per 1M requests", "billed in increments of".
- **Model:** Usage · Package — `{ packageSize: 1000, price: 0.5 }`.
- **Worked:** 2,500 tokens → CEILING(2.5) = 3 blocks × $0.50 = **$1.50**.
- **Not Flat rate at the block price:** it loses the rounding and prices per unit.
- **Ask:** does a partial block round up to a full one?

### 16. Credits: prepaid grant + priced top-ups
- **Buying:** a balance of credits bought up front, plus more at a stated rate once it runs out.
- **Tells:** a credit grant with an amount **and** a top-up rate; "top-ups", "additional credits",
  sometimes a premium on the top-up rate.
- **Model:** two lines on the same item — Fixed (the grant, in advance) plus Usage · Tiered (top-ups, in
  arrears, first tier $0 up to the granted volume). This is #11 with credits as the unit.
- **Worked:** 600K credits at $0.044238 = $26,543 in advance; overage tier at $0.065 beyond 600K.
- **Not one Usage line:** it double-charges the grant — see #11.
- **Ask:** does the balance reset each period, and does anything unused roll over? Rollover isn't
  modellable — see Gaps.

### 17. Per-outcome / per-resolution
*aka outcome-based, per-workflow, per-action*
- **Buying:** a unit of delivered work — a resolved ticket, a completed workflow, a booked meeting — not a
  unit of infrastructure.
- **Tells:** "per resolution", "per successful X", "per completed", "outcome-based".
- **Model:** Usage · Flat rate, or Usage · Tiered when the rate declines, on a metered item that counts
  outcomes.
- **Worked:** 1,200 resolutions × $0.99 = **$1,188**.
- **Not per-seat:** the value follows volume of work, not headcount — though such deals often carry both, a
  Fixed seat line and a usage outcome line.
- **Ask:** what event marks an outcome, and is it already metered? A usage model on a non-metered item is
  rejected outright.

### 18. Per-agent / per-bot
- **Buying:** a running AI agent, priced like a seat.
- **Tells:** "per agent per month", "per bot", "per assistant".
- **Model:** Fixed · Flat rate — `{ quantity: agents, price: rate }`. Usually paired with a usage line for
  what the agents consume.
- **Worked:** 5 agents × $200 = $1,000/mo, plus metered token usage on a second line.
- **Not a usage model:** the agent count is contracted, not measured — unless the deal explicitly bills
  active agents.
- **Ask:** is the count fixed for the term, or does it flex month to month?

---

## Gaps — what has no model, and what to do instead

Two of these have a workaround; the rest are things to name and stop on rather than approximate.

**Ramped / escalator pricing** (year 1 at $X, year 2 at $Y). **One period per contract.** A line item
carries a billing *cycle*, never its own period, and `PATCH .../billing/terms` writes a single window
across every line. Two yearly lines at different prices both span the whole term, so a two-year contract
bills both amounts in both years — four invoices, not two. A table repeated per period is one contract
*only when the amounts are identical*.

**Model it as one contract per period, each with that period's own billing period and prices.** The
consequence is that invoices are issued per contract rather than from one — which for a multi-year ramp is
what the customer already expects, since each year is a separate invoice anyway. Two things follow:

- **Check each contract's summary against that period's subtotal, not the order form's grand total.** Total
  contract value is per contract; the grand total is the sum across the chain.
- **Each contract publishes, amends and renews on its own** — its own legal entity, payment account and
  publish call. Say that when proposing the split, because it is real operational overhead the user is
  agreeing to, not a formatting detail.

**Percentage-of-value / revenue share / take rate.** No template — the Percentage model is no longer part
of the seeded set. Needs a model authored from Blank in the dashboard first.

**Hourly / time-and-materials.** Same: no Hourly Fee template. Fixed · Flat rate with hours as `quantity`
bills a known number of hours; it cannot price hours as they are logged.

**A dollar minimum-spend floor** ("the greater of $10,000 or actual usage"). Not expressible — no formula
spans line items to take a maximum. A *volume* commitment is expressible (#11); a *spend* floor is not.
Only model it as a Fixed floor plus a tiered overage if the parties agree the floor converts to a volume.

**Credit rollover caps and top-up premiums.** No entitlement and no contract field behind either. Flag them
as contract-only terms someone must honour manually.

**Milestone-based invoicing.** A line item's cycle is a recurrence, not a schedule of dates, so there is no
way to say "bill $40,000 at kickoff and $60,000 at go-live" on one contract. The same split works — a
contract per milestone, each with a billing period covering that milestone and a one-time line — but here
the invoices-per-contract consequence is worth raising explicitly: a customer who signed one statement of
work may not expect two contracts behind it, and the milestone dates have to be known up front to serve as
billing periods. If they aren't, this is manual invoicing, not a modelling problem. Ask before splitting.

---

## Before you commit to a model

1. Resolved by `isPayAsYouGo` + shape, not by name.
2. Every usage-priced line is on a **metered** item, and carries no `quantity`.
3. Each tier table's top row has `to: null` if the deal is open-ended.
4. The summary's total reads back equal to the order form's own total — that check catches a wrong shape
   faster than any argument about which model is right.
