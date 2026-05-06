# SDK Selection — Pick the Right Client

These skills focus on the integration paths most Stigg customers use today: **`@stigg/node-server-sdk` for backend, `@stigg/react-sdk` (or `@stigg/js-client-sdk`) for frontend, the Stigg MCP server for AI-assisted dev, and the raw REST / GraphQL API for everything else.**

Stigg ships SDKs for other runtimes (Python, Go, Java, Ruby, .NET, Vue, Sidecar) — they work and are documented at [docs.stigg.io](https://docs.stigg.io), but the skills don't try to mirror every runtime's exact field names and init signatures. If you're on one of those runtimes, see the SDK's own docs page; the patterns below transfer.

## Backend

### Default — Node.js

```ts
import Stigg from '@stigg/node-server-sdk';

const stigg = await Stigg.initialize({
  apiKey: process.env.STIGG_SERVER_API_KEY!,
});
```

**Currently in beta — wait for GA before adopting:** the auto-generated REST-based SDKs (`@stigg/typescript` and equivalents in other languages). Once they ship, the recommendation may flip; today, stay on `@stigg/node-server-sdk`.

### Other backend runtimes

For Python (`stigg-api-client-v2`), Go, Java, Ruby, .NET, or the Sidecar service: see the SDK's own page on [docs.stigg.io/api-and-sdks/integration/backend](https://docs.stigg.io/api-and-sdks/integration/backend). Init signatures vary; field names may differ from the Node SDK and from the underlying GraphQL mutation. Treat the Node-SDK examples in these skills as illustrative — confirm the exact shape against your SDK's README before authoring.

### No SDK? Use raw HTTP.

If your runtime isn't covered, hit the API directly:

- **REST:** `https://api.stigg.io/api/v1` — header `X-API-KEY`. See the REST API reference on docs.stigg.io for endpoints + body shapes.
- **GraphQL:** `https://api.stigg.io/graphql` — header `X-API-Key`. See the GraphQL reference for mutations + queries.

Direct HTTP is fine for production — Stigg's API is the contract. The SDKs are conveniences on top.

## Frontend

### Primary — React (also covers Next.js)

```tsx
import { StiggProvider } from '@stigg/react-sdk';

<StiggProvider apiKey="<STIGG_CLIENT_API_KEY>" customerId="<LOGGED_IN_CUSTOMER_ID>">
  <NestedComponents />
</StiggProvider>
```

**Next.js:** `@stigg/react-sdk` supports SSR (since v0.2.1) — there is no separate Next.js package. Use the React SDK in your Next.js app directly.

### Vanilla JS / web components

```ts
// Stigg.initialize() is synchronous; await waitForInitialization()
import { Stigg } from '@stigg/js-client-sdk';

const stiggClient = Stigg.initialize({
  apiKey: 'client-...',
  customerId: 'customer-123',
});
await stiggClient.waitForInitialization();
```

### Other frontend stacks

`@stigg/vue-sdk` exists for Vue projects; the Embed SDK supports drop-in iframe / CDN-style integration. See the SDK pages on docs.stigg.io for stack-specific snippets.

All frontend SDKs use the **publishable** (`client-`) key, never a server key. Production deployments should also enable client-side security hardening (HMAC SHA256 signed customer tokens) — see `stigg-api/references/auth.md`.

## Picking SDK vs raw HTTP (backend)

| You should… | When |
|---|---|
| Use the Node SDK | You're on Node / TypeScript and want typed methods, retries, rate-limit handling, in-memory caching of entitlements |
| Use raw REST | You're on a runtime not yet covered by a Stigg SDK, you're writing a thin proxy / gateway, or you need explicit HTTP control |
| Use raw GraphQL | Maintaining an existing GraphQL integration, or a query the SDKs don't expose yet |

## Persistent cache / Sidecar

For high-throughput services, Stigg offers a **persistent cache service** and a **Sidecar** deployment model that fronts the API for low-latency entitlement checks and offline fallback. Operational/architecture decisions — start with the regular Node SDK and graduate to the Sidecar when latency, durability, or independence from `api.stigg.io` becomes a hard requirement. See the Sidecar docs section on docs.stigg.io.

## Versioning

- Each SDK has its own changelog on [docs.stigg.io](https://docs.stigg.io). Watch the **SDK Changelog** section for breaking changes before upgrading.
- Pin major version with a floor (e.g. `^4.x`) so transitive bug fixes apply.
- The Stigg REST API itself is currently in **Public Beta** — re-check status before relying on it for high-stakes flows.
