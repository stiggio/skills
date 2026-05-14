---
name: stigg-api
description: Use when picking the right Stigg integration surface (REST vs GraphQL vs SDK) and authenticating against it. Triggers include "init the Stigg SDK", "Stigg API key", "STIGG_SERVER_API_KEY", "X-API-KEY", "401 Unauthenticated from Stigg", "403 Forbidden from Stigg", "scoped API key", "publishable key", "rotate Stigg key", "Stigg rate limits", "Stigg pagination", "Stigg idempotency". Skip for catalog / entitlement / subscription modeling — those have their own skills.
---

# Stigg API — Auth, Transport, SDK Selection

This skill answers two questions: **which Stigg surface to call**, and **how to authenticate against it correctly**. Catalog / entitlement / subscription operations belong to their own skills (see routing in the umbrella `stigg` skill).

## Before You Start

Per the umbrella `stigg` skill: **search first**. Stigg's API is in active development (REST API in Public Beta at the time of writing). Use the Mintlify Stigg docs MCP / `llms.txt` to confirm endpoint shapes and SDK init signatures before generating code.

## API Keys — The Three Types You Need to Know

Every environment is provisioned with two **default** keys; Scale-plan customers can also create **scoped** keys.

| Type | Prefix | Where it lives | Default scope | Customizable scope? |
|---|---|---|---|---|
| **Full access (server)** | `server-` | Backend only | All current and **future** permissions | No (immutable) |
| **Publishable (client)** | `client-` | Browsers, mobile | Read-only, narrow | No (immutable) |
| **Scoped (server)** | `server-` | Backend only | What you choose | **Yes**, per resource |

**Where to find them:** Stigg app → **Integrations → API keys** (each environment has its own set).

### Full access key

Default per-environment. Unrestricted within that environment, and **automatically inherits new permissions** as Stigg adds capabilities. Only one full-access key per environment exists; users cannot create more. Use it as the default backend key while bootstrapping; switch to a narrower scoped key as your integration matures.

### Publishable key

Designed for frontend / mobile SDKs. Read-only and **immutable** — its scope cannot be widened. Safe to ship in client-side code by design, but turn on **client-side security hardening** for production (HMAC SHA256 signed customer tokens — configure under the publishable key's detail panel).

> **Never put a full-access or scoped server key in the browser, in a public repo, or in a frontend bundle. Use environment variables on the server.**

### Scoped keys (Scale plan)

User-created server keys with explicit, restricted permissions — the **principle of least privilege** in action. Create one per service / AI client / use case so a leak's blast radius stays small. Scoped keys do **not** inherit future platform capabilities; new resources must be granted manually.

**Custom permission scope matrix** (presets when creating a scoped key):

- **Full access** — all read/write across the environment.
- **Read only** — read across all resources; no writes.
- **Customers write** — manage customer data.
- **Subscriptions write** — create/update subscriptions.
- **Coupons write** — manage discount/coupon logic.
- **API keys read and write** — manage API keys programmatically.
- **Event Queue read** — obtain temporary SQS event-queue credentials (read-only; this resource has no write).

Selecting *write* on a resource implicitly grants *read* on it.

### Key rotation

From **Integrations → API keys**, open the key's context menu (`⋮`) → **Rotate key**. Pick a grace period (Now / 1h / 24h / 3d / 7d). The old secret stays valid until the grace ends — zero-downtime rolling deploys are the design intent. Need more time? **Change grace period** before the old one expires.

Revocation is immediate and irreversible. Switch all services to the new key first.

> **Full reference:** `references/auth.md`.

## Pick the Right Transport

| Surface | When |
|---|---|
| **Node SDK** (`@stigg/node-server-sdk`) | Production hot paths on Node / TypeScript. **Default choice for backend.** Other backend runtimes (Python, Go, Java, Ruby, .NET, Sidecar) ship SDKs too — see [docs.stigg.io](https://docs.stigg.io); patterns are similar but exact field names may vary. |
| **Frontend SDK** (`@stigg/react-sdk` for React / Next.js; `@stigg/js-client-sdk` for vanilla JS) | Rendering paywalls, customer portal, or entitlement checks in the browser. |
| **REST API** (`https://api.stigg.io/api/v1`) | No SDK in your language; thin server-side proxies; explicit control. |
| **GraphQL API** (`https://api.stigg.io/graphql`) | Existing GraphQL integration, or a query the REST API doesn't expose yet. |
| **Stigg MCP server** (`https://mcp.stigg.io`) | AI-assisted dev. See `stigg-mcp`. |
| **Stigg CLI** (Go binary — `brew install stiggio/tools/stigg`; repo: https://github.com/stiggio/stigg-cli) | Scripts, CI/CD — **opt-in only**, when the user explicitly asks for the CLI. Default to the MCP server for integration work. |

Decision flowchart with edge cases: see `stigg/references/decision-tree.md`. SDK selection (Node + React canonical, raw HTTP for non-Node runtimes): `references/sdk-selection.md`.

## REST API — Quick Reference

- **Base URL:** `https://api.stigg.io/api/v1`
- **Auth header:** `X-API-KEY: <YOUR_SERVER_KEY>`
- **Content-Type:** `application/json` for bodies.
- **Idempotency:** add `Idempotency-Key: <unique-id>` to POSTs (cached 24h). Repeated calls with the same key return the cached response.
- **Pagination:** cursor-based. Query params: `limit` (default 20, max 100), `starting_after`, `ending_before`. Response includes `data[]`, `has_more`, `next_cursor`.
- **Filtering:** resource-specific query params. Enum-valued filters take the underlying enum symbol (**uppercase**), and most list endpoints accept comma-separated values for multi-value filtering. Examples: `?email=user@example.com`, `?status=ACTIVE,IN_TRIAL`, `?pricingType=PAID`. Date fields accept range operators (`?createdAt[gte]=2026-01-01T00:00:00Z`).
- **Errors:** standard HTTP codes with a structured body.

```bash
# Authenticated GET
curl -X GET "https://api.stigg.io/api/v1/customers" \
  -H "X-API-KEY: $STIGG_SERVER_API_KEY" \
  -H "Content-Type: application/json"

# Idempotent POST
curl -X POST "https://api.stigg.io/api/v1/customers" \
  -H "X-API-KEY: $STIGG_SERVER_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: bootstrap-customer-acme-001" \
  -d '{ "id": "customer-123", "name": "Acme Corp", "email": "billing@acme.com" }'
```

### Money amounts — **full dollars, not cents**

> **Critical for anyone coming from Stripe:** Stigg expresses monetary amounts as **full dollars / units of the currency**, not minor units. `49` means $49.00, not $0.49. `3990` means $3,990, not $39.90.
>
> Stripe convention is the opposite (amounts in the smallest unit — cents for USD). Don't multiply by 100 when handing values to Stigg, and don't divide by 100 when reading them back. This applies to plan prices, charge amounts, `cost_basis` on credit grants, the `subtotal` / `total` / `discount` / `tax` fields on invoice previews — every monetary field.

### Error semantics

| Status | Meaning |
|---|---|
| 200 | Success — body contains the resource |
| 201 | Created |
| 400 | Bad request — invalid params/body |
| 401 | Unauthenticated — missing or invalid `X-API-KEY` |
| 403 | Forbidden — authenticated, but the key lacks the required scope |
| 404 | Not found |
| 409 | Conflict — e.g., subscription already exists |
| 429 | Too many requests — see rate limits |

Two REST gotchas worth knowing up front (re-confirm on the page in case they change):

- `GET /api/v1/subscriptions` does **not** include `subscriptionEntitlements` for each item — too expensive at scale. Use `GET /api/v1/subscriptions/{id}` for the detailed entitlements.
- `subscriptionEntitlements` on a single subscription is the **overrides only**, not the full set inherited from the plan. To get the full plan entitlements, call `GET /api/v1/plans/{planId}/entitlements`.

## GraphQL API — Quick Reference

- **Endpoint:** `https://api.stigg.io/graphql`
- **Auth header:** `X-API-Key: <YOUR_SERVER_KEY>`
- **When:** keep GraphQL for existing integrations or queries REST hasn't surfaced yet. **Prefer REST for new work.**

```bash
curl -X POST https://api.stigg.io/graphql \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $STIGG_SERVER_API_KEY" \
  -d '{"query":"{ customers { edges { node { customerId name } } } }"}'
```

## SDKs — Initialization Snippets

> Always pick the SDK matching your runtime. Full matrix: `references/sdk-selection.md`.

**Backend (Node) — `@stigg/node-server-sdk`:**

```ts
import Stigg from '@stigg/node-server-sdk';

const stigg = await Stigg.initialize({
  apiKey: process.env.STIGG_SERVER_API_KEY!,
});
```

**Frontend (React) — `@stigg/react-sdk`:**

```tsx
import { StiggProvider } from '@stigg/react-sdk';

<StiggProvider apiKey="<STIGG_CLIENT_API_KEY>" customerId="<LOGGED_IN_CUSTOMER_ID>">
  {children}
</StiggProvider>
```

**Vanilla JS frontend** uses `@stigg/js-client-sdk` (synchronous `Stigg.initialize(...)` + `await client.waitForInitialization()`). Other runtimes (Python, Go, Java, Ruby, .NET, Sidecar, Vue, Embed) — see `references/sdk-selection.md` for a brief and the link out to docs.stigg.io. The auto-generated REST-based TypeScript SDK (`@stigg/typescript`) is currently in beta; keep using `@stigg/node-server-sdk` until it reaches GA.

## Rate Limits

Stigg enforces per-environment, per-endpoint rate limits. Production hot paths must handle `429` with backoff. The current limits and the "Using Stigg at Scale" guidance live at [docs.stigg.io](https://docs.stigg.io) — `references/rate-limits.md` summarizes the operating principles.

## Auth Failures — Quick Diagnosis

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 Unauthenticated` on every request | Header name typo (`Authorization` instead of `X-API-KEY`) | Use `X-API-KEY` exactly. |
| `401` only on POSTs | Missing / wrong `Content-Type` causing the gateway to reject before auth | Set `Content-Type: application/json`. |
| `401` after a rotation | Old key exceeded its grace period | Switch services to the new key. |
| `403 Forbidden` | Scoped key lacks permission for the resource | Grant the missing permission, or use a wider key. |
| Frontend SDK shows other customers' data | Missing client-side hardening | Enable HMAC signed customer tokens on the publishable key. |
| Production "Stigg" calls hitting sandbox | Wrong key in env config | Keys are environment-bound — verify and replace. |

## When NOT to Use This Skill

- Building a paywall / customer portal UI — use `stigg-widgets`.
- Modeling features / plans / addons — use `stigg-pricing-modeling`.
- Deciding between event-style and measurement-style usage reporting — use `stigg-entitlements`.
- First-time MCP install — use `stigg-mcp`.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Putting a `server-` key in a frontend app | Use the `client-` publishable key. |
| Hard-coding the full-access key everywhere | Create scoped keys per service. |
| Missing `Idempotency-Key` on customer/subscription POSTs | Set one — Stigg caches the response 24h, so retries are safe. |
| Treating `subscriptionEntitlements` as the full set | It's overrides only. Use the plan endpoint for the full list. |
| Calling `GET /subscriptions` and expecting entitlements | Not returned by design. Use `GET /subscriptions/{id}` per-row. |
| Mixing REST and GraphQL in new code | Pick one (REST by default). Don't build a hybrid unless you have a reason. |
