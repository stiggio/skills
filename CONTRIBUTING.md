# Contributing to Stigg Skills

Thanks for your interest in contributing. This guide covers structure, scope, and the process for submitting skills to this repo.

## What is a skill?

A skill is a markdown document that gives an AI agent actionable guidance for a specific Stigg task. When an agent loads a Stigg skill, it gets exactly enough context to call the right Stigg API correctly — without reading the full docs site.

Good skills are **opinionated and specific**. They tell the agent what to do, not everything it could do.

## Skill structure

Each skill lives under `skills/`:

```
skills/
  stigg-thing/
    SKILL.md              # Main skill file (required)
    references/           # Supporting docs (optional)
      deeper-topic.md
```

### `SKILL.md`

Every skill needs a `SKILL.md` with YAML frontmatter:

```yaml
---
name: stigg-thing
description: Use when [specific triggering conditions and symptoms]
---
```

The description is how agents decide whether to load your skill. Write it from the agent's perspective: start with **"Use when…"** and list concrete symptoms. **Do not summarize the skill's workflow** in the description — agents will follow the description shortcut and skip the skill body. The agent-skills spec caps frontmatter at 1,024 chars total; this repo's descriptions run 440–760 chars and that's fine. Pack triggers densely; avoid prose.

### `references/`

Use a `references/` directory when content is heavy enough that putting it in `SKILL.md` would dilute the core guidance. The main `SKILL.md` should be self-contained for the 80% case — references are deep dives.

**Put in `SKILL.md`:** setup, the most common patterns, decision guidance, common errors.

**Put in `references/`:** full parameter tables, response schemas, advanced configurations, edge cases.

## SDK scope

Code samples target the integration paths most Stigg customers use today:

- **Backend canonical:** `@stigg/node-server-sdk` (Node.js, async `await Stigg.initialize({ apiKey })`).
- **Frontend canonical:** `@stigg/react-sdk` for React (and Next.js with SSR), `@stigg/js-client-sdk` for vanilla JS.
- **AI-assisted dev:** the Stigg MCP server at `https://mcp.stigg.io`.
- **Universal fallback:** raw REST (`https://api.stigg.io/api/v1`) and raw GraphQL (`https://api.stigg.io/graphql`) via curl.

**Do not add inline code samples** for Python, Go, Java, Ruby, .NET, Vue, or Embed — these runtimes ship SDKs but their exact field names and init signatures aren't mirrored here. Use a one-line "see [docs.stigg.io](https://docs.stigg.io)" pointer instead. The auto-generated REST-based SDKs (`@stigg/typescript` and equivalents) are currently in beta — keep them as caveats, not recommendations.

This scope keeps verification rounds tractable and avoids cross-language field-name drift. When the auto-gen REST SDKs reach GA, this section will be updated.

## What makes a good Stigg skill

1. **Search-first.** Every skill instructs the agent to search live docs (Mintlify Stigg MCP / `docs.stigg.io/llms.txt`) before generating integration code. Stigg's API surface evolves; baked-in code goes stale.
2. **Scoped tightly, one job per skill.** "Set up the Stigg MCP" is a skill. "Use Stigg" is too broad.
3. **MCP-first when the Stigg MCP is connected.** If the `mcp.stigg.io` server is reachable, prefer it over hand-rolled REST/GraphQL — fewer hallucinations, schema-aware.
4. **Honest about limitations.** Include "when **NOT** to use this skill" — so agents don't load it for the wrong task.
5. **Token-efficient — by skill type:**
   - **Umbrella skill** (`stigg`) — loads on every Stigg conversation. Aim for **≤ 1,000 words**.
   - **Pillar skills** (the ones that own a substantive slice of Stigg — auth, MCP setup, pricing modeling, entitlements, subscriptions, credits, widgets, webhooks) — these are dense and the single source of truth for their domain. **≤ 1,800 words** is reasonable; aim lower when you can. Move heavy content to `references/*.md`.
   - **Advisory skills** (`stigg-pricing-expert`) and **composer skills** (`stigg-recipes`) — should stay tight. **≤ 1,500 words** for advisory; **≤ 800 words** for composers (recipes orchestrate references rather than carry content).
   - Run `wc -w skills/<name>/SKILL.md` to verify before submitting.
6. **No internal info leaks.** This repo is public — keep skills focused on what customers and integrators need.
7. **Tested against real usage.** Verify an agent can actually follow your skill to complete the task end-to-end. Snippets must work as-is.

## Submitting a skill

1. Fork the repo and create a branch.
2. Add your skill directory under `skills/`.
3. Update the **Available skills** table in `README.md`.
4. Open a PR with:
   - **What it does** — one sentence.
   - **Why it's useful** — what gap does it fill?
   - **How you tested it** — did an agent successfully use it end-to-end?

We review for structure, accuracy, and scope. Expect feedback — we'd rather iterate than merge a skill that isn't ready.

## Conventions

- All skill names use the `stigg-` prefix (the umbrella skill is just `stigg`).
- Use kebab-case directory and file names.
- Frontmatter `description` is in third person and starts with "Use when…".
- Reference live docs over hardcoded values where possible (rate limits, model lists, MCP install commands change).
- Frontmatter is capped at 1,024 chars by the agent-skills spec. Most descriptions in this repo land 440–760 chars — pack triggers, not prose.
- Standard markdown only — no HTML, no inline scripts.
- Do not embed API keys, customer IDs, or other secrets in examples; use placeholders like `<YOUR_API_KEY>`.

## Questions?

Open an issue on this repo, or visit [docs.stigg.io](https://docs.stigg.io).

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](./LICENSE).
