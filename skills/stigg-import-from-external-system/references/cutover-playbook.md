# Cutover Playbook — Reference

The day-of (or week-of) cutover from "Stigg in shadow-read" to "Stigg as source of truth". Concrete checklist.

## Prerequisites — don't cut over without these

- ✅ **Customer + subscription parity confirmed** through shadow-reads. Mismatch rate < your configured threshold (commonly < 0.1% over the last 7 days).
- ✅ **Recent usage replayed** so running totals are correct for active subs.
- ✅ **Stigg's billing-provider integration tested end-to-end** in sandbox: provision → preview → update → cancel → invoice generation.
- ✅ **Customer portal / paywall live** in your app and rendering correctly (`stigg-widgets`).
- ✅ **Webhook handlers wired** for subscription / trial / payment events you care about.
- ✅ **Rollback plan documented** — how do you revert if something breaks?
- ✅ **Customer support team trained** on Stigg's app for support workflows.

If any of these are missing, **do not cut over**. The remediation cost post-cutover is much higher.

## Cutover sequence

### T-7 days: communicate

- **Internal:** notify support, sales, finance. Share rollback procedure.
- **External:** if customer-facing UX changes (e.g. customer portal lives at a new URL), notify customers.

### T-1 day: final shadow-read review

- Pull the discrepancy dashboard. Confirm rate is in spec.
- Run a **dry-run cutover** in staging if possible — flip the flag, exercise the most-used flows, flip it back.

### T-0: flip the global flag

```ts
// Toggle the routing flag globally
await featureFlags.set('use-stigg', true);
```

All subscription / entitlement flows now go through Stigg. Watch the error / mismatch metrics dashboard live for the next several hours.

### T+0 to T+1 hour: smoke test

Exercise the production flows manually:

- New signup → customer + subscription created in Stigg.
- Existing customer's gating check returns the expected entitlement.
- Customer portal renders correctly.
- Paid plan upgrade → Stripe charge → Stigg subscription activated.
- Usage event ingested → ledger updated.

If anything fails, **toggle the flag back to legacy** (the rollback). Investigate offline.

### T+1 day: graceful legacy cancellations

For customers whose subscriptions are now managed by Stigg:

- **Schedule cancellations in the legacy system** to avoid duplicate charges.
- If Stigg created backdated subs through its native billing connector, the legacy sub can typically be cancelled once the corresponding Stigg sub is confirmed active.
- Coordinate with Finance on revenue-recognition cutover (legacy revenue stops; Stigg revenue starts).

### T+1 week: keep monitoring

- Continue logging errors and mismatches. Catch regressions early.
- Don't decommission the legacy system yet. **Retain validation logic temporarily.**
- Compare period-end revenue between legacy and Stigg sources. They should reconcile.

### T+2 to T+4 weeks: sunset

In phases:

1. **Turn off writes** to the legacy system. Stigg is now the only writer.
2. After ~1 week of stable Stigg-only writes: **turn off reads** from the legacy system (in your routing layer).
3. After another week: **clean up infrastructure / code** — delete unused legacy modules, decommission legacy services.

> Don't do all three on day one. Each step needs its own observation window.

### T+1 month: post-mortem

Share outcomes with the team:

- Performance wins (latency, error rate, throughput).
- Operational improvements (pricing changes without engineering, etc.).
- Migration stats (rollout cohort progression, mismatch rates over time, support tickets).

This builds confidence in the new system and informs future migrations.

## Rollback — when something breaks

**Don't panic.** The Strangler Fig pattern is rollback-friendly:

1. **Toggle the flag back to legacy.** All routes now go through the legacy system again. Customers are unaffected.
2. **Diagnose offline.** Compare the failing flow's behavior in legacy vs Stigg. Identify the root cause.
3. **Fix in Stigg / your integration.** Re-test in shadow-read.
4. **Re-attempt cutover** when the issue is resolved.

The investment in the abstraction layer (Stage 2 of the migration) pays off here — switching back is a flag flip, not a code change.

## Common cutover mistakes

| Mistake | Fix |
|---|---|
| Cutting over Friday afternoon | Pick early in the week so the team can react. |
| No rollback plan | Document it before the cutover. Test it in staging. |
| Decommissioning legacy too quickly | Wait at least 2-4 weeks of stable Stigg-only operation. |
| Forgetting to cancel legacy subs | Customers get double-charged. Schedule cancellations as part of cutover. |
| Communicating cutover internally only | If customer-facing UX changes, customers need a heads-up too. |
| Ignoring revenue reconciliation | Coordinate with Finance — the period spanning cutover is the most painful to reconcile post-hoc. |
| Cutting over with non-zero shadow-read mismatch rate | Mismatches become production incidents. Hit zero (or your defined threshold) first. |
