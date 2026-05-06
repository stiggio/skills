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

More skills are added in subsequent commits — pricing modeling, entitlements & usage, subscriptions, credits, widgets, webhooks, a pricing-strategy advisor, an external-system migration playbook, and composed recipes.

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
