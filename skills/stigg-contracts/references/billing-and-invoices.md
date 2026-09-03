# Billing contracts and invoices

The half of a contract that produces invoices — whether you picked **billing only** or **both**.
`SKILL.md` covers the mental model, the three ways to start, and the provisioning half; this is the billing
detail.

Everything here needs a contract that **has a billing contract** — created with `setupBilling: true`, or
enabled later on a provisioning-only contract by updating it with `setupBilling: true` (one-way: billing is
never removed by an update). Every billing endpoint refuses a provisioning-only contract and says so.

---

## Step 1 — Have the conversation first

**An order form is not a spec.** Pricing models are rarely stated outright, order forms contradict
themselves, and they contain outright mistakes — a real ACME order had two blocking conflicts and a support
tier that had been sold with nothing behind it. A wrong guess here bills a real customer wrongly, and the
mistake surfaces on an invoice, in front of that customer.

So **before any write call**, walk these four and confirm each with the user. Ask; don't infer.

**Ask in their words, not the API's.** Every question here is answered from an order form and a dashboard,
so that is the vocabulary to use: "set up billing" rather than `setupBilling: true`, "in advance" rather
than `IN_ADVANCE`, "the legal entity that issues the invoice" rather than `legalEntityId`. A field name in a
question is noise the user can't verify against anything they're looking at — and it invites them to
approve a shape rather than a deal. Keep the field names in the request and out of the conversation.

**1. Terms — and the exceptions.** Period, currency, payment terms. Then, explicitly: *does any line item
deviate?* Per-item cadence is real and common (ACME's Data Credits reset annually while the rest bills
monthly), and the API models billing cycle, payment time, tax, discount and net terms **per item** — so a
single contract-level assumption is usually wrong somewhere.

**2. Line items — one at a time.** For each: which catalog item it is, **which pricing model it uses**, and
the row values that model needs. Committed-volume-with-overage in particular has to be pinned down: what is
committed, at what rate, what the overage rate is, and what happens past it. Restate the mapping back in
the user's own words — "1.4M Actions committed at $0, overage at $0.0060 beyond that, as a tiered model" —
and get agreement before building anything.

**3. Legal entity — and its payment account.** Call `GET /contracts/legal-entities` and check **both**:
that an entity exists, and that it has a **payment account**. A default entity with an empty
`paymentAccounts` list is the common half-configured state, and it is easy to miss because the entity itself
looks fine:

```json
[ { "id": "...", "name": "Acme", "isDefault": true, "paymentAccounts": [] } ]
```

**If the list is empty, stop and say so.** There is no API to create a payment account — an entity can be
created, an account cannot — so someone has to add one in the dashboard, on the legal entity itself.
Building the contract anyway produces a draft that looks complete and then cannot be published, which is a
worse place to discover it. Treat an entity with no accounts as a blocker to raise in step 1, not a detail
to fill in later.

**Once an account exists, attach it over the API** — `PATCH .../billing/settings` with its
`paymentAccountId` (step 3). Only *creating* the account needs the dashboard; selecting it for a contract
does not, so don't send the user through the contract wizard for it.

**4. Customer details.** The billing customer on the contract, cross-checked against the order form, which
normally carries them. A mismatch between the entity named on the paper and the Stigg customer is a common
and expensive error, and it costs one question to catch.

**Sequencing, when the order form covers both halves.** Create the contract **first**, with its
subscriptions as `newSubscription` entries on the create call — then price the billing lines. Not
subscription-first: only the create call can provision, so a subscription provisioned standalone can be
*attached* later but never provisioned *into* the contract, and it sits live and unattached in between.
Elicit → create the contract with its subscriptions → verify entitlements → price the billing lines.

**Create hands you the contract id — hold it for the rest of the flow.** The `201` carries the whole
contract, `contractId` included:

```json
{ "data": { "contractId": "contract-hd74qrhr9kn253qy8uxxl", "state": "ACTIVE", "billingState": "DRAFT", ... } }
```

That is the id every later call keys on — `/contracts/{id}/billing/items`, `/terms`, `/settings`,
`/summary`, `/invoice-schedule`, `/publish`. **Create once, then reuse it.** Editing a contract never
creates another one; only `POST /contracts` does.

**Pass your own `contractId` on create and the call becomes idempotent** — send the same one again and you
get the existing contract back, not a second one:

```bash
curl -X POST ".../contracts" -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{ "customerId": "customer-acme", "contractId": "contract-acme-po-4821",
        "name": "Acme 2026", "poNumber": "PO-4821", "setupBilling": true, "subscriptions": [] }'
```

**Use one whenever a retry is possible** — which, building a contract from an order form, it always is.
Derive it from something stable in the deal (the PO number, the order number) so the same order always maps
to the same key. Omit it and an id is generated for you, but then nothing can tell a retry apart from a new
contract: a timeout, a fresh session picking up the same order form, or re-running your own steps creates a
*second* contract — publishable, and able to invoice the customer twice. There is no delete; the most anyone
can do is archive, which cancels it.

The repeat returns the existing contract **unchanged** — it is an idempotency key, not an upsert, so a
retry never overwrites a contract that has been edited since. Anything else in the retry's payload is
ignored. To actually change a contract, use `PATCH /contracts/:id` for its name, PO number and window, and
the billing endpoints for its lines.

Working on a contract you didn't create, with no key to hand? `GET /customers/:customerId/contracts`,
matched on `poNumber` or name.

**On contradictions: surface them, don't resolve them silently.** ACME's Actions line could not be both an
unlimited entitlement and a priced commitment. The right move was to raise it and have the plan changed —
not to pick whichever reading made the API call succeed.

### Confirm the whole reading before the first write

**Read it back and get agreement before you create anything.** Every line item, its pricing model and its
rows, the period, the payment terms, the legal entity, the customer. There is a review step before
publishing, but by then the contract exists and its line items have to be edited or deleted rather than
simply written — and a published contract can only be amended or reopened. Getting it wrong is expensive in
a way most API mistakes aren't: billing and entitlements are the two things a customer notices immediately,
one on an invoice and the other as access they do or don't have. The confirmation costs one message.

### What order forms hide — check each of these

Real order forms lose line items in predictable places. Each of these has been missed by an agent reading
a real one:

**The PO number is usually not on the form.** Order forms carry several identifiers — an order number, an
agreement or contract number, and a **purchase order reference** that is very often `N/A`. `poNumber` means
the *customer's own* purchase-order reference. If the form says N/A, leave it unset and say so; putting the
order number there records the seller's identifier as the buyer's, which is the field AP will reconcile
against. The order number belongs in the contract `name` if it belongs anywhere.

**Usage-based lines carry no amount, so they vanish.** A line whose total column reads *"In arrears"* or
*"As incurred"* instead of a figure contributes nothing to the subtotal — so it is invisible if you work
from the totals, and it is the line most often dropped. It is still a line item: price it with a
pay-as-you-go model, in arrears. Drop it and the overage is simply never billed, which nobody notices until
the customer exceeds their commitment.

> *"Data Credit top-ups (overage) · Usage-based · $0.065 / credit · As incurred · In arrears"* — a real
> line, on a real order form, that an agent read past because the amount column said "In arrears".

**"Included" lines are still lines.** Seats, support tiers and anything else priced *Included* belongs on
the contract **at price 0**. It is what the customer was sold: it should appear on the invoice, and its
entitlement should exist. Dropping it loses the record that it was in scope — and an "included" support
tier with nothing behind it is exactly the kind of thing that surfaces months later as a dispute.

**A table repeated per period is one contract, not two — if the amounts are identical.** An order form that
lists the *same* charges under *Period 1* and *Period 2* is a single contract whose billing cycle is yearly:
the term spans both, the TCV is the sum, and the billing schedule tells you how many invoices to expect
(two, for a two-year annual deal). Don't create a contract per period. Check your line items'
`billingCycleUnit` against that schedule — if the form promises two invoices and your cycle is monthly, one
of the two is wrong.

**But if the amounts differ per period, it is one contract per period.** A ramp or escalator — year 1 at
$X, year 2 at $Y — has no single-contract form: a line item carries a billing *cycle*, never its own
period, and the terms call writes one window across every line. Two yearly lines at different prices both
span the whole term, so a two-year contract bills both amounts in both years: four invoices, not two.
Raise it, and model each period as its own contract with that period's own billing period and prices —
invoices are then issued per contract, which for a multi-year ramp is what the customer already expects.
Check each contract's summary against **that period's** subtotal rather than the grand total. Milestone
billing splits the same way. See `references/pricing-models.md`.

**Quantities that read "Unlimited".** Not a number, and not a quantity of 1 either. An unlimited line is
either an `Included` line at price 0 (above) or an entitlement with no limit on the subscription — never a
billing quantity. Ask which.

---

## Step 2 — Pick pricing models; never invent a table

A line item is always priced by **one of the environment's existing pricing models**. There is no way to
define pricing inline, and that is deliberate: the model owns the table's shape — its columns and the
formulas that compute amounts — and those are what the billing engine and the invoice renderer both read.

**Which model a deal shape maps to — and how to defend the choice — is `references/pricing-models.md`.**
It carries the five shapes on one set of numbers, eighteen deal patterns with the row values each needs,
the nearest wrong neighbour for each and the symptom of picking it, and the shapes that have no model at
all. Read it before choosing: every model accepts the same rows and returns 200, so a wrong shape surfaces
as an amount on an invoice rather than an error.

```bash
curl "https://api.stigg.io/api/v1/contracts/pricing-models" -H "X-API-KEY: <YOUR_API_KEY>"
```

```json
[ { "id": "pm-flat", "name": "Flat rate", "categoryName": "fixed", "isPayAsYouGo": false,
    "editableInputs": ["price", "quantity"] },
  { "id": "pm-tiered", "name": "Tiered", "categoryName": "usage", "isPayAsYouGo": true,
    "editableInputs": ["from", "to", "price"] } ]
```

Two fields do the work:

- **`isPayAsYouGo`** — a usage model resolves quantity from reported usage; a fixed model bills a committed
  quantity. The model decides this and the charge timing that goes with it (usage bills in arrears), so
  there is no separate "metered" flag to pass. **But it only applies to metered items** — see below.
  **A pay-as-you-go model takes no `quantity`**, and its `editableInputs` won't list one: the quantity is
  whatever usage is reported, so a quantity sent with the line would never be what gets billed. The
  dashboard has no field for it either — that cell shows the metered item. Sending one is rejected. To
  price a commitment, use a fixed model, or express the committed volume as a tier's `from`/`to` (which is
  how ACME's 1.4M Actions are modelled, below).
- **`editableInputs`** — the only row values you may set for that model, **named exactly as `rows` accepts
  them**. The whole vocabulary is `quantity`, `price`, `from`, `to`, `packageSize` — nothing else. A value
  for an input the model doesn't expose is **ignored, not rejected**, so one row shape works everywhere;
  but that also means a typo silently does nothing.

  **There is no `usage` and no `item` row field.** The visible count is `quantity` (internally it fills the
  model's `usage` column), and the item is set once per line item via `itemName`, never per row. A row
  carrying any other key is rejected outright.
**You decide how many rows.** A model's own table is a template, not a constraint: send as many rows as the
deal has and the table is resized to match — extra tiers are appended, surplus ones dropped. Two rows for a
commitment plus an overage, five for five tiers. Send none and the model's own rows are left as they are.

### Where the contract period lives

Stigg calls it the contract's **start and end date**; the billing contract's Terms step calls it the
**Billing period**. Same dates, held in two places — the contract's own activation window, and the period
on each line item that the billing contract derives its period from.

`PATCH .../billing/terms` writes **both**, so you state the term once: the line items get the billing
period, and the contract's `activationStartDate` / `activationEndDate` are set to match. Creating the
contract with dates, or passing them when adding items, also sets them.

Send a full ISO instant (`2026-06-01T00:00:00.000Z`); only the **calendar day** is stored, so a time-of-day
is ignored rather than shifting anything. Reading a period back: the contract's own window comes from
`GET /contracts/:id`, and the line items' from the billing summary.

### Charge type

The dashboard calls this **Charge type** and offers exactly two options. The API takes the same two words:

| Dashboard | Send | Means |
|---|---|---|
| **In advance (Prepaid)** | `"IN_ADVANCE"` | Billed at the start of the period, before the customer consumes it. The normal choice for a committed fee. |
| **In arrears (Postpaid)** | `"IN_ARREARS"` | Billed after the period, once consumption is known. Required for pay-as-you-go, where the quantity isn't knowable up front. |

Say "in advance" and "in arrears" when talking to a user — those are the words on their screen — and send
`IN_ADVANCE` / `IN_ARREARS`. Nothing else is accepted. Don't invent hyphenated forms like "before-use", and
don't write "advanced": the option is "in advance".

> If you have seen `DURING_USE`, `AFTER_USE` or `BEFORE_USE` elsewhere in Stigg, those are the billing
> engine's internal names and **this API does not take them.** `BEFORE_USE` in particular reads like the
> obvious match for "in advance" and means something else again; it was dropped as an option, and no
> request can reach it.

Omit the charge type and the model picks: a pay-as-you-go model defaults to in arrears, a fixed one to in
advance. **Pay-as-you-go is always in arrears** — usage can't be charged before it happens — so pairing it
with `IN_ADVANCE` is rejected rather than quietly corrected.

### Pay as you go needs a metered item

A pay-as-you-go model resolves quantity from **reported usage**, so the item has to be one that is metered.
Pairing a usage model with a plain catalog item is rejected — it would produce a line that can never compute
an amount:

> Line item "Platform Actions" uses a pay-as-you-go pricing model, which requires a metered item.
> "Actions" is not a usage product: price it with a fixed pricing model, or meter the item first.

The dashboard enforces the same rule structurally: the "Pay as you go" toggle only appears on a metered
item. A plain item's editor starts at Charge type, with no toggle at all. So when an order form describes
consumption billing, check that the item is actually metered before reaching for a usage model — and if it
isn't, either meter it or price the commitment with a fixed model.

`services` models (installation, platform and base fees) are not selectable for a contract line item and
don't appear in the list.

---

## Step 3 — Add line items

```bash
curl -X POST "https://api.stigg.io/api/v1/contracts/contract-acme-2026/billing/items" \
  -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{
    "legalEntityId": "entity-workwise",
    "lineItems": [
      { "name": "Platform Fee", "externalId": "acme-platform-fee",
        "itemName": "Platform Fee", "pricingModelId": "pm-flat",
        "rows": [ { "quantity": 1, "price": 12000 } ],
        "billingCycleUnit": "yearly", "paymentTime": "IN_ADVANCE" },
      { "name": "Platform Actions", "externalId": "acme-actions",
        "itemName": "Actions", "pricingModelId": "pm-tiered",
        "rows": [ { "from": 0, "to": 1400000, "price": 0 },
                  { "from": 1400000, "to": null, "price": 0.0060 } ] }
    ]
  }'
```

### The line item fields, in full

| Field | Notes |
|---|---|
| `name` | **Required.** The line's title on the contract and the invoice. |
| `externalId` | Your own id for the line. Optional, but **it is how you address the line later** — see below. |
| `itemName` | The catalog item to price. Resolved by name, **created if nothing matches**. Omit and the model's placeholder item is used. |
| `pricingModelId` | From `GET /contracts/pricing-models`. Omit for the Flat rate model. |
| `rows` | The row values, in order. Keys: `quantity`, `price`, `from`, `to`, `packageSize` — nothing else, and no `quantity` on a pay-as-you-go item. |
| `billingCycleUnit` / `billingCycleCount` | `monthly`, `yearly`, `quarterly`, `weekly`, `one_time`; count defaults to 1. |
| `paymentTime` | Charge type: `IN_ADVANCE` or `IN_ARREARS`. Omit and the model picks. Pay-as-you-go rejects `IN_ADVANCE`. |
| `discount` / `tax` | Percentages, per line. |
| `netTerms` | Payment terms in days, per line. |
| `separateInvoice` | Bill this line on its own invoice instead of the contract-wide one. |

Contract-level, alongside `lineItems`: `currency`, `legalEntityId`, `activationStartDate`,
`activationEndDate`. The dates default to the period already on the contract's other items.

### Terms and settings, without touching the line items

The period, currency and payment terms are contract-wide, and so are the legal entity and payment account
— but all of them are **stored on the line items**. Two endpoints write them across every line, so you
don't have to `PATCH` each one:

```bash
curl -X PATCH ".../contracts/contract-acme-2026/billing/terms" \
  -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{ "activationStartDate": "2026-06-01T00:00:00.000Z",
        "activationEndDate": "2028-05-31T00:00:00.000Z",
        "currency": "USD", "netTerms": 30 }'

curl -X PATCH ".../contracts/contract-acme-2026/billing/settings" \
  -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{ "legalEntityId": "entity-workwise", "paymentAccountId": "acct-boa-wire",
        "note": "PO 20260511-2-R2" }'
```

Only the fields you send are written; the rest are left as they are. Both apply to **every** line item, so
they are the wrong tool for a per-item exception — set those on the item itself (Step 3), and set them
*after* the contract-wide values, or the sweep will overwrite them.

Both need the contract to have line items already: they have nothing to write to otherwise, and say so
rather than silently succeeding. Both are draft-only.

`paymentAccountId` belongs to the legal entity, and setting it is what makes the contract collectable —
`GET /contracts/legal-entities` lists each entity's accounts alongside it. An id that doesn't exist is
rejected by name, so a wrong guess is cheap to spot; an entity with **no** accounts is a dashboard fix, not
an API one.

### Addressing a line later — read this before you add

`PATCH` and `DELETE` key on a **ref**, and there are exactly two things that work as one:

- the **`externalId`** you assigned when adding the line, or
- the **`itemId`** the summary reports for it.

So either assign `externalId`s up front, or read `GET /contracts/:id/billing/summary` afterwards — its
`items[]` carry `itemId` and `name`. Every mutation returns that same summary, so adding lines hands you
their ids immediately. **There is no `GET .../billing/items`**; the summary is the item listing.

### Adding vs updating

Adding is **additive** — it never replaces what's there — so calling it twice with the same line creates
**two** lines. Before adding to a contract you didn't just build, read the summary and check by name:

- Not there → `POST .../billing/items`
- There, and the pricing needs changing → `PATCH .../billing/items/:ref` with the full row list
- There, and it shouldn't be → `DELETE .../billing/items/:ref`, or
  `DELETE .../billing/items/:ref/rows/:rowIndex` for a single tier

`PATCH` replaces the line's **whole** row list: extra entries append tiers, missing ones drop the surplus.
Send the full set you want, not a delta.

## Step 4 — Check the totals, then publish

```bash
curl "https://api.stigg.io/api/v1/contracts/contract-acme-2026/billing/summary" -H "X-API-KEY: <YOUR_API_KEY>"
curl -X POST "https://api.stigg.io/api/v1/contracts/contract-acme-2026/billing/publish" -H "X-API-KEY: <YOUR_API_KEY>"
```

The summary is subtotal, discount, tax, **total contract value**, a per-item breakdown, and the terms and
settings read back (`activationStartDate`, `activationEndDate`, `netTerms`, `legalEntityId`,
`paymentAccountId`) — computed the same way the dashboard's review step computes it. Each term is reported
only when **every** line agrees on it, so a `null` means either it was never set or the lines differ; a
per-item exception is legitimate, and the summary won't pretend it's uniform. **Read it against the order form's own total before
publishing.** After publishing, these are the figures the customer is invoiced.

The figures are real on a **draft** — they're computed from the line items' own pricing tables, the same
way the dashboard shows live totals while you're still editing. So this is a genuine check, not a
placeholder: if the total doesn't match the order form, a rate or a quantity is wrong, and it is far
cheaper to find now than on the first invoice.

A usage-priced line contributes only what its committed rows price to; consumption beyond that appears on
invoices later, not here. Say that when presenting a total for a contract with a pay-as-you-go line, rather
than implying the figure is what the customer will pay.

**Publishing is what starts invoicing.** Only a draft can be published; a contract whose billing is already
live is rejected rather than silently reset.

### Check the schedule, not just the total

The summary says whether the money adds up. It cannot say whether the customer will be **invoiced when the
order form says** — and an order form usually states its own billing schedule (*n* invoices, on these dates,
for this amount). That comparison is its own endpoint:

```bash
curl ".../contracts/contract-acme-2026/billing/invoice-schedule" -H "X-API-KEY: <YOUR_API_KEY>"
curl ".../contracts/contract-acme-2026/billing/invoice-schedule?count=1" -H "X-API-KEY: <YOUR_API_KEY>"
```

```json
{ "isPreview": true, "invoiceCount": 2, "total": 64272,
  "invoices": [
    { "invoiceNumber": "INV-AA-17", "state": "CONTRACT_DRAFT", "issueDate": "2026-09-02...",
      "dueDate": "2026-10-02...", "periodStart": "2026-09-02...", "periodEnd": "2027-09-01...",
      "total": 32136, "lineItems": [ ... ] } ]}
```

Each invoice carries the **period it covers**, its issue and due dates, its total and its line items.
`count` returns only the first N by issue date — a two-year monthly contract otherwise returns 24, and the
first one or two are usually what you're checking.

**Why it catches what the total can't.** Line items that agree on the money but disagree on *when* they
bill are invoiced separately. A contract whose total is exactly right can still produce twice the invoices
the order form promises, each covering the wrong span — most often a $0 invoice a day before the real one.
The total is identical either way; the schedule is where it shows.

Read it against the order form's billing schedule: **the invoice count first**, then the dates, then the
amounts. A count that doesn't match means a line item's period or cadence is wrong, not its price.

`isPreview` tells you which you're looking at: `true` on a draft, where the documents are regenerated from
the contract as it stands, and `false` on a published contract, whose invoices are returned untouched.
Reading the schedule of a draft is safe — regeneration only ever replaces unissued drafts.

### What publishing requires

Publishing generates the contract's invoices, and it refuses if the contract can't produce one. Three
things have to be in place, all of them set in the steps above:

| Missing | Set it with |
|---|---|
| The **billing period** | `PATCH .../billing/terms` (or dates on create / add-items) |
| The **payment terms** | `PATCH .../billing/terms` — `netTerms` |
| The **legal entity** | `PATCH .../billing/settings` — and it needs a payment account |

A refusal comes back as `BillingContractOperationRejected` naming which one is missing and the call that
fixes it. Check the summary read-back before publishing (Step 4) and none of this should bite.

Publishing returns the contract with `billingState: "ACTIVE"` and its `nextInvoice`. Confirm the invoices
themselves with the invoice list below — that is the proof it worked, not the 200.

---

## The state model — where an agent gets stuck

Billing edits apply to a **draft**. Once published:

| Situation | What to do |
|---|---|
| Change a live contract's terms or items | **Amend it** — the contract stays live and its invoices are handled |
| Rebuild it from scratch | **Reopen it** as a draft, then publish again |
| `BillingContractEditBlocked` | **Terminal.** Issue a credit note (below) or a new contract |

A draft-only endpoint called on a live contract fails with `code: "BillingContractOperationRejected"` and a
message naming both recovery paths, rather than picking one. That code covers every rejection that is yours
to fix rather than to retry — not a draft, no line items yet, a pricing model that needs a metered item, an
id that doesn't exist — so read the message: it names the remedy, and retrying the same call never helps. That is deliberate: the dashboard's own save silently reopens a published contract *and
deletes its unsent documents*, which is fine behind a Save button and unacceptable for an API caller.

`BillingContractEditBlocked` fires when the contract's invoices have been sent, or carry usage reports.
**Retrying will never work** — the amendment has to become a credit note or a new contract.

---

## Invoices

Most of this already works; the filters do the heavy lifting.

```bash
# All of a customer's invoices, newest issue date first
curl "https://api.stigg.io/api/v1/customers/customer-acme/invoices?orderBy=issueDate&orderDir=DESC" \
  -H "X-API-KEY: <YOUR_API_KEY>"

# Just this contract's; just the open ones
curl "https://api.stigg.io/api/v1/customers/customer-acme/invoices?contractExternalId=contract-acme-2026&stateIn=OPEN" \
  -H "X-API-KEY: <YOUR_API_KEY>"
```

| Need | Call |
|---|---|
| List, filter by contract / state / issue-date range | `GET /customers/:id/invoices` |
| Open invoices | the same, `?stateIn=OPEN` |
| Line items | already in the list payload — no second call |
| One invoice | `GET /customers/:id/invoices/:invoiceRef` |
| The PDF | `GET /customers/:id/invoices/:invoiceRef/pdf` |
| Mark paid | `POST /customers/:id/invoices/:invoiceRef/paid` |
| Next invoice preview | `contract.nextInvoice`, on the contract itself |

Invoice states are `OPEN`, `PAID`, `CANCELED`.

### The PDF

Renders are asynchronous and cached transiently, so there is **no durable URL** — the file comes back inline
as base64. Three endpoints, because the right shape depends on who is waiting:

| Call | Use it when |
|---|---|
| `GET .../invoices/:ref/pdf` | You just want the file. Stigg starts the render and waits, bounded; a timeout says to retry, and a retry is cheap because a finished render is cached. |
| `POST .../invoices/:ref/pdf/generate` | You want to drive the wait yourself — start here. Answers `READY` when a render already existed. |
| `GET .../invoices/:ref/pdf/status` | …then poll this. `PENDING` while rendering, `READY` with the file attached. |

`PENDING` is a **status, not a 404** — a 404 would be indistinguishable from an unknown invoice. Reading
never triggers a render: the generate call does that.

For an agent, use the single `GET .../pdf` and decode `contentBase64` to a file. The generate/status pair
exists for interactive callers that want to show progress.

**Mark-paid is addressed by invoice id**, which is why it reaches a contract's invoices. The
subscription-scoped `POST /subscriptions/:id/invoice/paid` resolves a subscription first and cannot.

---

## Credit notes

How a change gets settled once a contract can no longer be amended.

```bash
curl -X POST "https://api.stigg.io/api/v1/customers/customer-acme/invoices/inv-123/credit-note" \
  -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{ "reasonCode": "ORDER_CHANGE" }'

curl "https://api.stigg.io/api/v1/contracts/contract-acme-2026/billing/credit-notes" -H "X-API-KEY: <YOUR_API_KEY>"
```

- **Credits the invoice in full.** Crediting part of an invoice means rewriting its pricing, which is a
  dashboard operation.
- **Usage-based lines are skipped** — they carry no fixed amount to credit against. An invoice made up
  entirely of them is rejected, with that as the reason.
- Reason codes include `ORDER_CHANGE` (the default), `ORDER_CANCELLATION`, `WAIVER`,
  `PRODUCT_UNSATISFACTORY`, `SERVICE_UNSATISFACTORY`, `USAGE_CREDIT`, `OTHER`.
- Issuing is addressed by **invoice** id, not contract — deliberately, since the contract may be
  un-editable precisely because it is the thing being settled.

---

## Dashboard-only

Say so and stop; don't reach for an endpoint that isn't there.

- **Partial credit notes** — crediting some of an invoice rather than all of it.
- **Cloning and renewing** a contract.
- **Regenerating** an invoice, and document generation beyond the invoice PDF.
- **Credit rollover and priced top-ups** — an order form may promise them (ACME's did: a 15% annual
  rollover cap and top-ups at a 30% premium), and neither has an entitlement or a contract field behind it.
  Flag them as contract-only terms someone must honour manually.

---

## The ACME order, end to end

An order restructured onto a custom Enterprise plan: 2-year term, USD, net 30; a $12,000 flat platform fee;
1.4M committed Actions with a $0.0060 overage; 200,000 Data Credits at $0.044238 resetting annually; legal
entity WorkWise, paid by wire; **total contract value $29,240.00**.

1. **Elicit** (step 1). The order form's Actions line said "unlimited" *and* carried a priced commitment —
   a contradiction to raise, not resolve. Its Data Credits reset annually while everything else bills
   monthly: a per-item exception.
2. **Look up** the pricing models and the legal entity. Flat rate for the platform fee; a tiered model for
   Actions (committed tier at $0, overage tier open-ended at $0.0060); Data Credits priced per unit with
   `billingCycleUnit: "yearly"` and `paymentTime: "IN_ADVANCE"`.
3. **Add the items**, then read the summary and check it against the order form's own total — the memo's
   $29,240.00. A gap here means a rate or a quantity is wrong, and it is far cheaper to find now.
4. **Publish**, then confirm the first invoice matches the expected schedule.

Note what is *not* in the contract: the seat SKU was eliminated and its value folded into the Action
commitment, so revenue became entirely consumption-linked. That is a commercial consequence to surface to
the user, not an API concern — but it is the kind of thing an order restructure hides, and worth reading
back before publishing.
