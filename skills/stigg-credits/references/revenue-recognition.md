# Credit Revenue Recognition — Reference

Stigg gives Finance an auditable trail for credit-based revenue. Per grant, you have:

- `amount_granted`, `amount_remaining`, `amount_consumed`
- `effective_at`, `expires_at`
- `category` — paid / promotional
- `cost_basis` — $ per credit, **in full dollars** (Stigg uses dollars, not cents — `0.001` = $0.001/credit; don't multiply by 100)

Plus an **append-only ledger** capturing every burn / expiry / revocation. Use this to compute recognized revenue precisely from actual consumption, while keeping deferred balances correct.

## The math

```text
Deferred revenue   = amount_remaining × cost_basis            (paid grants only)
Recognized revenue = consumed_in_period × cost_basis
Breakage           = expired_in_period × cost_basis           (per your policy)
```

Promotional grants have **zero `cost_basis`** → never affect revenue.

## Journal entries (example policy — align with your accounting rules)

### On purchase / grant of paid credits

```text
Dr  Cash / AR
Cr  Deferred revenue        (for amount_granted × cost_basis)
```

### On consumption

```text
Dr  Deferred revenue
Cr  Revenue                 (for consumed × cost_basis)
```

### On expiry (breakage)

```text
Dr  Deferred revenue
Cr  Revenue                 (for expired × cost_basis, timing per policy)
```

### On revocation / refund

Reverse the appropriate portion of deferred / revenue per your policy.

All supporting facts come from the credit ledger and grants exports.

## Operational pattern

1. **Connect a warehouse.** **Integrations → Data warehouse.** Use the exported `credit_grants` and `credit_ledger` tables.
2. **Model balances by grant.** For each `grant_id`: track `amount_granted`, `amount_remaining`, `amount_consumed`, `expires_at`, `category`, `cost_basis`.
3. **Compute period values.**
   - Recognized revenue (consumption) = `consumed_in_period × cost_basis`
   - Breakage (expiry) = `expired_in_period × cost_basis`
   - Deferred revenue (period end) = `amount_remaining × cost_basis` (paid only)
4. **Translate to journal entries** per the formulas above.
5. **Reconcile.**
   - Tie period consumption (ledger decrements) × `cost_basis` to recognized revenue.
   - Tie ending `remaining × cost_basis` to deferred revenue.
   - Review expired and revoked entries separately for breakage / refunds.

## ERP integration

Exported credits data joins into your ERP (NetSuite, SAP, etc.) via your existing ETL. Each grant's `cost_basis` plus every ledger event provides the evidence trail auditors expect.

Native ERP integrations are on Stigg's roadmap (re-check); until then, use the warehouse export plus your finance integration to post journals and produce revenue schedules.

## Common mistakes

| Mistake | Fix |
|---|---|
| Ignoring `cost_basis` on grants | You can't compute revenue without it. Always include it on exports. |
| Treating promotional grants as revenue | Zero `cost_basis` by design. |
| Recognizing revenue at grant time, not at consumption | That's pre-recognition (cash basis). For accrual / ASC 606, recognize at consumption — Stigg's data supports both, but pick a policy. |
| Using event-level usage data instead of the ledger | Events don't account for grant-priority burn order. The ledger does. |
| Treating breakage (expiry) as automatic revenue | Recognition timing is a policy choice. Some entities recognize at expiry; others, ratably. Coordinate with Finance. |
| Refunds without revocation in Stigg | Books drift. Always pair the refund in your billing system with a `revoke grant` in Stigg. |
