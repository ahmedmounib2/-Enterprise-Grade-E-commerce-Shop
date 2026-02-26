# Settlement & Payout Operations Runbook

This document covers settlement scheduling, payout execution behavior, Stripe Connect prerequisites,
and manual fallback procedures.

---

## 1) Double-entry ledger behavior and settlement lifecycle

The settlement engine aggregates immutable seller ledger entries:

- `sale`
- `commission`
- `refund`
- `commission_reversal`
- `adjustment`

Every entry carries an integer `amountCents`, currency, and source references (order/refund).
Entries start with `payoutStatus=pending` and move through these phases:

1. **Pending**: eligible for upcoming settlement periods.
2. **Scheduled**: attached to a `SettlementBatch` and frozen for payout execution.
3. **Paid**: finalized after transfer success.

### Settlement lifecycle

1. Scheduler computes next settlement period using `SETTLEMENT_PERIOD_DAYS`.
2. Scheduler checks eligibility using period end + `SETTLEMENT_HOLD_DAYS`.
3. Batch is created in `SettlementBatch`.
4. Per-seller payout rows are created in `SettlementPayout`.
5. Admin executes payout batch:
   - Stripe transfer succeeds => payout marked `paid` and entries marked `paid`.
   - Stripe account missing or invalid amount => payout marked `manual_required`.
   - Stripe transfer API error => payout marked `failed`.

---

## 2) Status references

### Settlement batch statuses (`SettlementBatch.status`)

- `calculated`: pre-scheduled or informational calculation state.
- `scheduled`: payouts queued and pending execution.
- `paid`: all payouts terminally paid.
- `failed`: at least one payout failed and requires operator attention.

### Settlement payout statuses (`SettlementPayout.status`)

- `scheduled`: queued for execution.
- `paid`: transfer succeeded (`externalTransferId` populated).
- `failed`: transfer attempt failed.
- `manual_required`: auto payout blocked; manual operator flow required.

---

## 3) Stripe Connect prerequisites and onboarding flow

Automatic settlement payout requires seller-level prerequisites:

1. `seller.payout.stripeAccountId` exists.
2. `seller.payout.bank.externalAccountId` exists.
3. Approved payout bank metadata (`seller.payout.bank.bankName`, `last4`, etc.) is present.

### Account and external-account flow

1. During payout method onboarding, backend checks for `stripeAccountId`.
2. If missing, backend creates a Stripe Connect Custom account for the seller.
3. Backend attaches submitted `bankToken` via Stripe external account APIs.
4. When bank info is updated, previous external account is replaced where possible.
5. If payout execution finds no Stripe account, payout is automatically marked `manual_required`
   with reason `missing_stripe_account_id`.

---

## 4) Environment variables and scheduling presets

### Settlement env vars

```env
SETTLEMENT_PERIOD_DAYS=15
SETTLEMENT_HOLD_DAYS=3
SETTLEMENT_CRON="0 0 * * *"
SETTLEMENT_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
```

### Production baseline example

```env
SETTLEMENT_PERIOD_DAYS=15
SETTLEMENT_HOLD_DAYS=3
SETTLEMENT_CRON="0 0 * * *"   # daily
SETTLEMENT_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
```

### Local testing / QA example (short cycle)

```env
SETTLEMENT_PERIOD_DAYS=1
SETTLEMENT_HOLD_DAYS=0
SETTLEMENT_CRON="*/2 * * * *" # every 2 min
SETTLEMENT_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
```

Use very short hold periods plus frequent scheduler intervals for local manual tests.

---

## 5) Local manual test guidance (fast feedback)

1. Set `SETTLEMENT_PERIOD_DAYS=1`, `SETTLEMENT_HOLD_DAYS=0`, and a frequent cron (`*/1` or `*/2`).
2. Create test orders and confirm ledger entries are generated with `payoutStatus=pending`.
3. Wait for cron (or manually trigger settlement schedule endpoint from admin).
4. Confirm entries move to `scheduled`, with a new `SettlementBatch` + `SettlementPayout` rows.
5. Execute the batch from admin endpoint.
6. Validate one of:
   - `paid` + `externalTransferId` present (Stripe happy path), or
   - `manual_required` export generated when Stripe account prerequisites are absent.

---

## 6) Operator fallback runbook (manual payouts + CSV export)

When `manual_required` payouts exist, follow this short procedure:

1. Execute settlement batch endpoint.
2. Inspect response `manualExport` payload (columns + rows + CSV string).
3. Save CSV to your secure operator workspace (do **not** store in source control).
4. Perform payout manually through approved banking/treasury process.
5. Record payout evidence in internal ticketing (batch ID, seller ID, amount, transaction ref).
6. Reconcile affected sellers by updating payout records per internal SOP (or re-run batch actions
   after prerequisites are fixed).

### CSV export fields

- `batchId`
- `periodStart`
- `periodEnd`
- `sellerId`
- `payoutId`
- `amountCents`
- `currency`
- `reason`
- `bankName`
- `last4`
- `accountHolder`

### Common manual-required reasons

- `missing_stripe_account_id`
- `non_positive_amount`
