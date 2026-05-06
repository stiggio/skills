# Stigg Rate Limits — Operating Principles

Rate limits and the "Using Stigg at Scale" guidance evolve. **Always re-fetch the live numbers from [docs.stigg.io](https://docs.stigg.io)** (the "Rate Limits (GraphQL & REST)" page and the "Using Stigg at Scale" page in the API Reference). The principles below stay stable.

## What's rate-limited

- **Per-environment, per-endpoint.** Different endpoints have different ceilings.
- **HTTP `429 Too Many Requests`** signals that you've hit one. Production paths must handle it.

## Operating principles

1. **Don't synchronously call Stigg on every request that doesn't need it.** Cache effective entitlements for the customer. Stigg already pre-computes them — the SDKs can cache the result client-side.
2. **Push high-volume usage through the events path** (`reportEvent` / `events.report`-style ingestion) rather than per-request synchronous reports — covered in `stigg-entitlements`. The "scaling" distinction is by throughput, not by who counts.
3. **Use the persistent cache / Sidecar** when latency or independence from `api.stigg.io` matters more than perfect freshness.
4. **Idempotency on writes.** `Idempotency-Key` headers (24h cache) make safe retries possible without double-creates.
5. **Backoff on `429`.** Exponential with jitter, capped — don't hammer the same endpoint after the limit.

## Patterns to apply

### Read path (gating)

```
Incoming request → SDK in-memory cache → (miss) → Stigg → cache → respond
                                       ↓
                                   on transport error: SDK fallback (last-known-good)
```

### Write path (usage)

```
Customer action → enqueue event locally (durable) → batch flush to events.report
                                                  ↓
                                           on 429 / 5xx: backoff + retry
```

### Catalog ops (rare, in the UI / CI)

Catalog edits are not rate-limited like read paths, but bulk operations should still be batched. Use the bulk-import endpoints (`POST /api/v1/customers/import`, `POST /api/v1/subscriptions/import`) rather than looping individual creates.

## Where to look

- [Rate limits (GraphQL & REST)](https://docs.stigg.io/api-and-sdks/api-reference/rest/rate-limits) (re-confirm path) — current numbers.
- "Using Stigg at Scale" — architectural patterns for production volume.
- "High availability and scale" / "Local caching and fallback strategy" — durability and offline behavior.

## Don't memorize specific numbers

The numbers change as Stigg's infrastructure evolves. Re-fetch the live limits before authoring back-pressure logic, sizing batch jobs, or designing your retry config.
