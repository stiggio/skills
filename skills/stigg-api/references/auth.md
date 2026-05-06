# Stigg API Auth — Full Reference

Sourced from [docs.stigg.io / API keys](https://docs.stigg.io/documentation/managing-your-account/environments/api-keys). Re-fetch for the canonical version.

## Key types — full matrix

| Type | Prefix | Created by | Scopes | Where used | Customizable scope? |
|---|---|---|---|---|---|
| **Full access** | `server-` | System (one per environment) | All current + future permissions | Backend, server-to-server | No |
| **Publishable** | `client-` | System (one per environment) | Read-only, narrow | Frontend / mobile SDKs | No |
| **Scoped** | `server-` | Users, unlimited per environment | User-defined preset | Backend, per-service | Yes |
| **Integration** | `server-` | Stigg (auto, per integration) | Immutable per integration | Reserved for the integration | No |

Scoped keys require the **Scale plan**; all plans get the default Full access + Publishable pair.

## Custom permission presets (scoped keys)

| Preset | What it grants |
|---|---|
| Full access | Read/write across all resources in the environment |
| Read only | Read across all resources, no writes |
| Customers write | Create / update customers (implies customer read) |
| Subscriptions write | Create / update subscriptions (implies subscription read) |
| Coupons write | Manage coupons / discounts (implies read) |
| API keys read and write | Programmatically manage API keys |
| Event Queue read | Obtain temporary SQS event-queue credentials (no write op exists) |

Selecting a write permission on a resource also grants read on it.

## Key statuses

| Status | Behavior |
|---|---|
| Active | Valid; accepts all requests |
| Expiring soon | Being rotated; valid until grace ends |
| Expired | Invalid; all requests get `401 Unauthorized` |

## Lifecycle ops

### Create a scoped key

1. **Integrations → API keys → + Add scoped key**
2. Name + optional description.
3. Configure scope per resource.
4. **Create.** Copy the secret immediately, or reveal it later via the **Show / Hide** toggle in the key's detail panel.

### Rotate a key (zero downtime)

1. `⋮` next to the key (or open its detail panel) → **Rotate key**.
2. Pick when the old secret expires: Now / 1h / 24h / 3d / 7d.
3. Confirm.
4. Update services to use the new secret before the grace ends.
5. Need more time? Open `⋮` → **Change grace period**.

### Revoke a key

`⋮` → **Revoke** → confirm. **Immediate, irreversible.** Switch services first.

## Where each key goes — strict rules

### Backend (server-side)

```ts
// Node — @stigg/node-server-sdk (async init)
import Stigg from '@stigg/node-server-sdk';
const stigg = await Stigg.initialize({ apiKey: process.env.STIGG_SERVER_API_KEY! });
```

> Stigg also has an auto-generated REST-based SDK (`@stigg/typescript`) currently in beta — wait for GA before adopting.

**Server keys (full access or scoped)** only. **Never** in browser bundles, public repos, or mobile binaries.

### Frontend (browser / mobile)

```tsx
import { StiggProvider } from '@stigg/react-sdk';

<StiggProvider apiKey="client-..." customerId="customer-123">
  {children}
</StiggProvider>
```

**Publishable key** only.

### Direct REST

```bash
curl -X GET "https://api.stigg.io/api/v1/customers" \
  -H "X-API-KEY: $STIGG_SERVER_API_KEY" \
  -H "Content-Type: application/json"
```

### Direct GraphQL

```bash
curl -X POST https://api.stigg.io/graphql \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $STIGG_SERVER_API_KEY" \
  -d '{"query":"{ customers { edges { node { customerId name } } } }"}'
```

> Header capitalization differs between REST (`X-API-KEY`) and GraphQL (`X-API-Key`) in the docs. Both servers are case-insensitive on header names per HTTP spec — match the docs to be safe.

## Client-side security hardening

The publishable key is visible in client-side code by design. To prevent unauthorized cross-customer access, Stigg supports **HMAC SHA256 signed customer tokens**: your backend signs a token tied to the customer ID, the frontend includes it on every request, Stigg validates.

Enable from the publishable key's **detail panel → Overview**. Full guide: [Hardening client-side access](https://docs.stigg.io/api-and-sdks/integration/frontend/hardening-client-side-access).

## Audit trail

- The key's **Activity tab** logs only lifecycle events for the key itself: created, rotated, scope updated, revoked.
- **Logs → Activity** in the Stigg app shows all API activity across the environment, with the key used per call.

## Access control (RBAC inside the Stigg app)

| Action | Owner | Member | Read-only |
|---|---|---|---|
| View key prefix | ✅ | ✅ | ✅ |
| Copy full key value | ✅ | ❌ | ❌ |
| Create / rotate / revoke | ✅ | ❌ | ❌ |

## Migration from legacy keys

| Legacy | New |
|---|---|
| Client API Token | Publishable key |
| Server API Token | Full access key |

No action required — existing integrations continue to work.

## Anti-patterns

- **Sharing one full-access key across many services.** A leak compromises the whole environment. Use scoped keys.
- **Hard-coding keys in source.** Use env vars, secret managers, or the platform's secret store. Rotate any key that appears in a commit, even briefly.
- **Reusing one key across environments.** Keys are environment-bound by design. Don't try to "shortcut" multi-env setups by reusing keys — you can't.
- **Putting the publishable key in production without client-side hardening.** Customers can see each other's data without it. Enable HMAC signed tokens.
- **Treating `Expiring soon` as harmless.** It's a countdown. Update services *before* expiration, not after.
