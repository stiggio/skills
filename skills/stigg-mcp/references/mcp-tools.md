# Stigg MCP — Tool Surface (Snapshot)

A terse, by-area listing of what the Stigg MCP can do. **This is a snapshot.** The MCP surface grows continuously — always re-fetch [docs.stigg.io/api-and-sdks/mcp-server](https://docs.stigg.io/api-and-sdks/mcp-server) (or call the MCP server's `search_docs` tool) before relying on a specific operation.

## How to discover the live tool list

- **Claude Code:** type `/mcp`, expand the `stigg` server.
- **Cursor / VS Code:** the MCP panel lists tools per server.
- **Programmatically:** the MCP server returns its tool catalog on connect.

## Coverage by area

### Catalog

- **Plans** — create / update / archive / list, with pricing, features, and entitlements.
- **Features** — create / update / list across boolean, configuration, and metered types.
- **Add-ons** — create / update / list, with their entitlements.
- **Packages** — publish / archive pricing packages.

### Customers & subscriptions

- **Customers** — provision, update, retrieve.
- **Subscriptions** — provision (including scheduled / trial), preview pricing, cancel (immediate or end-of-period).
- **Promotional entitlements** — grant temporary feature access outside a customer's plan.

### Entitlements & usage

- **Check entitlements** — query whether a customer has access to a feature and at what usage level.
- **Report usage** — record metered feature usage for a customer's active subscription (calculated style).
- **Report metering events** — submit raw events for aggregation and billing.

### Docs

- **`search_docs`** (or equivalent) — search the Stigg docs from inside the agent. Prefer this over web fetches when the MCP is connected.

## What's missing today

MCP coverage is broad but not 100% of the Stigg API. When an operation isn't MCP-exposed, fall through to the SDK / REST API (skill: `stigg-api`) or the Stigg CLI (`brew install stiggio/tools/stigg` — https://github.com/stiggio/stigg-cli) for scripted ops.

A **canonical MCP-coverage matrix** lives in `stigg-pricing-modeling/references/mcp-coverage-matrix.md` — that file is updated as the MCP grows.

## Don't memorize this list

Capabilities change. The right move on every turn is:

1. Search the live docs / call `search_docs` first.
2. Use the actual tool listing surfaced by your client (`/mcp` etc.).
3. Treat this file as "what to expect" — not as the source of truth.
