---
name: stigg
description: Use when integrating, extending, or operating Stigg (pricing, plans, features, entitlements, subscriptions, credits, usage reporting, paywalls, customer portal). Triggers include "set up Stigg", "add Stigg to this app", "gate a feature with Stigg", "report usage to Stigg", "create a plan / addon / feature", "monetize with credits", "paywall", "customer portal", or any docs.stigg.io URL. Load before any other stigg-* skill.
---

# Stigg — Umbrella Skill

[Stigg](https://stigg.io) is a pricing, packaging, and entitlements platform. This skill is the **first stop** for any Stigg work. It enforces the search-first rule and routes you to the right sub-skill.

## Step 0 — You Need a Stigg Account

Every other step assumes you already have:

1. **An account at [app.stigg.io](https://app.stigg.io)** (sign up if you don't).
2. **At least one environment** in that account (dev / staging / prod — Stigg provisions a default sandbox automatically; production environments are on the Scale plan).
3. **An API key** from **Integrations → API keys** in the environment you intend to operate on.

Without these three, every "set up Stigg" step in the sub-skills will fail with `401 Unauthenticated`. Surface this to the user at the start of the integration if they haven't done it.

## The Search-First Rule (non-negotiable)

**Before generating any Stigg integration code, search the live docs.** Stigg ships rapidly; baked-in snippets go stale. The rule applies even when you "remember how it works."

```
For every Stigg task → search docs → then (if connected) call the Stigg MCP → only then write code.
```

How to search, ranked:

1. **Stigg MCP server** (`mcp.stigg.io`) — if the user has it connected, the agent can call `search_docs` *and* execute against the real environment. See `stigg-mcp`.
2. **Mintlify Stigg docs MCP** — if the agent has the official docs MCP available (`search_stigg`, `query_docs_filesystem_stigg`), use it.
3. **`https://docs.stigg.io/llms.txt`** — LLM-friendly URL index. Fetch it, find the relevant page, fetch that page.
4. **Direct fetch of `docs.stigg.io/<path>`** — last resort.

Full rationale and examples: `references/search-first.md`.

**Red flag — STOP and search:** you're about to write `import Stigg from '@stigg/...'`, a `curl https://api.stigg.io/...` call, or any GraphQL mutation, and you have **not** opened a docs page in this turn.

## Pick the Right Surface

Stigg exposes several integration surfaces. Pick deliberately.

| Goal | Use |
|---|---|
| Modeling pricing during development, exploration, one-off ops | **Stigg MCP server** (`stigg-mcp`) |
| Production hot path: gating, usage reporting, customer-facing pages | **Backend or frontend SDK** (`stigg-api` → SDK matrix) |
| Direct REST calls (no SDK in your language, or thin server-side proxy) | **REST API** at `https://api.stigg.io/api/v1` (`stigg-api`) |
| GraphQL (legacy or specific advanced queries) | **GraphQL API** at `https://api.stigg.io/graphql` (`stigg-api`) |
| Scripts, CI/CD, terminal admin | **Stigg CLI** |
| Drop-in UI (paywall, customer portal, checkout) | **Widgets** — see `widgets.stigg.io` Storybook |

Decision flowchart with edge cases: `references/decision-tree.md`.

> **CLI vs MCP server:** both are programmatic. The CLI is deterministic — exact commands, scriptable. The MCP server is non-deterministic — natural-language driven by an agent. Use the CLI for scripts/CI; use the MCP for AI-assisted dev. They are complementary, not alternatives.

## Routing — Which Sub-Skill You Need

| Task | Skill |
|---|---|
| First-time MCP setup, connecting Claude Code/Cursor/etc. to Stigg | `stigg-mcp` |
| Picking SDK vs REST vs GraphQL, API key types, rate limits, auth | `stigg-api` |
| Modeling features / plans / addons / products / credits in the catalog | `stigg-pricing-modeling` |
| Gating features, reporting usage, fallback strategy | `stigg-entitlements` |
| Provision / update / cancel / migrate subscriptions, trials | `stigg-subscriptions` |
| Credit grants, consumption, auto-recharge, seat pools | `stigg-credits` |
| Paywall, customer portal, checkout, credit widgets | `stigg-widgets` |
| Receiving webhook events from Stigg — signature verification, payload, retries, handler skeleton | `stigg-webhooks` |
| Choosing the right pricing / monetization model | `stigg-pricing-expert` |
| Multi-step recipes (freemium, hybrid, AI-credits, trials) | `stigg-recipes` |

If a sub-skill is not yet loaded, route the user to it by name — do not improvise content from outside this repo.

## Stigg in 60 Seconds

Minimum vocabulary before building. Full glossary: `references/core-concepts.md`.

- **Customer** holds **Subscriptions** to **Plans** under a **Product**. Subscriptions can include **Add-ons**.
- **Features** (boolean / configuration / metered) define what's gateable. **Entitlements** = feature + configured value (e.g. "10k API calls / month" on Pro).
- **Promotional entitlements** are per-customer overrides outside the subscription. **Credit pools / grants / ledger** power credit-based pricing — one pool per credit currency. See `stigg-credits`.
- **Environments** (sandbox / staging / production) are isolated workspaces; each has its own API keys.

## When NOT to Load This Skill

- The task is unrelated to Stigg (pricing/billing/entitlements).

## Common Mistakes

| Mistake | Fix |
|---|---|
| Generating REST/SDK code from memory without searching docs first | Search first — the API surface evolves. See `references/search-first.md`. |
| Confusing CLI and MCP server (they are different tools) | CLI = deterministic shell; MCP = agent-driven. See the table above. |
| Mixing `events.report` and `usage.report` | Different scaling models, not different audiences. Covered in `stigg-entitlements`. |
| Calling against production while developing | Always start in **sandbox**. API keys are environment-bound. |
| Embedding the full-access (server) key in client-side code | Never. Use the publishable (client) key for frontend SDKs. See `stigg-api`. |
