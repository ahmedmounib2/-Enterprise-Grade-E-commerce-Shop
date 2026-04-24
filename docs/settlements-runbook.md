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

Scheduler netting rule is carry-forward by design: each cycle nets all pending entries where
`createdAt <= periodEnd`. This means pending rows from prior windows are intentionally included
until they are scheduled/paid.

### Settlement lifecycle

1. Scheduler computes next settlement period using `SETTLEMENT_PERIOD_DAYS`.
2. Scheduler checks eligibility using period end + `SETTLEMENT_HOLD_DAYS`.
3. Batch is created in `SettlementBatch`.
4. Per-seller payout rows are created in `SettlementPayout`.
5. Admin executes payout batch:
   - Stripe transfer succeeds => payout marked `paid` and entries marked `paid`.
   - Stripe account missing or invalid amount => payout marked `manual_required`.
   - Stripe transfer API error => payout marked `failed`.
6. Duplicate-safe scheduling outcomes (manual/admin and cron):
   - Manual/admin trigger returns HTTP 200 with `created: false` when no new batch is created.
   - Duplicate race/idempotency outcome is `created: false, reason: 'duplicate'`.
   - Cron logs no-op/duplicate outcomes with `No settlement batch scheduled by cron` plus `reason`
     and `deduplicated` flags.

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
SETTLEMENT_TZ="Africa/Cairo"  # display formatting only; boundaries remain UTC
SETTLEMENT_SCHEDULER_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler
SETTLEMENT_LOCK_TTL_SECONDS=120
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=skip_cycle
```

`SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY` valid values:

- `skip_cycle` (default/safest): skip scheduler cycle when Redis lock backend is unavailable.
- `run_unlocked` (dev/test convenience): continue scheduler cycle without lock when Redis is
  unavailable.

### Production baseline example

```env
SETTLEMENT_PERIOD_DAYS=15
SETTLEMENT_HOLD_DAYS=3
SETTLEMENT_CRON="0 0 * * *"   # daily
SETTLEMENT_TZ="Africa/Cairo"  # display formatting only; boundaries remain UTC
SETTLEMENT_SCHEDULER_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
```

### Local testing / QA example (short cycle)

```env
SETTLEMENT_PERIOD_DAYS=1
SETTLEMENT_HOLD_DAYS=0
SETTLEMENT_CRON="*/2 * * * *" # every 2 min
SETTLEMENT_TZ="Africa/Cairo"  # display formatting only; boundaries remain UTC
SETTLEMENT_SCHEDULER_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
```

Use very short hold periods plus frequent scheduler intervals for local manual tests.

Migration note: `SETTLEMENT_TZ` is display-only metadata for UI consumers. Use
`SETTLEMENT_SCHEDULER_TZ` to control cron execution timezone. Settlement period boundaries remain
UTC.

### Lock configuration by environment

```env
# Development (recommended values)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:dev # Lock key namespace for local dev; avoids collisions with staging/prod
SETTLEMENT_LOCK_TTL_SECONDS=120 # Lock auto-expiry in seconds; enough for local runs while keeping stale locks short
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=run_unlocked # If local Redis is unavailable, still run scheduler for easier development


# Production (safe default, avoid duplicate runs)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler # Redis distributed lock key; must be shared by all prod scheduler instances
SETTLEMENT_LOCK_TTL_SECONDS=300 # Lock expiry in seconds to avoid deadlocks; should cover worst-case cycle duration with buffer
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=skip_cycle # If Redis lock cannot be acquired due to Redis outage, skip this cycle (safest for financial jobs)


# Staging (prod-like behavior, isolated keyspace)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:staging # Same purpose as prod, but namespaced so staging never collides with prod
SETTLEMENT_LOCK_TTL_SECONDS=300 # Keep prod-like TTL to validate behavior under realistic conditions
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=skip_cycle # Match production safety policy during validation


# Testing (unit/integration convenience)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:test # Test-only lock namespace to isolate parallel/local test runs
SETTLEMENT_LOCK_TTL_SECONDS=30 # Short TTL keeps tests fast and reduces stale-lock wait during repeated runs
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=run_unlocked # Allows tests to proceed when Redis is absent/unreliable (except lock-specific test cases)
```

Operational expectations:

- Redis lock is required for strict single-runner behavior across multiple scheduler instances.
- Alert if repeated lock failures appear in logs (`lock acquisition failed` or
  `cycle skipped because lock backend is unavailable`).
- Investigate frequent `lock not acquired` skips (runner contention, TTL too long, stuck workers, or
  topology mismatch).

---

## 5) Reconciliation operations and incident triage

### Scheduled reconciliation job

A dedicated reconciliation cron validates recent settlement integrity:

- Batch checks: `SettlementBatch.totalAmountCents` vs summed payout and ledger totals.
- Payout checks: internal payout state/amount vs Stripe transfer state/amount.
- Scan window: controlled by `SETTLEMENT_RECONCILIATION_SCAN_DAYS`.

Recommended baseline:

```env
SETTLEMENT_RECONCILIATION_ENABLED=true
SETTLEMENT_RECONCILIATION_REPORT_ROLES=admin,staff
SETTLEMENT_RECONCILIATION_CRON=15 2 * * *
SETTLEMENT_RECONCILIATION_TIMEZONE=UTC
SETTLEMENT_RECONCILIATION_SCAN_DAYS=2
```

### COD payout reconciliation job

Purpose:

- scans stale COD refund records that are still `in_progress`
- fetches provider payout status (`CHIMONEY_PAYOUT_STATUS_PATH`)
- marks records completed on terminal success
- marks unresolved items for manual review with explicit failure codes

Required config:

- `COD_PAYOUT_ADAPTER` (`COD_PAYOUT_PROVIDER` is still accepted as fallback alias)
- `CHIMONEY_API_KEY`
- `CHIMONEY_BASE_URL` (use `https://api-sandbox.chimoney.io` for local/dev/testing environments)
- `CHIMONEY_WEBHOOK_SECRET`
- `CHIMONEY_PAYOUT_STATUS_PATH`
- `COD_PAYOUT_RECONCILIATION_ENABLED`
- `COD_PAYOUT_RECONCILIATION_CRON`
- `COD_PAYOUT_RECONCILIATION_TIMEZONE`
- `COD_PAYOUT_RECONCILIATION_AGE_MINUTES`

Auth/header expectations:

- Chimoney requests use `X-API-KEY: <CHIMONEY_API_KEY>`.
- Webhook reconciliation/matching depends on `idempotencyKey` first, with `providerRefundId`
  fallback.

Dev/local recommended values:

```env
COD_PAYOUT_RECONCILIATION_ENABLED=true
COD_PAYOUT_RECONCILIATION_CRON=*/1 * * * *
COD_PAYOUT_RECONCILIATION_TIMEZONE=UTC
COD_PAYOUT_RECONCILIATION_AGE_MINUTES=1
CHIMONEY_PAYOUT_STATUS_PATH=/payouts/{payoutId}
```

Production safe values:

```env
COD_PAYOUT_RECONCILIATION_ENABLED=true
COD_PAYOUT_RECONCILIATION_CRON=*/20 * * * *
COD_PAYOUT_RECONCILIATION_TIMEZONE=UTC
COD_PAYOUT_RECONCILIATION_AGE_MINUTES=45
CHIMONEY_PAYOUT_STATUS_PATH=/payouts/{payoutId}
```

How to test:

1. Seed an order refund record with `provider=cod_payout`, `providerStatus=in_progress`, and stale
   `updatedAt`.
2. Run one reconciliation cycle.
3. Verify status changed to `completed` or `failed` with `reconciliation_*` failure codes.
4. Confirm seller hold transition matches final refund status.

Operational notes:

- COD refund records use fixed category `provider=cod_payout` for payout-provider attempts.
- Failed payout-provider states keep orders in `pending_refund` until webhook/reconciliation/manual
  review resolves the case.
- Provider metadata should remain sanitized (no raw account numbers; no raw webhook body unless
  explicit debug storage is enabled).

### COD refund error-rate alerting and dashboarding

The COD refund workflow emits `cod_refund.execution.failure_by_error_code` with dimensions:

- `provider`
- `countryCode`
- `paymentMethod`
- `errorCode`
- `providerMessage` (best-effort; useful for drill-down)

The Chimoney adapter also emits `cod_payout.provider_error_code.count` with the same dimensions for
provider-side error attribution before workflow persistence.

Recommended alert baselines (rolling window):

- `errorCode=beneficiary_validation_failed`: alert when count >= 5 in 15 minutes (per
  `provider,countryCode`) because this often indicates onboarding/rules drift.
- `errorCode=provider_invalid_request`: alert when count >= 8 in 10 minutes (per
  `provider,paymentMethod`) because this usually means payload mapping regression or provider API
  contract changes.
- `errorCode=country_or_currency_missing`: alert when count >= 3 in 10 minutes (per
  `provider,countryCode`) because this suggests config/data quality issues that will keep failing
  until corrected.
- Any repeated same `errorCode` + `providerMessage` pair >= 10 in 30 minutes should page ops for
  manual intervention and temporary payout gating.

Dashboard panel recommendation (Datadog/Grafana/New Relic equivalent):

- Panel title: `COD Refund Failures by Error Code + Provider Message`
- Query:
  - metric: `cod_refund.execution.failure_by_error_code`
  - filter: `provider:*`
  - group by: `errorCode,providerMessage,provider`
  - visualization: stacked time-series + companion table sorted by 1h count descending
- Add template filters: `provider`, `countryCode`, `paymentMethod` to narrow corridors quickly.

Troubleshooting stuck pending refunds:

1. Check webhook delivery and ensure the event carries at least one matching key (`idempotencyKey`
   or `providerRefundId`).
2. Run/inspect reconciliation output for stale `in_progress` records.
3. If unresolved, keep payout hold active and route to manual review before any new payout attempt.

### Triage workflow for discrepancies

1. Open reconciliation result from admin batch/payout reconcile endpoints.
2. Categorize by discrepancy code (`*_mismatch`, `stripe_transfer_missing`,
   `stripe_transfer_lookup_failed`, etc.).
3. For Stripe lookup failures, confirm transfer id/metadata linkage and API access health.
4. For amount/status mismatches, correlate payout row, ledger rows, and Stripe transfer timeline.
5. For stale `processing` or in-progress payouts, avoid duplicate payout actions; use retry/manual
   actions after confirming current provider state.
6. Document all interventions in ops ticketing with batch/payout IDs and timestamps.

### Payout reversal operator notes

- Endpoint: `POST /api/admin/settlements/payouts/:payoutId/reverse`
- Preconditions: payout exists, has `externalTransferId`, status is `paid`.
- Success effects:
  - payout status updates to `reversed`,
  - `reversalId`/`reversedAt` persisted,
  - negative ledger posting created (`type: payout_reversal`).

Idempotency expectation: already-reversed payouts should return a no-op response without creating
another reversal entry.

---

## 6) Local manual test guidance (fast feedback)

1. Wait for cron (or manually trigger settlement schedule endpoint from admin).
2. Confirm entries move to `scheduled`, with a new `SettlementBatch` + `SettlementPayout` rows.
3. Execute the batch from admin endpoint.
4. Validate one of:
   - `paid` + `externalTransferId` present (Stripe happy path), or
   - `manual_required` export generated when Stripe account prerequisites are absent.

### Admin scheduling API outcomes (created vs skipped vs duplicate)

- **Created (HTTP 201)**

  ```json
  {
    "batch": { "id": "...", "status": "scheduled" },
    "scheduledEntries": 24,
    "sellerCount": 3
  }
  ```

- **Skipped/no-op (HTTP 200)**

  ```json
  {
    "created": false,
    "reason": "period_not_eligible"
  }
  ```

- **Duplicate (HTTP 200)**

  ```json
  {
    "created": false,
    "reason": "duplicate"
  }
  ```

Other common no-op reasons: `no_pending_entries` and `no_positive_payouts`.

---

## 7) Operator fallback runbook (manual payouts + CSV export)

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
