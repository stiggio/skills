# MCP Coverage Matrix — Catalog Operations

**Status:** the Stigg MCP exposes the full Stigg API surface. The goal is **zero UI fallbacks** for catalog modeling.

This file is a checklist for the agent: when in doubt about whether an op is MCP-doable, **search the Stigg MCP's tool list first** (`/mcp` in Claude Code, or the equivalent in your client). MCP coverage expands continuously — anything in this file marked "UI only" is a snapshot, and should be re-verified before falling back.

## How to use this file

1. **Default: try the MCP.** Don't ask the user to open the Stigg app unless the MCP doesn't expose the operation.
2. **If the MCP returns "tool not found" or the equivalent**, *re-search* the docs and the MCP tool listing — coverage may have changed since this file was written.
3. **If genuinely UI-only**, name the operation here, link to the Stigg app page, and open an issue / Stigg ticket asking for MCP coverage.

## Catalog operations — coverage table

| Area | Operation | Status | Notes |
|---|---|---|---|
| **Features** | Create boolean / configuration / metered feature | MCP | Re-verify metered meter aggregation options. |
| **Features** | Edit feature details | MCP | |
| **Features** | Archive / unarchive feature | MCP | |
| **Feature groups** | Create / update / archive feature groups | MCP | |
| **Plans** | Create plan (free / paid / custom) | MCP | |
| **Plans** | Edit a plan draft | MCP | |
| **Plans** | Publish plan + pick migration type | MCP | |
| **Plans** | Archive / unarchive plan | MCP | |
| **Plans** | View plan history / list | MCP | |
| **Plans** | Variable entitlement values (custom plans) | MCP | |
| **Plans** | Defining overages | MCP | |
| **Add-ons** | Create / edit / publish add-on (paid / custom) | MCP | |
| **Add-ons** | Define add-on price | MCP | |
| **Add-ons** | Define compatibility | MCP | |
| **Products** | Create / edit / archive product | MCP | |
| **Coupons** | Create / edit / apply / archive coupon | MCP | |
| **Credits — currencies** | Full lifecycle: define / update / archive / unarchive / list / list associated entities | MCP | Currencies live at the account level. `list associated entities` returns active plans + add-ons referencing the currency — call before archiving so the archive doesn't fail on dependencies. |
| **Credits — consumption mapping** | Configure a metered feature's credit consumption **on a plan** (per-plan, not per-feature) | MCP | |
| **Credits — formulas** | Custom credit consumption formulas — also **per-plan** | MCP | Re-verify formula syntax. |
| **Price localization** | Set localized prices per currency | MCP | No-code UI is also available. |
| **Price localization** | No-code configuration in the Stigg app | UI | Some teams prefer the UI for pricing changes; both paths produce the same result. |
| **Account-level settings** | API key creation, scope changes, rotation | UI (recommended) | Programmatic key management exists via the **API keys read and write** scoped permission preset, but the Stigg app UI is the recommended path for sensitive ops. See `stigg-api/references/auth.md`. |
| **Environments** | Create new environment / copy changes between envs | Mixed | Copying changes between envs is supported via UI; some pieces also via CLI. |

## Update protocol

When you (the skill author) confirm new MCP coverage:

1. Mark the row **MCP** here.
2. Remove any "UI fallback" wording from the parent skill that describes the now-MCP-doable op.
3. Don't add new UI-only rows unless you've confirmed there's truly no MCP path.

## Don't memorize this file

It's a living snapshot. The right move on every turn:

1. Search docs / call the live MCP tool listing first.
2. Treat this file as "what to expect", not as the source of truth.
