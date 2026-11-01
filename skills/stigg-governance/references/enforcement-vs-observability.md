# Enforcement vs Observability — Hot Path vs Cold Path

Governance exposes two reads that look similar but serve opposite purposes. Keeping them apart is the single most important operational distinction in this skill: one is on the request's **hot path** and gates access; the other is a **cold path** that lags and only observes.

| | **Hot path — enforcement** | **Cold path — observability** |
| --- | --- | --- |
| Operation | The entitlements **check** (`client.v1.customers.checkEntitlement`) | The **governance tree query** (`client.v1Beta.customers.governance.list`) |
| Purpose | Gate the request — allow / deny | Report usage-vs-limit |
| Consistency | Synchronous, enforces in the access path | Usage figures that **may lag by a few seconds** |
| Use for | Per-request gating | Dashboards, burn-down alerts, reconciliation, admin UIs |
| Never | — | **Never** gate access from it |

## The two reads

- **Access (hot path — gates requests):** the standard entitlements **check**. It runs synchronously on the request path, checks the governance limits when the request has `dimensions`, and returns `isGranted`. This is the *only* surface that gates access. Full semantics: `enforcement-and-usage.md`.
- **Usage vs limit (cold path — observability):** the **governance tree query** (`client.v1Beta.customers.governance.list`, still on v1-beta). It returns the entity tree with usage-vs-limit, but its usage figures may lag by a few seconds.

## The lag contract

The tree's usage figures **may lag by a few seconds** (more for raw events, which Stigg aggregates first). Consequences:

- **Never gate access from the tree.** Gating belongs to the entitlements check, which enforces in the access path.
- Do use the tree for: limit dashboards, burn-down alerts ("proj-atlas at 80%"), reconciliation, admin UIs.
- A freshly reported event not showing in the tree for a few seconds is expected behavior, not data loss.

## Why it matters

Mixing the two is the classic governance mistake: polling the cold-path tree per-request to "pre-check" a limit both adds latency and gives a stale answer, while an over-limit entity can slip through until the usage figures catch up. Gate on the hot-path check; watch the cold-path tree. Details, response shapes, and the ancestor rule: `enforcement-and-usage.md`.
