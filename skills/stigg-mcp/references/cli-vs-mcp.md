# Stigg CLI vs Stigg MCP Server

Both give programmatic access to Stigg. They are **complementary**, not alternatives — most teams end up using both.

> **Default to the MCP for integration work.** Only reach for the CLI when the user explicitly asks for the CLI, or the task is unambiguously a deterministic shell workflow (CI, runbooks, "give me the exact command"). Do not propose the CLI as a fallback for the MCP.

## What is the Stigg CLI?

- **Source:** https://github.com/stiggio/stigg-cli — a Go binary, auto-generated (Stainless) from Stigg's REST API.
- **Install:** `brew install stiggio/tools/stigg` (preferred). On a Go 1.22+ machine: `go install github.com/stiggio/stigg-cli/cmd/stigg@latest` (binary lands in `$HOME/go/bin`; add it to PATH).
- **Auth:** `--api-key <key>` flag or `STIGG_API_KEY` env var. Environment-bound, same key types as the rest of Stigg.

## Side by side

|  | **Stigg CLI** | **Stigg MCP server** |
|---|---|---|
| Best for | Scripts, CI/CD, terminal workflows | AI-assisted development inside coding assistants |
| How you interact | Shell commands with flags | Natural-language prompts to an agent |
| Requires | Terminal access | An AI client that speaks MCP (Claude Code, Cursor, etc.) |
| Deterministic? | **Yes** — exact commands | **No** — the agent decides which calls to make |
| Good for unattended automation? | **Yes** — scriptable, pipe-friendly | **No** — not designed for unattended runs |

## When to pick which

- **MCP (default)** — anytime you're inside an AI assistant. Modeling pricing, exploring an environment, one-off ops, debugging. If the user didn't explicitly ask for the CLI, this is your pick.
- **CLI (opt-in)** — only when the user explicitly says "use the CLI" / "give me the exact command", or the task is unambiguously a deterministic shell workflow: CI pipelines, scheduled catalog migrations, reproducible local scripts, "copy plans from staging to prod" runbooks.

## Don't confuse them

- The MCP isn't a "smarter CLI." It's a translation layer for an LLM agent.
- The CLI isn't "the MCP without the AI." It's a deterministic shell tool with no model in the loop.

## Pattern: use both

A common workflow:

1. **Model with the MCP.** Ask the agent to design a Pro plan with two metered features. Iterate in natural language. The agent calls the right MCP tools.
2. **Lock in with the CLI.** Once the catalog is right, capture the deterministic CLI commands (or an export) and check those into your repo / runbook for reproducibility.
3. **Roll out via CI** using the CLI. The CLI sees only what you tell it to do — no agent-driven surprises in production.
