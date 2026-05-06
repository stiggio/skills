# The Search-First Rule

The single most important rule for working with Stigg from an AI agent.

## The rule

> **Before generating any Stigg integration code, search the live docs.**

This applies even when you're "sure" you remember how something works. Stigg ships fast — endpoint shapes, SDK init signatures, recommended patterns, and rate limits all change.

## Why this matters

- **Hallucinated patterns are the #1 customer-agent failure.** Agents confidently produce SDK calls with the wrong import path, wrong method name, or wrong required field.
- **`events.report` vs `usage.report`** is the classic example. They look similar; their scaling characteristics differ. Reading the right doc page each time prevents picking the wrong one.
- **MCP coverage expands continuously.** Operations that needed the UI six months ago are MCP-doable now. Re-checking ensures you don't fall back to manual steps that are no longer required.

## What "search the live docs" means in practice

In rough order of preference:

### 1. Stigg MCP server (best, when available)

If the user has `mcp.stigg.io` connected, the agent can both **search** and **execute** against the actual environment. Use the MCP first whenever it's reachable. See the `stigg-mcp` skill for setup.

### 2. Mintlify docs MCP (`docs.stigg.io` knowledge base)

If the agent has the docs MCP available, prefer it over generic web fetches:

- `search_stigg(query)` — broad/conceptual lookups
- `query_docs_filesystem_stigg(command)` — exact ripgrep / cat-style access to the docs as a virtual filesystem

Common patterns:

```bash
# Find pages mentioning rate limits
rg -il "rate limit" /

# Read a specific page
cat /api-and-sdks/api-reference/rest/overview.mdx

# Layout
tree / -L 2
```

### 3. `https://docs.stigg.io/llms.txt`

The LLM-friendly URL index. Fetch it once per session, find the relevant section, then fetch the linked page. Useful when no docs MCP is available.

### 4. Direct fetch of `docs.stigg.io/<path>`

Last resort, only when neither MCP is available and `llms.txt` doesn't disambiguate.

## How to integrate the rule into your turn

```text
User asks: "Add Stigg to gate this feature."
1. Search:  "feature gating" + "entitlement check" via Mintlify MCP / llms.txt.
2. Open the page that matches the user's stack (Node SDK / React SDK / REST).
3. (If Stigg MCP is connected) confirm the feature exists / check the env / dry-run.
4. NOW write the integration code.
```

If you wrote any Stigg code without doing steps 1–3, delete it and start over. The skill `stigg` is explicit about this — it's not a stylistic preference, it's the operating contract.

## Red flags — STOP and search

You are about to violate the rule if:

- You typed `import Stigg from '@stigg/...'` and have not opened a docs page this turn.
- You wrote a `curl https://api.stigg.io/...` from memory.
- You're authoring a GraphQL mutation against `api.stigg.io/graphql` without checking the schema page.
- You're suggesting "the Stigg MCP doesn't support that" without re-checking the MCP coverage.

In every case: stop, search, then proceed.

## Common rationalizations (and the correct response)

| "I'm sure of the call shape" | Be sure *and* search. Costs ~5 seconds. |
| "This is a trivial example" | Trivial code with the wrong field name still fails for the user. Search. |
| "The user is in a hurry" | A wrong snippet is slower than a right one. Search. |
| "I just searched two turns ago" | Different page → different content. If the question changed surface, search again. |
| "I'll search if it doesn't work" | Then it doesn't work for the user, who has to roll back. Search first. |
