# Credit Ledger — Reference

The credit ledger is the **append-only chronological record** of every state change affecting a customer's credit pool. It's the source of truth for finance, support, and analytics.

## What's logged

Every change to the pool is captured:

- **Grants** — new blocks added (recurring / prepaid / promotional / manual / overdraft auto-create).
- **Top-ups and adjustments** — manual or automatic increases / decreases.
- **Deductions** — reductions from metered usage, recorded per event.
- **Expirations** — blocks reaching their `expires_at`.
- **Revocations** — credits manually voided.

## Per-entry fields

| Field | What it captures |
|---|---|
| `timestamp` | When the event occurred. |
| `type` | `Grant` / `Consumed` / `Expired` / etc. |
| `source_id` | Identifier of the originating grant or feature event. |
| `source` | Where the change came from — e.g. "Promotional grant", "Credit purchase", "Image Upscale". |
| `amount` (delta) | Positive = added; negative = removed. |
| `starting_balance` / `ending_balance` | Pool balance before and after the event. |
| `actor` | `System` / `Admin` / `API` (and which API key, when relevant). |

## Where to view

In the Stigg app: **Customers → select a customer → Credits → Ledger tab**. Filters: date range, group-by, source. Pagination supported.

The same data is exportable to your warehouse via the **Native integrations → Data warehouse** path. See `revenue-recognition.md`.

## Reading the ledger programmatically

GraphQL: `creditLedger` query (and the `creditGrants` query for grant-level state). REST: `GET /credit-ledger` and `GET /credit-grants`. Re-fetch the docs page for current parameters.

Pagination is cursor-based on the REST API (per `stigg-api`'s pagination conventions).

## Three things the ledger is good for

1. **Finance reconciliation.** Tie period consumption × `cost_basis` back to recognized revenue. See `revenue-recognition.md`.
2. **Customer support.** "Why did my balance drop yesterday?" — the ledger has the answer with timestamps and sources.
3. **Forensics.** Track down anomalies — a customer disputing usage, an upstream bug spamming events, etc.

## Common mistakes

| Mistake | Fix |
|---|---|
| Reconstructing balance from raw usage events | Don't. Read from the ledger — it's authoritative and accounts for grants, expirations, revocations, and overdraft. |
| Treating `starting_balance` / `ending_balance` as approximate | They aren't. The ledger is exact. |
| Filtering ledger by feature without checking `source_id` | A single feature may consume across multiple grants; one usage event creates one ledger entry per grant consumed. |
| Trying to edit ledger entries to "correct" history | Append-only. Issue a correcting grant or revocation; it generates new entries. |
