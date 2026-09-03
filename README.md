# Stigg Skills

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![Stigg Docs](https://img.shields.io/badge/docs-docs.stigg.io-6C5CE7)](https://docs.stigg.io)
[![Agent Skills Format](https://img.shields.io/badge/format-agentskills.io-2ecc71)](https://agentskills.io)

**Agent skills for [Stigg](https://stigg.io)** — the pricing, packaging, entitlements, and credits-based monetization platform. Drop these into Claude Code, Claude Desktop, ChatGPT, Cursor, VS Code, Windsurf, Codex, or any Agent-Skills-compatible client and your coding agent knows how to integrate Stigg correctly on the first try: which SDK to use, how to model a pricing catalog, how to gate features at runtime, how to handle credits, webhooks, and checkout — without guessing at API shapes or going stale mid-conversation.

Maintained by the Stigg team. Community contributions welcome — see [Contributing](#contributing).

## Contents

- [Why use these skills](#why-use-these-skills)
- [Installation](#installation)
- [Available Skills](#available-skills)
- [Staying up to date](#staying-up-to-date)
- [Contributing](#contributing)
- [Support](#support)

## Why use these skills

- **MCP-first, always current.** Every skill instructs the agent to search live docs before generating code, so integrations don't drift from the actual API surface.
- **One job per skill.** Twelve focused skills instead of one giant prompt — the umbrella `stigg` skill routes to the right one automatically.
- **Covers the full integration surface.** Auth, pricing catalog modeling, runtime entitlements, subscriptions, credits, governance, widgets, webhooks, and end-to-end recipes.
- **Built for real client integrations.** Snippets are tested against the SDKs Stigg customers actually use in production.

## Installation

You'll need a [Stigg account](https://app.stigg.io) (Stigg provisions a free sandbox environment automatically). Then pick the install path that matches your client.

### Claude Code plugin

The skills ship as a Claude Code plugin via Stigg's marketplace:

```bash
# Add the Stigg marketplace
/plugin marketplace add stiggio/skills

# Install the Stigg plugin (all 13 skills)
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
| `stigg-governance` | Per-entity budget enforcement — entity types with attribution keys, entity hierarchies, usage-limit assignments per feature or credit currency, governance tree balances |
| `stigg-widgets` | Drop-in UI — paywall, customer portal, checkout, credit widgets — driven by the live Storybook index |
| `stigg-webhooks` | Receive Stigg events — signature verification, payload envelope, retry semantics, idempotency, handler skeleton |
| `stigg-pricing-expert` | Advisory — picks the right monetization model and hands off to implementation skills |
| `stigg-recipes` | Composed end-to-end workflows — freemium, checkout, hybrid pricing, AI-credits monetization, trial with addons, payment links |
| `stigg-sp` | Provision a Stigg environment through Stripe Projects — `stripe projects add stigg/environment`, what the provisioned credentials are, first wiring of the server and client SDKs |

## Staying up to date

Skills are delivered as the `stigg` plugin via the `stigg-marketplace`. To get the latest:

- Enable auto-update: `/plugin` → Marketplaces → toggle **Enable auto-update** for `stigg-marketplace`, then `/reload-plugins`.
- Or update manually: `/plugin marketplace update stigg-marketplace` then `/reload-plugins`.

Updates apply per plugin (all Stigg skills move together), and only when a new version is released. See [CHANGELOG.md](./CHANGELOG.md) for release notes.

## Contributing

Want to fix a skill or add a new one? See [CONTRIBUTING.md](./CONTRIBUTING.md) for skill structure, scope, and the PR process.

## Support

- **Questions or issues:** [open an issue](https://github.com/stiggio/skills/issues) on this repo.
- **Product docs:** [docs.stigg.io](https://docs.stigg.io)
- **Talk to us:** [stigg.io](https://stigg.io)

## License

[MIT](./LICENSE)
