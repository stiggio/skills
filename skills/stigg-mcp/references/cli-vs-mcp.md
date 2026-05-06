# Stigg CLI vs Stigg MCP Server

Both give programmatic access to Stigg. They are **complementary**, not alternatives — most teams end up using both.

## Side by side

|  | **Stigg CLI** | **Stigg MCP server** |
|---|---|---|
| Best for | Scripts, CI/CD, terminal workflows | AI-assisted development inside coding assistants |
| How you interact | Shell commands with flags | Natural-language prompts to an agent |
| Requires | Terminal access | An AI client that speaks MCP (Claude Code, Cursor, etc.) |
| Deterministic? | **Yes** — exact commands | **No** — the agent decides which calls to make |
| Good for unattended automation? | **Yes** — scriptable, pipe-friendly | **No** — not designed for unattended runs |

## When to pick which

- **CLI** — anytime you need to know exactly what API call will run. CI pipelines, scheduled catalog migrations, reproducible local scripts, "copy plans from staging to prod" runbooks.
- **MCP** — anytime you're already inside an AI assistant and want to describe the goal in plain English. Modeling pricing, exploring an environment, one-off ops.

## Don't confuse them

- The MCP isn't a "smarter CLI." It's a translation layer for an LLM agent.
- The CLI isn't "the MCP without the AI." It's a deterministic shell tool with no model in the loop.

## Pattern: use both

A common workflow:

1. **Model with the MCP.** Ask the agent to design a Pro plan with two metered features. Iterate in natural language. The agent calls the right MCP tools.
2. **Lock in with the CLI.** Once the catalog is right, capture the deterministic CLI commands (or an export) and check those into your repo / runbook for reproducibility.
3. **Roll out via CI** using the CLI. The CLI sees only what you tell it to do — no agent-driven surprises in production.
