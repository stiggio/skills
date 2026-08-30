# Contract model

Deep dive behind `stigg-contracts`. The `SKILL.md` covers the 80% case; this is the field-level detail.

## Contract fields

| Field | Type | Notes |
|---|---|---|
| `contractId` | string | The contract's ref ID — **the key** you read, amend, and archive by |
| `refId` | string \| null | The same ref ID, on Stigg-managed contracts |
| `id` | string \| null | Internal UUID; what a subscription's contract link points at |
| `externalId` | string | External handle for the contract |
| `name` | string \| null | Free-text contract name |
| `poNumber` | string \| null | Purchase-order number |
| `state` | enum | The contract's state — **derived on read**, see below |
| `billingState` | enum \| null | The billing contract's own state. **Always `null` without a billing contract** |
| `activationStartDate` | ISO 8601 \| null | Start of the agreed activation window |
| `activationEndDate` | ISO 8601 \| null | End of the activation window |
| `createdAt` | ISO 8601 \| null | When the contract was created |
| `billingId` | string \| null | The billing contract's ID. **`null` on a provision-access contract** |
| `nextInvoice` | object \| null | Upcoming-invoice preview; `null` without a billing contract |
| `latestInvoice` | object \| null | Most recent issued invoice; `null` without a billing contract |
| `subscriptions` | array | `{ subscriptionId, planDisplayName, productDisplayName }` per attached subscription |

`state` and `billingState` are **different questions**: the first is the Stigg contract, the second is
only about billing. On a provision-access contract the second is always `null` — don't read that as an
error.

## Which fields can change

**Amendable** (`PATCH /contracts/:id`): `name`, `poNumber`, `activationStartDate`, `activationEndDate`.

**Membership**: `subscriptionIds` on the same PATCH **replaces** the whole set. Prefer the additive
`POST /contracts/:id/subscriptions` and `DELETE /contracts/:id/subscriptions/:subscriptionId`.

**One-way**: `setupBilling: true` on a PATCH adds a billing contract to a contract that has none. It is
enable-only — `false` never removes billing, and there is no path back to provision-access-only.

**Immutable**: the customer. A contract belongs to the customer it was created for; move the deal by
creating a new contract.

## State machine

Four states: `DRAFT`, `ACTIVE`, `CANCELED`, `END_BILLING`.

What's stored is deliberately minimal — a contract is written as `DRAFT` and only ever moved to
`CANCELED` when it's archived. **The state you read back is derived**: a stored `DRAFT` is reported as
`ACTIVE` when at least one attached subscription is active, or the billing contract is active. Any other
stored state is returned as-is.

Consequences worth internalizing:

- A provision-access contract becomes `ACTIVE` **by itself** once its subscriptions are live. There is no
  publish or activation call, and `activationStartDate` does not drive it — that field records the agreed
  window, it doesn't gate anything.
- An empty contract, or one whose subscriptions have all ended, reads as `DRAFT`.
- `ACTIVE` says nothing about billing. Read `billingState` for that.

## Membership rules

Every rule below is enforced server-side, on create and on attach alike:

1. **Custom-priced only** — the subscription's pricing type must be `CUSTOM`: a CUSTOM plan, or a PAID
   plan provisioned with `paymentCollectionMethod: "NONE"`. FREE-plan subscriptions are rejected.
2. **One subscription per product** — rejected with `DuplicatedEntityNotAllowed` (409).
3. **One currency** — every subscription on the contract must share a billing currency.
4. **Same customer** — a subscription belonging to another customer is rejected.
5. **Not already on another contract** — attach rejects it rather than moving it. Detach it from the
   first contract, then attach.

Rules 1–4 are checked before anything is written, so a rejected create leaves nothing behind.

## Archive vs detach

| | Effect on the contract | Effect on its subscriptions |
|---|---|---|
| `POST /contracts/:id/archive` | Canceled | **All canceled** — the customer loses access |
| `DELETE /contracts/:id/subscriptions/:subscriptionId` | Unchanged | Unlinked only; **stays active** |

Archive is the destructive one. If the intent is "this subscription doesn't belong on this contract",
that's detach.
