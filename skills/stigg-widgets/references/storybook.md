# Stigg Widgets Storybook — Agent Recipe

The Storybook at `https://widgets.stigg.io/` is the **live, authoritative reference** for widget variants, props, and code snippets. It's JavaScript-rendered, so a basic HTTP fetch returns only the title. **But the index is machine-readable.**

## Discovery — `index.json`

```text
GET https://widgets.stigg.io/index.json
```

Returns the full story index. Each story has an ID and a component name. **Always fetch this first** when the user references widgets — it's the canonical, current list of variants.

## Story IDs — naming pattern

```text
widgets-<component>--<variant>
```

Categories at the time of writing (re-fetch `/index.json` for the current set):

- **`widgets-paywall--*`** — paywall variants: `playground`, `with-custom-theme`, `authenticated-customer`, `upgrade-only`, `async-plan-selection`, `resource-based`, `with-addons`, `loading-state`.
- **`widgets-checkout--*`** — `playground`, `with-promotion-code`, `collect-phone-number`, `collect-tax-id`.
- **`widgets-customer-portal--*`** — `playground`, `with-custom-theme`, `resource-based`, `modular-layout`, `subscriptions-only`, `billing-details`, `usage-tracking`, `addons-management`.
- **`widgets-credits-credit-balance--*`** — credit-balance variants.
- **`widgets-credits-credit-grants--*`** — credit-grants table variants.
- **`widgets-credits-credit-usage-chart--*`** — credit-usage-chart variants.
- **`widgets-credits-credit-utilization--*`** — credit-utilization variants.
- **`widgets-credits-automatic-recharge--*`** — auto-recharge widget variants.

## Workflow when the user says "build a paywall"

1. Fetch `https://widgets.stigg.io/index.json` to see all current paywall variants.
2. **Open the docs page** `docs.stigg.io/documentation/snap-in-widgets/paywall` (or the equivalent for the widget) for the canonical prop tables and code snippets.
3. Mention the matching Storybook URL for the human: `https://widgets.stigg.io/?path=/story/widgets-paywall--<variant>`.
4. Author the integration code referencing the docs-confirmed props and the user's stack (React / Next.js for the primary path; Vue / vanilla JS / Embed if applicable).
5. Don't memorize props from past sessions — variants change.

## Why a basic WebFetch returns only the title

The Storybook is a single-page app rendered by JavaScript. `WebFetch` (and similar tools) deliver the static HTML, which only has `<title>storybook - Storybook</title>` and no story content. To inspect a specific story's rendered output, a human (or a headless browser) needs to load the Storybook URL.

For the agent's purposes, **`/index.json` plus the Mintlify docs MCP for prop tables is enough.**

## Per-widget canonical doc paths

Use these alongside the Storybook for prop / code-snippet detail:

| Widget | Docs path |
|---|---|
| Paywall / Pricing Table | `/documentation/snap-in-widgets/pricing-table.mdx`, `/documentation/snap-in-widgets/paywall.mdx` (where applicable) |
| Customer Portal | `/documentation/snap-in-widgets/customer-portal.mdx` |
| Checkout | `/documentation/snap-in-widgets/checkout.mdx` |
| Credit widgets | `/documentation/modeling-your-pricing-in-stigg/credits/widgets.mdx` |

(Use `query_docs_filesystem_stigg` against these `.mdx` paths via the Mintlify Stigg docs MCP.)
