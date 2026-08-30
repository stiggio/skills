---
name: stigg-contracts
description: Use for contract-based (sales-led / enterprise) agreements in Stigg — the contract that is both an entitlement and an invoice. Covers both ways to start one — provision access (entitlement provisioning, attaching custom subscriptions) and set up billing (negotiated terms, custom pricing, invoicing) — plus attaching or detaching subscriptions, amending, archiving, refreshing contract usage, and reading a customer's contracts and invoices. Triggers include "contract", "enterprise deal", "negotiated terms", "order form", "PO number", "provision access", "entitlement provisioning", "billing contract", "set up billing", "attach subscription to contract", "custom plan subscription", "next invoice", "customer invoices", "amendment", "renewal". Skip for catalog modeling (stigg-pricing-modeling), subscriptions with no contract (stigg-subscriptions), credits (stigg-credits), and per-entity budgets (stigg-governance).
---

# Stigg Contracts — Entitlements and Invoicing

**"A contract is an entitlement and an invoice. Stigg treats it as both."** A contract is the source of
truth for a sales-led deal: what the customer can do, and what you charge for it.

A contract **groups subscriptions** for one customer. Those subscriptions are what actually grant
entitlements — the contract is the agreement they belong to, not a second entitlement mechanism. On top
of that grouping it can optionally carry a **billing contract**, which is what produces invoices. The two
halves are independent: a contract with no billing contract still provisions access, and that is a
first-class way to use contracts, not a half-configured state.

> **The contract itself holds no commercial terms.** It has no price, quantity, or commitment fields —
> only a name, PO number, activation window, and its subscription links. **Negotiated price and committed
> volumes live on the custom subscription** (`priceOverrides`, `charges`, `unitQuantity`, `minimumSpend` —
> see `stigg-subscriptions`). Line items with prices exist only on a *billing* contract (Path B). Don't
> look for a price on a provision-access contract; there isn't one.

## Before You Start

Per the umbrella `stigg` skill: **search first.** Contract shapes and field names evolve — open the
relevant docs page (Mintlify Stigg docs MCP) before authoring integration code.

> **No SDK surface.** Contracts are **not** in `@stigg/node-server-sdk` or `@stigg/js-client-sdk` — unlike
> every other pillar, there is no `stigg.createContract(...)`. Use the REST API (canonical here) or
> GraphQL. Samples below are REST; the base URL is `https://api.stigg.io/api/v1` and auth is
> `X-API-KEY: <YOUR_API_KEY>`, same as everywhere else (see `stigg-api`).

> **Availability.** Contract management is not enabled for every account. If the contract endpoints 404 or
> the Contracts section is missing from the dashboard, it isn't turned on — ask Stigg rather than
> working around it.

## The Two Ways to Start a Contract

Provisioning and billing are selected **independently** — one, the other, or both. This choice is the
`setupBilling` flag, and it is the spine of everything below.

| | What it does | Flag |
|---|---|---|
| **Provision access** (*entitlement provisioning*) | "Grant entitlement access & limits, e.g. a trial." Attaches custom subscriptions; entitlements activate immediately. No billing contract. | `setupBilling: false` |
| **Set up billing** (*billing contract*) | "Negotiated terms, custom pricing & invoicing." Creates the billing contract that produces invoices. | `setupBilling: true` — **the default** |

> **Vocabulary — recognize all three.** The product UI says **"Provision access"** / **"Set up billing"**;
> the docs' setup-flow page says **"Entitlement provisioning"** / **"Billing contract"** / **"Both"**; the
> API says `setupBilling`. Same two things. Prefer the UI wording when talking to a user.

## The Operation Map

| Op | Purpose | Reference |
|---|---|---|
| **Create** | Create a contract, with or without a billing contract | inline below |
| **Attach subscriptions** | Add subscriptions to a contract (additive) | inline below |
| **Detach subscription** | Remove one subscription's link, leaving it active | inline below |
| **Amend** | Edit name, PO number, activation dates | inline below |
| **Archive** | Cancel the contract **and every subscription on it** | inline below |
| **Read** | Get one contract, list the environment's, list a customer's | inline below |
| **Enable billing later** | Turn a provision-access contract into a billing one (one-way) | `references/billing-and-invoices.md` |
| **Refresh contract usage** | Push not-yet-reported metered usage for the current period | `references/billing-and-invoices.md` |
| **Invoices** | A customer's invoices, and the next-invoice preview | `references/billing-and-invoices.md` |

---

# Path A — Provision Access

Group custom subscriptions under a contract and grant entitlements. **No invoices are produced on this
path.**

## Prerequisites

**Only custom plans qualify.** Per the docs: *"Contract provisioning only supports custom plans and their
add-ons. Self-service plans are not currently supported."* Concretely, every subscription on a contract
must be **custom-priced**, which means either:

- its plan's pricing type is **CUSTOM**, or
- it's a **PAID** plan provisioned with `paymentCollectionMethod: "NONE"` (the older
  `isCustomPriceSubscription: true` sets the same thing).

A **FREE**-plan subscription can never join a contract. Attaching one fails validation, and the fix is to
model the deal as a custom plan (`stigg-pricing-modeling`), not to retry.

**Provision the customer's account first.** Per the docs: *"Create the contract only after the customer's
account, organization, or workspace has already been provisioned in the application integrated with
Stigg."*

**Three more rules, enforced server-side:**

- **One subscription per product** — a contract may not hold two subscriptions for the same product.
- **One currency** — every subscription on the contract must bill in the same currency.
- **Same customer** — every subscription must belong to the contract's customer.

## Create the Contract

`setupBilling: false` is the whole difference. **It defaults to `true`, so it must be sent explicitly** —
omitting it creates a billing contract you didn't ask for.

```bash
curl -X POST "https://api.stigg.io/api/v1/contracts" \
  -H "X-API-KEY: <YOUR_API_KEY>" \
  -H "content-type: application/json" \
  -d '{
    "customerId": "customer-acme",
    "name": "Acme 2026",
    "poNumber": "PO-4821",
    "setupBilling": false,
    "subscriptions": [
      { "existingSubscriptionId": "subscription-acme-platform" }
    ]
  }'
```

Each entry in `subscriptions` is **exactly one of**:

- `existingSubscriptionId` — the ref ID of a custom subscription that already exists, or
- `newSubscription` — a full **provision-subscription body**, which Stigg provisions as part of the same
  call. It carries its own `customerId` and `planId` (see `stigg-subscriptions`); pass the contract's
  customer, not a different one.

`subscriptions` is optional — omit it to create an empty contract and attach subscriptions later. Only
`customerId` is required.

**The create is atomic.** Every new subscription and the contract itself are written in one transaction;
any validation or creation failure rolls back the whole thing, so a failed call never leaves a
half-built contract.

The response is the contract: `contractId` (the ref ID you address it by), `state`, `subscriptions[]`,
and — on this path — `billingId: null` and `nextInvoice: null`.

## Attach and Detach Subscriptions

```bash
# Attach — additive and idempotent
curl -X POST "https://api.stigg.io/api/v1/contracts/contract-acme-2026/subscriptions" \
  -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{ "subscriptionIds": ["subscription-acme-analytics"] }'

# Detach one — unlink only; the subscription stays active
curl -X DELETE "https://api.stigg.io/api/v1/contracts/contract-acme-2026/subscriptions/subscription-acme-analytics" \
  -H "X-API-KEY: <YOUR_API_KEY>"
```

Both are idempotent: re-attaching something already attached, or detaching something that isn't, succeeds
and changes nothing. Attaching a subscription that belongs to a *different* contract is rejected rather
than silently moved.

> **Don't use `PATCH /contracts/:id` to add a subscription.** Its `subscriptionIds` **replaces** the whole
> set — any ID you leave out gets unlinked. Use it only when you genuinely intend to declare the complete
> membership.

## Amend, Archive, Read

```bash
# Amend metadata — name, PO number, activation dates
curl -X PATCH "https://api.stigg.io/api/v1/contracts/contract-acme-2026" \
  -H "X-API-KEY: <YOUR_API_KEY>" -H "content-type: application/json" \
  -d '{ "poNumber": "PO-4899" }'

curl "https://api.stigg.io/api/v1/contracts/contract-acme-2026" -H "X-API-KEY: <YOUR_API_KEY>"
curl "https://api.stigg.io/api/v1/customers/customer-acme/contracts" -H "X-API-KEY: <YOUR_API_KEY>"
curl "https://api.stigg.io/api/v1/contracts?state=ACTIVE&limit=25" -H "X-API-KEY: <YOUR_API_KEY>"
```

> **Archive is not detach.** `POST /contracts/:id/archive` cancels the contract **and cancels every
> subscription linked to it** — the customer loses access. To remove a subscription from a contract while
> keeping it running, detach it.

## What This Path Does NOT Give You

No billing contract, therefore: `billingId` and `nextInvoice` are always `null`, no invoices are generated,
and the cancel-billing operation rejects the contract outright ("it is provision-access-only"). None of
this is a misconfiguration — it's the path working as intended. To add billing later, see
`references/billing-and-invoices.md`; enabling it is **one-way**.

## Errors

Errors come back as `{ "code", "message" }`:

| Code | Status | Usually means |
|---|---|---|
| `InvalidArgumentError` | 400 | Not custom-priced, wrong customer, currency mismatch, or already on another contract |
| `ContractNotFound` | 404 | Unknown contract ref ID |
| `PlanNotFound` | 404 | A `newSubscription` entry names a plan that doesn't exist |
| `DuplicatedEntityNotAllowed` | 409 | A second subscription for a product the contract already covers |

## Events

`contract.created`, `contract.updated`, `contract.canceled`, `contract.subscription_added`,
`contract.subscription_removed` — emitted after the write commits, so they never describe a rolled-back
change. Delivery and verification are `stigg-webhooks`' job.

## Contract State

The stored state is `DRAFT` until the contract is canceled; the state you **read back** is derived — a
contract shows as `ACTIVE` once at least one subscription on it is active. So a provision-access contract
flips to `ACTIVE` on its own as soon as its subscriptions are live, with no publish step. Full field table
and state machine: `references/contract-model.md`.

---

# Path B — Set Up Billing

<!-- TODO: negotiated terms, custom pricing, invoicing, the order form, and enabling billing on a
     contract that started as provision-access-only (one-way). Deep dive:
     references/billing-and-invoices.md -->

---

## When NOT to Use This Skill

- Self-serve catalog modeling — plans, addons, charges, coupons → `stigg-pricing-modeling`.
- Subscription lifecycle with no contract attached → `stigg-subscriptions`.
- Credit currencies, grants, ledger → `stigg-credits`.
- Per-entity budgets and usage limits → `stigg-governance`.

## Common Mistakes

Each mistake — the fix:

- **Omitting `setupBilling` when you only want to provision access** — it defaults to `true`, so a billing
  contract gets created. Send `false` explicitly.
- **Attaching a self-serve subscription** — free/paid self-serve plans can't join a contract. Model the
  deal as a custom plan.
- **Using `PATCH subscriptionIds` to add one subscription** — it replaces the whole set and unlinks
  everything you omitted. Use `POST /contracts/:id/subscriptions`.
- **Archiving to remove a subscription** — archive cancels the contract *and* every subscription on it.
  Detach instead.
- **Expecting invoices or a next-invoice preview on a provision-access contract** — there is no billing
  contract, so there is nothing to invoice.
- **Treating the contract as the thing that grants entitlements** — the *subscriptions* grant them; the
  contract groups them. An empty contract grants nothing.
- **Looking for negotiated price or committed volumes on the contract** — it has no such fields. They're
  on the custom subscription; only a billing contract carries priced line items.
- **Retrying a 409 on product duplication** — it's a modeling conflict, not a transient error. Detach the
  existing subscription for that product first.
