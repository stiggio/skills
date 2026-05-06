# Stigg Skills

Agent skills for [Stigg](https://stigg.io) — pricing, packaging, entitlements, and credits-based monetization for AI-built apps.

These skills give an AI coding assistant opinionated, search-first, scoped guidance for every common Stigg integration task: connect the MCP server, pick the right API surface, model your pricing, gate features, report usage, monetize with credits, and drop in production-grade widgets.

## Installation

> Distribution via the standard agent-skills format. Once published, install with:

```bash
# Install a specific skill
npx skills add stigg/skills --skill stigg
npx skills add stigg/skills --skill stigg-mcp
npx skills add stigg/skills --skill stigg-api

# Install all skills
npx skills add stigg/skills --all
```

Until then, point your agent at this directory directly.

## Available skills

| Skill | Description |
|-------|-------------|
| `stigg` | **Start here.** Umbrella entry point with the search-first rule, decision tree (MCP vs CLI vs SDK vs raw API), and the Stigg core-concepts glossary. |
| `stigg-mcp` | Connect the Stigg MCP server (`https://mcp.stigg.io`) to Claude Code, Claude Desktop, ChatGPT, Cursor, VS Code, Windsurf, or Codex; pick the right key; smoke-test. |
| `stigg-api` | Authentication, REST vs GraphQL selection, SDK selection (Node + React canonical; raw HTTP for everything else), rate limits, pagination, idempotency. |
| `stigg-pricing-modeling` | Model the catalog: features, plans, addons, products, charges, coupons, custom credit currencies, price localization. MCP-first rule of thumb. |
| `stigg-entitlements` | Runtime gating + the raw-events vs calculated-usage rule + cache & fallback strategy for production. Owns promotional entitlements. |
| `stigg-subscriptions` | Lifecycle ops: provision, preview, update, cancel, trials, multi-active subscriptions, plan-version migration. |
| `stigg-credits` | Credit currencies, grants, ledger, consumption logic, custom formulas, auto-recharge, seat-based pools, billing integration, revenue recognition. |
| `stigg-widgets` | Drop-in UI: paywall, customer portal, checkout, credit widgets. Uses the live Storybook index at `widgets.stigg.io/index.json`. |
| `stigg-webhooks` | Receive webhook events from Stigg — signature verification (`Stigg-Webhooks-Secret`), payload envelope, retry semantics, idempotency via `messageId`, handler skeleton. |
| `stigg-pricing-expert` | Advisory — picks the right monetization model and hands off to implementation skills. |
| `stigg-import-from-external-system` | One-off migration from Stripe / Recurly / Chargebee / Zuora / custom in-house systems into Stigg. Strangler Fig pattern. |
| `stigg-recipes` | Composed end-to-end workflows — freemium, checkout, hybrid pricing, AI-credits monetization, trial with addons, payment links. |

## Authoritative sources

These skills do not bake in copy that goes stale. They link to live sources:

- [docs.stigg.io](https://docs.stigg.io) — product docs, every skill links to specific pages
- [docs.stigg.io/llms.txt](https://docs.stigg.io/llms.txt) — LLM-friendly URL index
- [Mintlify docs MCP](https://docs.stigg.io) — every skill instructs the agent to search live docs *before* generating Stigg code
- [widgets.stigg.io](https://widgets.stigg.io) — Storybook for Stigg widgets

## Links

- [Stigg](https://stigg.io)
- [Stigg docs](https://docs.stigg.io)
- [Agent Skills format](https://agentskills.io)
