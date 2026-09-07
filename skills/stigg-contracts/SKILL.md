---
name: stigg-contracts
description: Use for contract-based (sales-led / enterprise) agreements in Stigg — the contract that is both an entitlement and an invoice. Covers all three ways to start one — provisioning only (entitlement provisioning, attaching custom subscriptions), billing only (negotiated terms, custom pricing, invoicing), or both — plus attaching or detaching subscriptions, amending, archiving, refreshing contract usage, and reading a customer's contracts and invoices. Triggers include "contract", "enterprise deal", "negotiated terms", "order form", "PO number", "provision access", "entitlement provisioning", "billing contract", "set up billing", "attach subscription to contract", "custom plan subscription", "next invoice", "customer invoices", "amendment", "renewal". Skip for catalog modeling (stigg-pricing-modeling), subscriptions with no contract (stigg-subscriptions), credits (stigg-credits), and per-entity budgets (stigg-governance).
---

# Stigg Contracts — Entitlements and Invoicing

A contract is the source of truth for a sales-led deal: what the customer can do, and what you charge for
it — an entitlement and an invoice, treated as both.

A contract **groups subscriptions** for one customer. Those subscriptions grant the entitlements — the
contract is the agreement they belong to, not a second entitlement mechanism. It can also carry a **billing
contract**, which produces invoices. The halves are independent — a contract with no billing contract still
provisions access.

> **The contract itself holds no commercial terms.** No price, quantity or commitment fields — only a name,
> PO number, activation window, and its subscription links. **Negotiated price and committed volumes live on
> the custom subscription** (`priceOverrides`, `charges`, `unitQuantity`, `minimumSpend` —
> `stigg-subscriptions`). Priced line items exist only on the billing half, below.

## Before You Start

Per the umbrella `stigg` skill: **search first.** Confirm shapes and field names against the docs.

> **No SDK surface.** Contracts are **not** in `@stigg/node-server-sdk` or `@stigg/js-client-sdk` — there
> is no `stigg.createContract(...)`. Use REST (canonical here, `https://api.stigg.io/api/v1`, header
> `X-API-KEY`) or GraphQL.

> **Availability.** Contract management is not enabled for every account. If the endpoints 404, it isn't
> turned on — ask Stigg rather than working around it.

## How would you like to start?

The dashboard asks exactly this, with two independent checkboxes — **Provision access** and **Set up
billing** — so there are **three** ways to start. Establish which first; it decides every step below.

| Choice | What the contract does | Send |
|---|---|---|
| **Provisioning only** | Groups custom subscriptions and grants their entitlements. **No invoices.** | `setupBilling: false` + `subscriptions` |
| **Billing only** | Priced line items and invoices, no subscriptions attached. | `setupBilling: true`, no `subscriptions` |
| **Both** | Entitlements *and* invoicing on one agreement — the usual shape for a signed order form. | `setupBilling: true` + `subscriptions` |

`setupBilling` **defaults to `true`**, so provisioning-only is the one you must ask for explicitly.

> **Vocabulary — recognize every name, say only one.** UI: **"Provision access"** / **"Set up billing"**.
> Docs: **"Entitlement provisioning"** / **"Billing contract"** / **"Both"**. API: `setupBilling`. Same
> three choices.
>
> **Never show the user a field name — `setupBilling` included.** Say "provisioning only", "billing only"
> or "both" and keep the flag in the request. That holds for every field below: the user confirms the deal
> in the words of their order form, not in JSON.

## The Operation Map

| Op | Purpose | Reference |
|---|---|---|
| **Create** | A contract, with or without a billing contract | inline below |
| **Attach / detach / amend / archive / read** | Membership, metadata, cancellation, lookups | inline below |
| **Set up billing** | Terms, settings, priced line items, totals, publish | below + `references/billing-and-invoices.md` |
| **Pick a pricing model** | Deal shape → model + rows, and the shapes with no model | `references/pricing-models.md` |
| **Invoices** | List, filter, one invoice, the PDF, mark paid | `references/billing-and-invoices.md` |
| **Credit notes** | Settle a change to a contract that can no longer be amended | `references/billing-and-invoices.md` |

# Provision Access

Group custom subscriptions and grant their entitlements. **No invoices** unless billing is also
set up.

## Prerequisites

**Only custom plans qualify** — custom plans and their add-ons, not self-service ones. Every subscription
must be **custom-priced**: a `CUSTOM` plan, or a PAID plan provisioned with `paymentCollectionMethod:
"NONE"`. A FREE-plan subscription can never join; model the deal as a custom plan
(`stigg-pricing-modeling`).

**Provision the customer's account first** — create the contract only once the customer exists.

Three server-side rules: **one subscription per product**, **one currency**, and every subscription
**belongs to the contract's customer**.

## Both: the order to do it in

From an order form, **create the contract first and let it provision the subscriptions** — provisioning
standalone leaves one live and unattached until a second write links it. Both contract-first shapes are one
transaction: `POST /contracts` with `newSubscription` entries, or `POST /contracts/:id/subscriptions` with
the same entries later.

So: elicit → **create the contract with its subscriptions** → verify entitlements → price the lines.

## Create the Contract

`setupBilling: false` is the difference — and it defaults to `true`, so send it explicitly.

```bash
curl -X POST "https://api.stigg.io/api/v1/contracts" \
  -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{
    "customerId": "customer-acme",
    "contractId": "contract-acme-po-4821",
    "name": "Acme 2026",
    "poNumber": "PO-4821",
    "setupBilling": false,
    "subscriptions": [ { "existingSubscriptionId": "subscription-acme-platform" } ]
  }'
```

Each entry is **exactly one of** `existingSubscriptionId` or `newSubscription` — the full
provision-subscription body with its own `customerId` and `planId` (`stigg-subscriptions`). Only
`customerId` is required; omit `subscriptions` for an empty contract. **The create is atomic**, so a failure
leaves nothing behind.

**Always send your own `contractId`** — the idempotency key. The same one again returns the existing
contract instead of a second one; derive it from the deal (the PO or order number). Without it a retry
silently duplicates, and contracts cannot be deleted.

## Attach and Detach Subscriptions

```bash
C=https://api.stigg.io/api/v1/contracts/contract-acme-2026

# Attach existing ones (shorthand), or provision into the contract with create's own entries
curl -X POST "$C/subscriptions" -H "X-API-KEY: <KEY>" -H "content-type: application/json" \
  -d '{ "subscriptionIds": ["subscription-acme-analytics"] }'
curl -X POST "$C/subscriptions" -H "X-API-KEY: <KEY>" -H "content-type: application/json" \
  -d '{ "subscriptions": [ { "newSubscription": { "customerId": "customer-acme", "planId": "plan-ent" } } ] }'

# Detach — unlink only; the subscription stays active
curl -X DELETE "$C/subscriptions/subscription-acme-analytics" -H "X-API-KEY: <KEY>"
```

Idempotent: re-attaching, or detaching what isn't attached, changes nothing. A subscription on a *different*
contract is rejected, not moved.

> **Don't use `PATCH /contracts/:id` to add a subscription.** Its `subscriptionIds` **replaces** the whole
> set, unlinking any ID you leave out. Use it only to declare complete membership.

## Amend, Archive, Read

`PATCH /contracts/:id` (name, PO number, activation dates), `GET /contracts/:id`,
`GET /customers/:customerId/contracts`, `GET /contracts?state=ACTIVE`, `POST /contracts/:id/archive`.

> **Archive is not detach.** Archiving cancels the contract **and every subscription on it** — the customer
> loses access. To take a subscription off a contract while keeping it running, detach it.

## What Provisioning Alone Does NOT Give You

No billing contract: `billingId` and `nextInvoice` stay `null`, no invoices are generated, and
cancel-billing rejects the contract — working as intended, not a misconfiguration. Adding billing later is
one-way; see `references/billing-and-invoices.md`.

## Contract State

Stored state is `DRAFT` until canceled; the state you read back is **derived** — `ACTIVE` once any
subscription on it is active. So a provisioning-only contract flips to `ACTIVE` on its own, with no publish
step. Field table, state machine, error codes, `contract.*` events: `references/contract-model.md`.

---

# Set Up Billing

Negotiated terms, priced line items, invoices. Needs a billing contract — created with `setupBilling:
true`, or enabled later on a provisioning-only one (one-way; billing is never removed).

**Full walkthrough: `references/billing-and-invoices.md`.** Five things first:

1. **Confirm the whole reading before the first write.** An order form is not a spec. Read back every line
   item with its pricing model and rows, the terms and exceptions, the legal entity, the customer — and get
   agreement **before any write call**. The pre-publish review is too late to be the safety net: the lines
   already exist, and billing and entitlements are what a customer notices first. Surface contradictions
   rather than resolving them. Order forms lose lines predictably — usage-based lines show no amount,
   `Included` lines belong at price 0, a per-period table is one contract, "purchase order reference" is
   often `N/A` and is *not* the order number. The reference covers each.
2. **Line items are priced by existing pricing models — never a table you build.** `GET
   /contracts/pricing-models` lists them with `isPayAsYouGo` and `editableInputs`; supply only the inputs a
   model exposes, named as `rows` accepts them. Charge type is **in advance** (`IN_ADVANCE`) or **in
   arrears** (`IN_ARREARS`) — the engine's `DURING_USE`/`AFTER_USE` are rejected. Pay-as-you-go applies
   only to metered items, takes **no quantity** (usage supplies it), and is always in arrears.
   Resolve a model by **`isPayAsYouGo` + shape, never by name** — five of the ten selectable models share a
   name with their opposite-category twin, and both twins accept the same rows and return 200.
   `references/pricing-models.md` maps deal shapes to models, with the nearest wrong neighbour for each.
3. **A legal entity needs a payment account.** `GET /contracts/legal-entities` lists each entity's
   accounts; an empty `paymentAccounts` is the common half-configured state and no API creates one — raise
   it before building, or the contract looks complete then refuses to publish. Set both, plus period,
   currency and payment terms, with `PATCH .../billing/settings` and `.../billing/terms`; both are
   contract-wide, so apply them before any per-item exception.
4. **Two checks before publishing, both real on a draft.** `GET .../billing/summary` against the order
   form's figures, and `GET .../billing/invoice-schedule` (`?count=N`) against its billing schedule. The
   totals can be exactly right while the invoices are wrong — items that agree on money but differ on
   *when* they bill split into extra invoices, and only the schedule shows it. Publishing refuses without
   a period, payment terms and a legal entity.
5. **Billing edits apply to a draft.** On a live contract: amend it (stays live) or reopen it (back to
   draft). Rejections come back as `BillingContractOperationRejected` with the remedy in the message — read
   it rather than retrying. `BillingContractEditBlocked` is terminal: credit note or new contract.

Invoices and credit notes: same reference.

## When NOT to Use This Skill

- Self-serve catalog modeling — plans, addons, charges, coupons → `stigg-pricing-modeling`.
- Subscription lifecycle with no contract attached → `stigg-subscriptions`.
- Credit currencies, grants, ledger → `stigg-credits`.
- Per-entity budgets and usage limits → `stigg-governance`.

## Common Mistakes

Each mistake — the fix:

- **Omitting `setupBilling` for provisioning only** — it defaults to `true`. Send `false` explicitly.
- **Attaching a self-serve subscription** — free/paid self-serve plans can't join. Model the deal as a
  custom plan.
- **`PATCH subscriptionIds` to add one** — it replaces the whole set, unlinking what you omitted. Use
  `POST /contracts/:id/subscriptions`.
- **Archiving to remove a subscription** — it cancels the contract *and* every subscription on it. Detach
  instead.
- **Treating the contract as what grants entitlements** — the *subscriptions* do. An empty contract grants
  nothing and holds no price or quantity fields.
- **Retrying a 409 on product duplication** — a modeling conflict, not transient. Detach that product's
  existing subscription first.
- **Inventing a pricing table** — pricing comes from an existing model, chosen by id. Fill only the inputs
  it exposes.
- **Publishing without reading the summary** — afterwards those figures are what the customer is invoiced.
- **Retrying `BillingContractEditBlocked`** — terminal. The change becomes a credit note or a new contract.
- **Expecting a URL for an invoice PDF** — it returns inline base64; no durable link. `GET .../pdf` waits
  for the render, or poll the generate/status pair.
