# Cache & Fallback — Production Reference

Entitlement checks are on the critical path. They must remain reliable when entitlement data is missing (a feature added in code before it's added to a plan) or when the Stigg API is briefly unreachable.

## Layers of resilience

```
App  →  In-memory SDK cache       (default; refreshed by polling / live updates)
     →  Persistent cache (Redis)  (optional; Node SDK + Sidecar)
     →  Sidecar service           (optional; separate process / deployment)
     →  Configured fallback rules (last line of defense)
     ↳  Stigg API (cold path)
```

Every layer above the API is optional except the in-memory cache, which is on by default.

## In-memory caching (default, all SDKs)

- The SDK caches the customer's effective entitlements after the first read.
- Stays current via **periodic polling** (frontend) or **real-time updates** (backend).
- In-memory caching is fine for most apps — most servers have stable processes.

## Persistent caching (large fleets / serverless)

For Lambda-style runtimes, container fleets, or anything that restarts often:

- Stigg ships a **persistent-cache-service** that consumes entitlement-update messages and keeps a Redis cache fresh.
- Supported in **Node SDK** and **Sidecar**. Other SDKs may follow — re-check the docs.
- Benefits: entitlements survive restarts; lower latency; fewer direct API calls; consistent data across instances.

## Sidecar service

A dedicated process / container that fronts Stigg:

- Local-network entitlement checks (sub-millisecond).
- Persistent caching baked in.
- Offline mode + global fallback strategy.
- Useful when the app must keep working independently of `api.stigg.io`.

Architecture, scaling, and ops concerns: the **High availability and scale → Sidecar** section of the docs. Re-fetch when designing for the Sidecar.

## Fallback rules — the safety net

When entitlement data is missing or the SDK can't reach Stigg *and* the cache is cold, fallbacks decide what to return.

- **Global fallback** — default behavior across all features. Often "allow" for soft enforcement, "deny" for security-sensitive features.
- **Per-entitlement fallback** — overrides the global. Useful for "always allow this critical feature" or "always deny this admin feature".
- The SDK marks the response with a fallback flag — log / monitor for anomalies.

> If you don't configure fallback rules and the SDK has no cached data and Stigg is unreachable, the SDK has nothing to return. Decide explicitly — don't rely on whatever default ships in the SDK.

## Usage reporting during outages

- **Entitlement checks**: keep working from cache.
- **Usage reports**: buffered in-memory and flushed when connectivity returns.
- **With persistent cache**: usage events can be persisted to Redis to survive process restarts during a long outage.

## Setup checklist (production)

1. **Pick a deployment model.** SDK in-app (in-memory only) → SDK + persistent cache → Sidecar. Pick by your fleet shape.
2. **Configure global fallback** rules. Default to "allow" unless you have a reason to deny on outage.
3. **Configure per-entitlement fallback** for the few features where the global default is wrong.
4. **Log fallback events.** Hook into the SDK's marker so you can graph "fallback rate" — a sudden spike means cache miss + outage.
5. **Test the cold-cache + outage case** before production. Kill network in staging, restart your process, see what your app returns.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying on in-memory cache in serverless | The cache dies with the process. Use persistent caching or the Sidecar. |
| No fallback rules configured | Outage + cold cache = your app has no entitlement data. Always configure. |
| Treating the fallback marker as noise | It's the early-warning signal. Monitor it. |
| Synchronously round-tripping to Stigg on every request | Defeats the cache. Let the SDK do its job. |
| Caching entitlements yourself in your app on top of the SDK | Double-cache — your TTL fights the SDK's. Don't. |
| Configuring "deny on fallback" globally | Can lock customers out during a transient blip. Almost always wrong as a default. |
