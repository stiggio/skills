# Bulk Import Customers — Reference

The `bulkImportCustomers` endpoint imports many customers in one async call. Used during Stage 1 backfill of the migration playbook.

## Endpoint

- **REST:** `POST /api/v1/customers/import`
- **GraphQL:** `importCustomerBulk` mutation (singular `Customer`; input type `ImportCustomerBulkInput`)
- **SDK:** the bulk-import method exposed by your SDK

> **Re-fetch the docs page** before authoring — the input shape and the response envelope evolve.

## Recommended approach

1. **Use a dedicated import API key.** Production keys hit rate limits during large imports. **Contact Stigg support** to issue a separate scoped key for the import.
2. **Test in sandbox first.** A botched bulk import is hard to undo cleanly.
3. **Batch reasonably.** Stigg accepts large payloads but smaller batches make retries simpler.
4. **Track enrollment in your source system.** Mark each customer `enrolled = true` after a successful import; skip on retry.
5. **Pull payment-method identifiers** from your billing provider into Stigg's customer record where needed (downstream subscription import will use them).

## Per-customer fields

Field names differ by surface — verify exact field names per docs for your call path.

**REST `POST /api/v1/customers/import`** body shape `{ customers: [{ ... }] }`:

| Field | Notes |
|---|---|
| `id` | **Required.** Your stable customer identifier. Immutable in Stigg once set. |
| `name` | Optional display name. |
| `email` | Optional. |
| `billingId` | Identifier in your billing provider (e.g. Stripe customer ID). |
| `metadata` | Arbitrary key/value for integration logic. |
| `coupons` | Pre-applied coupons (rare during import). |

**GraphQL `importCustomerBulk` mutation** uses `customerId` (not `id`) on the input items — confirm in the mutation reference page before authoring raw GraphQL.

## Async semantics

The import is **asynchronous**:

- Submit a batch → get a job ID back.
- Poll for status until complete.
- Inspect failures per row.
- Retry failed rows individually (don't re-submit the whole batch — you'll create duplicate enrollment markers).

## Idempotency

Stigg keys customers by `customerId`. Re-submitting the same customer is **mostly safe** (the customer record won't duplicate), but be deliberate:

- **Track enrollment** in your source-system database.
- **Skip already-enrolled** customers on retry.
- **Use unique batch IDs** in your job tracker to avoid running the same batch twice.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hitting rate limits mid-import | Use a dedicated import key (contact support). |
| Importing customers without `billingId` | Downstream subscription import can't link to the billing provider. Pull the ID before import. |
| Re-running the batch on partial failure | Retry only the failed rows, not the whole batch. |
| Importing into production before sandbox validation | Always sandbox first. |
| Creating duplicate enrollments by re-running without an enrollment flag | Track enrollment in your source DB. |
| Using legacy customer IDs that won't be referenced from your app post-cutover | Use stable IDs your app will keep using post-cutover, not just the legacy ones. |
