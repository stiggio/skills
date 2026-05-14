# Stigg Skills

Agent skills for [Stigg](https://stigg.io) — pricing, packaging, entitlements, and credits-based monetization for AI-built apps.

## Installation

Pick the install path that matches your client.

### Claude Code plugin

The skills ship as a Claude Code plugin via Stigg's marketplace:

```bash
# Add the Stigg marketplace
/plugin marketplace add stiggio/skills

# Install the Stigg plugin (all 11 skills)
/plugin install stigg@stigg-marketplace
```

Once installed, the agent auto-discovers every skill and the umbrella `stigg` skill routes to the right pillar.

### Agent skills via npx

For Claude Desktop, Claude.ai, or any client that consumes the [Agent Skills format](https://agentskills.io) directly:

```bash
npx skills add stiggio/skills --all
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `stigg` | **Start here.** Umbrella entry point with the search-first rule, the decision tree (MCP vs CLI vs SDK vs raw API), and the Stigg core-concepts glossary |
| `stigg-mcp` | Connect the Stigg MCP server (`https://mcp.stigg.io`) to Claude Code, Claude Desktop, ChatGPT, Cursor, VS Code, Windsurf, or Codex |
| `stigg-api` | Authentication, REST vs GraphQL, SDK selection (Node + React canonical), rate limits, pagination, idempotency |
| `stigg-pricing-modeling` | Model the catalog — features, plans, addons, products, charges, coupons, custom credit currencies, price localization |
| `stigg-entitlements` | Runtime gating, raw-events vs calculated-usage, cache and fallback strategy, promotional entitlements |
| `stigg-subscriptions` | Lifecycle ops — provision, preview, update, cancel, trials, multi-active subscriptions, plan-version migration |
| `stigg-credits` | Credit currencies, grants, ledger, consumption logic, custom formulas, auto-recharge, seat-based pools, billing integration |
| `stigg-widgets` | Drop-in UI — paywall, customer portal, checkout, credit widgets — driven by the live Storybook index |
| `stigg-webhooks` | Receive Stigg events — signature verification, payload envelope, retry semantics, idempotency, handler skeleton |
| `stigg-pricing-expert` | Advisory — picks the right monetization model and hands off to implementation skills |
| `stigg-recipes` | Composed end-to-end workflows — freemium, checkout, hybrid pricing, AI-credits monetization, trial with addons, payment links |

## Links

- [Stigg](https://stigg.io)
- [Stigg Documentation](https://docs.stigg.io)
- [Agent Skills Format](https://agentskills.io)
