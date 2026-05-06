# Decision Tree — Picking the Right Stigg Surface

Stigg exposes several surfaces. Picking the wrong one is a common time sink.

## Quick decision

```text
Is the goal …
├── Setting up Stigg, exploring, modeling pricing, one-off ops?
│     → Stigg MCP server   (skill: stigg-mcp)
│
├── A production hot path (gating, usage, customer-facing)?
│     → SDK in your runtime (skill: stigg-api)
│       • Backend (default): Node — `@stigg/node-server-sdk`
│         (Python / Go / Java / Ruby / .NET / Sidecar SDKs also exist; see docs.stigg.io)
│       • Frontend (default): React — `@stigg/react-sdk` (with Next.js SSR)
│         (Vue / vanilla JS / Embed also exist; see docs.stigg.io)
│
├── No SDK in your language, or a thin server-side proxy?
│     → REST API at https://api.stigg.io/api/v1   (skill: stigg-api)
│
├── Legacy code, or a query the REST API doesn't expose yet?
│     → GraphQL API at https://api.stigg.io/graphql   (skill: stigg-api)
│
├── Scripts, CI/CD, terminal admin?
│     → Stigg CLI
│
└── Drop-in UI (paywall, customer portal, checkout)?
      → Stigg Widgets   (Storybook: https://widgets.stigg.io/)
```

## Detail per surface

### Stigg MCP server (`mcp.stigg.io`)

- **Best for:** AI-assisted dev, exploring a Stigg environment in plain English, modeling catalog changes, one-off ops ("provision this customer on the Pro plan").
- **Avoid for:** unattended automation, deterministic scripts, production hot paths.
- **Auth:** environment-bound API key in the `X-API-KEY` header. See `stigg-mcp`.

### Backend SDKs

- **Best for:** server-side hot paths — `getEntitlement`, `reportEvent`, `provisionCustomer`, `provisionSubscription`.
- **Default backend SDK is `@stigg/node-server-sdk`.** Other runtimes (Python, Go, Java, Ruby, .NET, Sidecar) have SDKs too — see docs.stigg.io. The auto-generated REST-based SDKs (`@stigg/typescript` and equivalents) are currently in beta — wait for GA.
- **Auth:** full-access server key, or a scoped key with the right permissions.

### Frontend SDKs

- **Best for:** rendering paywalls, customer portal, entitlement checks in the browser.
- **Auth:** **publishable** (client) key only. Never put a server key here.
- **Hardening:** enable client-side security (HMAC SHA256 signed customer tokens) for production. Configure under the publishable key's detail panel.

### REST API

- **Base URL:** `https://api.stigg.io/api/v1`
- **Auth header:** `X-API-KEY`
- **Idempotency:** set `Idempotency-Key` on POSTs (cached 24h).
- **Pagination:** cursor-based with `limit`, `starting_after`, `ending_before`; response includes `has_more` and `next_cursor`.
- **Status:** Public Beta at the time of writing — re-check.

### GraphQL API

- **Endpoint:** `https://api.stigg.io/graphql`
- **Auth header:** `X-API-Key`
- **Use when:** the REST API doesn't yet expose what you need, or you're maintaining an existing GraphQL integration. For new integrations, prefer REST.

### Stigg CLI

- **Best for:** scripts, CI/CD, terminal admin, copying changes between environments, one-off catalog edits.
- **Deterministic** — you write the exact command. The MCP server is non-deterministic; the agent decides which calls to make.
- **Complementary to the MCP**, not a replacement. Use the CLI when you need to know exactly what API call will run.

### Widgets (drop-in UI)

- **Storybook:** `https://widgets.stigg.io/` — the live, authoritative reference for props, variants, and snippets. Open it before authoring widget code.
- **Skill:** `stigg-widgets` covers paywall, pricing table, customer portal, checkout, and credit widgets.

## Common confusions

| Confusion | Resolution |
|---|---|
| "CLI vs MCP — which one do I install?" | Both. They serve different jobs. CLI for scripts; MCP for AI. |
| "Should I use REST or GraphQL?" | REST for new work. GraphQL only when REST doesn't expose what you need. |
| "Which key goes in the React SDK?" | The **publishable** (client) key. Server key in the browser is a security incident. |
| "MCP doesn't support this op." | Re-check via Mintlify MCP — coverage expands. The umbrella skill's search-first rule applies here too. |
| "Backend SDK vs REST" | SDK if available for your runtime. REST when no SDK fits, or for very thin proxies. |
