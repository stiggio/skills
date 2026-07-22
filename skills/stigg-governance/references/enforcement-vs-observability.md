# Enforcement vs Observability — Hot Path vs Cold Path

Governance exposes two reads that look similar but serve opposite purposes. Keeping them apart is the single most important operational distinction in this skill: one is on the request's **hot path** and gates access; the other is a **cold path** that lags and only observes.

| | **Hot path — enforcement** | **Cold path — observability** |
| --- | --- | --- |
| Operation | The entitlements **check** (`client.v1Beta.customers.entitlements.check`) | The **governance tree query** (`retrieveGovernance`) |
| Purpose | Gate the request — allow / deny | Report usage-vs-budget |
| Consistency | Synchronous, enforces in the access path | Read model that **may lag by minutes** |
| Use for | Per-request gating | Dashboards, burn-down alerts, reconciliation, admin UIs |
| Never | — | **Never** gate access from it |

## The two reads

- **Access (hot path — gates requests):** the standard entitlements **check**. It runs synchronously on the request path, enriched by the governance chain when governance is enabled, and returns `isGranted`. This is the *only* surface that gates access. Full semantics: `enforcement-and-usage.md`.
- **Balances (cold path — observability):** the **governance tree query** (`retrieveGovernance`). It returns the entity tree with usage-vs-budget, but its usage figures come from a read model that may lag.

## The lag contract

The tree's usage figures come from a **read model that may lag by minutes**. Consequences:

- **Never gate access from the tree.** Gating belongs to the entitlements check, which enforces in the access path.
- Do use the tree for: budget dashboards, burn-down alerts ("proj-atlas at 80%"), reconciliation, admin UIs.
- A freshly reported event not showing in the tree for a few minutes is expected behavior, not data loss.

## Why it matters

Mixing the two is the classic governance mistake: polling the cold-path tree per-request to "pre-check" a budget both adds latency and gives a stale answer, while an over-budget entity can slip through until the read model catches up. Gate on the hot-path check; watch the cold-path tree. Details, response shapes, and the ancestor rule: `enforcement-and-usage.md`.
