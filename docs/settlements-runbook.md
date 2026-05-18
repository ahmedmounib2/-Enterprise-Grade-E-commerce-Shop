# Settlement & Payout Operations Runbook

This document covers settlement scheduling, payout execution behavior, Stripe Connect prerequisites,
and manual fallback procedures.

---

## 1) Double-entry ledger behavior and settlement lifecycle

The settlement engine aggregates immutable seller ledger entries. Every entry carries an integer
`amountCents`, currency, and source references (order/refund).

| Type                       | Trigger                                                                                         | Merchant-scoped sign                                 | Platform-scoped sign                       | Idempotency-key pattern                                                                                                                               |
| -------------------------- | ----------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sale`                     | Order marked delivered (`recordDeliveryLedger`)                                                 | Positive (seller revenue)                            | —                                          | `delivered:sale:{orderId}:{sellerId}`                                                                                                                 |
| `commission`               | Same delivery event as `sale`                                                                   | Negative (platform fee deducted)                     | —                                          | `delivered:commission:{orderId}:{sellerId}`                                                                                                           |
| `platform_fee`             | Same delivery event as `sale`/`commission`                                                      | —                                                    | Positive (mirrors commission credit)       | `delivered:platform_fee:{orderId}:{sellerId}:platform`                                                                                                |
| `tax`                      | Same delivery event; two rows posted                                                            | Negative (seller tax share)                          | Positive (platform collects)               | `delivered:tax:{orderId}:{sellerId}:seller` / `:platform`                                                                                             |
| `shipping_fee`             | Same delivery event; two rows posted                                                            | Negative (seller shipping share)                     | Positive (platform collects)               | `delivered:shipping_fee:{orderId}:{sellerId}:seller` / `:shipping`                                                                                    |
| `refund`                   | `refund.updated` or `charge.refunded` webhook (`recordRefundLedger`)                            | Negative (revenue reversed)                          | —                                          | `refunded:refund:{orderId}:{sellerId}:{refundKeyPart}`                                                                                                |
| `commission_reversal`      | Same refund event as `refund`                                                                   | Positive (commission returned to seller)             | Negative (platform gives back fee)         | `refunded:commission:{orderId}:{sellerId}:{refundKeyPart}`                                                                                            |
| `tax_refund`               | Same refund event; two rows posted                                                              | Positive (tax refunded to seller)                    | Negative (platform returns tax)            | `refunded:tax_refund:{orderId}:{sellerId}:seller:{refundKeyPart}` / `:platform:{refundKeyPart}`                                                       |
| `shipping_refund`          | Same refund event; two rows posted                                                              | Positive (shipping refunded to seller)               | Negative (platform returns shipping)       | `refunded:shipping_refund:{orderId}:{sellerId}:seller:{refundKeyPart}` / `:shipping:{refundKeyPart}`                                                  |
| `processor_fee`            | Stripe balance transaction resolved (`recordPaymentProcessorFeeLiabilities`)                    | —                                                    | Negative (processor cost to platform)      | `processor_fee:{orderId}`                                                                                                                             |
| `processing_fee`           | Same event as `processor_fee`; per-seller row                                                   | Negative (seller's share of processor cost)          | —                                          | `processing_fee:{orderId}:{sellerId}`                                                                                                                 |
| `processor_fee_recovery`   | Corrective entry when actual processor fee differs from estimate (`recordProcessorFeeRecovery`) | Positive or negative (delta vs estimate, per seller) | Positive or negative (platform-side delta) | `processor_fee_recovery:{orderId}:delta:{deltaKey}` / `:{sellerId}`                                                                                   |
| `cod_refund_disbursement`  | COD refund paid out to customer via provider (Chimoney webhook / reconciliation)                | —                                                    | Negative (platform disburses)              | `refunded:cod_refund_disbursement:{orderId}:{sellerId}:{refundKeyPart}:platform`                                                                      |
| `cod_refund_reimbursement` | Same COD refund event                                                                           | Negative (seller reimburses platform)                | —                                          | `refunded:cod_refund_reimbursement:{orderId}:{sellerId}:{refundKeyPart}:seller`                                                                       |
| `adjustment`               | Manual operator correction                                                                      | Variable                                             | Variable                                   | Caller-supplied; no standard shape                                                                                                                    |
| `payout_reversal`          | Admin reversal endpoint (`POST /api/admin/settlements/payouts/:id/reverse`)                     | Negative (offsets original payout credit)            | —                                          | `payout_reversal:{payoutId}:{reversalId}`                                                                                                             |
| `payout_execution`         | Successful retry execution of a `retryable` payout                                              | Positive (re-credited after reversal)                | —                                          | `settlement:{batchId}:{sellerId}:{currency}:retry:{retrySeq}:{timestamp}`                                                                             |
| `manual_payout`            | Manual payout via `manualSettlementPayout.service.js`                                           | Positive (payout credit)                             | —                                          | `manual_payout:{payoutId}:{transactionId}`                                                                                                            |
| `reserve`                  | Reserve withholding job at settlement cycle                                                     | Negative (amount withheld from net payout)           | —                                          | Caller-supplied or derived from `reserve_release:{sellerId}:{currency}:{amount}:{reason}:{actor}:{reserveMonth}`                                      |
| `reserve_release`          | Reserve release job after hold period expires                                                   | Positive (withheld amount returned)                  | —                                          | Derived from components: `reserve_release:{sellerId}:{currency}:{amount}:{reason}:{actor}:{reserveMonth}:{marker}`                                    |
| `reserve_used`             | Reserve applied against outstanding debt (refund path or release job debt path)                 | Positive (cancels outstanding reserve debit)         | —                                          | `refunded:reserve_used:{orderId}:{sellerId}:{refundKeyPart}` (refund) or `reserve_release_job:reserve_used:debt:{sellerId}:{currency}:{reserveMonth}` |

Every entry carries an integer `amountCents`, currency, and source references (order/refund).
Entries start with `payoutStatus=pending` and move through these phases:

1. **Pending**: eligible for upcoming settlement periods.
2. **Scheduled**: attached to a `SettlementBatch` and frozen for payout execution.
3. **Paid**: finalized after transfer success.
4. **Retryable**: a previously paid payout was reversed and is queued for re-execution with a
   regenerated payout idempotency key.

Scheduler netting rule is carry-forward by design: each cycle nets all pending entries where
`createdAt <= periodEnd`. This means pending rows from prior windows are intentionally included
until they are scheduled/paid.

Each `SettlementPayout` now exposes two complementary fields that break down the gross eligible
amount:

- `inPeriodAmountCents`: the portion of `grossEligibleAmountCents` attributable to ledger entries
  created within the current settlement period.
- `carryForwardAmountCents`: the portion attributable to pending entries from prior periods that
  were carried forward into this payout.

When `carryForwardAmountCents > 0`, the seller financials panel in the dashboard displays an inline
indicator — "Includes $X.XX from prior periods" — alongside the payout total so sellers can
distinguish current-period earnings from accumulated carry-forward amounts.

### Delivery ledger posting

When an order is marked delivered, `recordDeliveryLedger` posts `sale` and `commission` entries for
all sellers in the order. The implementation guarantees atomicity through a single Mongo session:

1. An atomic `findOneAndUpdate` claims the order by setting `deliveryLedgerPostedAt`. Any concurrent
   call for the same order finds the field already set and exits without writing financial rows.
2. Inside the same session, `withSellerPairTransaction` (which accepts an optional `externalSession`
   parameter) writes sale + commission rows for each seller.
3. The session commits only after all seller rows succeed; a failure on any seller row rolls back
   the entire batch.

Prior to this implementation, each seller used its own independent session and
`deliveryLedgerPostedAt` was written outside any transaction. The current approach ensures the claim
flag and all financial rows are always consistent.

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
- `retryable`: paid transfer was reversed and is eligible for a controlled re-execution. This is the
  correct post-reversal status. `reversed` is **not** a valid enum value and was removed from the
  schema; any historical reference to `reversed` in tooling or scripts should be replaced with
  `retryable`.
- `under_review`: payout auto-quarantined by the reconciliation job after detecting a Stripe
  mismatch (see Section 5 — Reconciliation). Payouts in this status are excluded from automated
  retries and require operator investigation before re-enabling.

---

## 3) Stripe Connect prerequisites and onboarding flow

Automatic settlement payout requires seller-level prerequisites:

1. `seller.payout.stripeAccountId` exists.
2. `seller.payout.bank.externalAccountId` exists.
3. Approved payout bank metadata (`seller.payout.bank.bankName`, `last4`, etc.) is present.

Before executing a transfer, `settlementExecution.service.js` performs a Stripe account pre-flight
check that blocks on unresolved requirements in **any** of the following Stripe requirement sets:
`currently_due`, `past_due`, and `eventually_due` (when the `eventually_due` deadline has passed).
If unresolved requirements are found, the payout is automatically marked `manual_required` with a
reason that identifies the blocking requirement set. Operators should resolve any `eventually_due`
items before their deadline to prevent payout interruption.

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

- scans stale COD refund records that are still `in_progress` regardless of their `providerName`
  (not scoped to any single provider)
- fetches provider payout status via the configured adapter
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

### COD refund debt model and transaction boundaries

When a COD refund completes, the seller `payoutHold` flag remains active until the associated ledger
debt is resolved:

- The Chimoney webhook handler (`chimoneyWebhook.service.js`) writes all ledger entries inside the
  same Mongo session as the status updates. The refund record, order status, and ledger rows are
  committed atomically; a partial write cannot leave the system in an inconsistent state.
- The COD reconciliation job (`codPayoutReconciliation.js`) wraps the order update and
  `onOrderStatusChanged` call in a single session for the same reason.
- A seller with `payoutHold: true` is excluded from payout execution and from the reserve release
  job until the hold is lifted (see Section 8 — Reserve withholding & release).

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

### Stripe refund lifecycle

Stripe `refund.updated` events are processed by a dedicated webhook handler:

- **Idempotency key:** `processed_refund_updated_event:{event.id}` stored in Redis. Duplicate event
  deliveries are discarded without re-processing.
- **On `succeeded`:**
  - Marks the refund record `completed`.
  - Updates the order status.
  - Posts refund ledger entries via `recordRefundLedger` (reverses the relevant sale and commission
    rows for the refunded items).
  - Restores reserved stock for full refunds.
- **On `failed`:**
  - Marks the refund record `failed` and updates the order status.
  - No ledger entries are written; the original sale/commission rows remain in place.

If you observe a mismatch between refund record status and order status, replay the `refund.updated`
event from the Stripe dashboard (see _Replay webhook_ section) or correct the refund record status
manually via the admin API.

### Auto-quarantine on reconciliation mismatch

When the reconciliation job detects one of the following discrepancy codes for a `paid` payout, it
automatically transitions that payout to `under_review`:

- `stripe_transfer_missing`: no matching Stripe transfer can be found for the payout's
  `externalTransferId` or metadata.
- `stripe_status_mismatch`: the Stripe transfer status does not align with the internal payout
  state.

Payouts in `under_review` are:

- Excluded from automated retry queues (batch retry and single-payout retry).
- Visible to admins via the standard payout listing with `?status=under_review` filter.

To resolve a quarantined payout, investigate the root cause using the payout reconcile endpoint
(`GET /api/admin/settlements/payouts/:payoutId/reconcile`), fix the underlying issue (missing
transfer, Stripe state drift, etc.), and then manually progress the payout through the appropriate
path (re-execute or escalate to manual payout flow).

### Triage workflow for discrepancies

1. Open reconciliation result from admin batch/payout reconcile endpoints.
2. Categorize by discrepancy code (`*_mismatch`, `stripe_transfer_missing`,
   `stripe_transfer_lookup_failed`, etc.).
3. For payouts automatically moved to `under_review`, use `?status=under_review` to list them;
   investigate and resolve before re-enabling automated retry.
4. For Stripe lookup failures, confirm transfer id/metadata linkage and API access health.
5. For amount/status mismatches, correlate payout row, ledger rows, and Stripe transfer timeline.
6. For stale `processing` or in-progress payouts, avoid duplicate payout actions; use retry/manual
   actions after confirming current provider state.
7. Document all interventions in ops ticketing with batch/payout IDs and timestamps.

### Payout reversal operator notes

- Endpoint: `POST /api/admin/settlements/payouts/:payoutId/reverse`
- Preconditions: payout exists, has `externalTransferId`, status is `paid`.
- Success effects:
  - payout status transitions to `retryable` (not terminal `reversed`),
  - reversal metadata is persisted under `reversalInfo` (`reversalId`, `reversedAt`, actor),
  - payout idempotency key is regenerated to a `...:retry:<n>:<timestamp>` key for the next
    execution attempt,
  - negative ledger posting is upserted (`type: payout_reversal`, key is payout+reversal scoped).

Idempotency expectation: repeating the same reversal request does not create duplicate transitions
and does not create duplicate `payout_reversal` rows. If the payout is already `retryable` for the
same reversal key, the API returns a no-op success payload.

### Retry endpoints (operator usage quick reference)

- **Batch retry** — `POST /api/admin/settlements/payouts/retry`
  - Use when multiple payouts need replay after provider/API incidents or after large reversal
    waves.
  - Eligible statuses: `failed` and `retryable`.
  - Typical transition on success: `failed|retryable` -> `processing` -> `paid` (or
    `manual_required` if prerequisites are missing).

- **Single payout retry (if enabled in your deployment)** —
  `POST /api/admin/settlements/payouts/:payoutId/retry`
  - Use for targeted replay of one seller/payout after validating provider state.
  - Eligible statuses: `failed` and `retryable` (same eligibility as batch retry).
  - Typical transition on success: `failed|retryable` -> `processing` -> `paid` (or
    `manual_required`).

- **When to choose batch vs single**
  - Choose **batch** for queue-level recovery and consistent replay across many payouts.
  - Choose **single** for surgical retries where broad reprocessing is unnecessary or risky.

> **Duplicate-transfer safeguard:** In the retryable execution path, the handler queries Stripe via
> `findTransferByMetadata` (exported from `reconciliation.service.js`) before issuing a new
> transfer. If a matching prior transfer is found for the payout metadata, the payout is marked
> `paid` using the existing transfer ID without creating a second transfer. This makes retry
> execution safe to run even when a previous attempt partially succeeded before crashing.

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

### Ledger atomicity on manual payout execution

When a manual payout is executed via `manualSettlementPayout.service.js`, the following steps run
inside a single Mongo transaction:

1. A `manual_payout` ledger row is created. This row now carries `merchantSellerId`, which allows
   correct attribution when the `sellerId` on the row points to a platform/operator account rather
   than directly to the merchant.
2. The `SettlementPayout` record is updated to `paid`.
3. `LedgerEntry.updateMany({ settlementBatchId, sellerId, payoutStatus: 'scheduled' })` sets
   `payoutStatus: 'paid'` on all underlying `sale`, `commission`, `refund`, and related entries.

All three writes commit together or not at all. If you encounter a `SettlementPayout` with
`status: paid` but underlying entries still showing `payoutStatus: scheduled`, those records
pre-date this fix and can be corrected with a targeted backfill against the affected
`settlementBatchId` and `sellerId`.

### Re-execution ledger semantics (`payout_execution`)

When a `retryable` payout is executed successfully, the system creates a new payout row linked by
`previousPayoutId` and writes an idempotent ledger entry of type `payout_execution`.

Reconciliation math for reversed/re-executed payouts:

- Original payout economic effect: `+P` (already reflected by prior paid settlement state).
- Reversal entry: `-P` (`payout_reversal`).
- Re-execution entry: `+P` (`payout_execution`).
- Net replay-safe effect across reversal + re-execution: `(-P) + (+P) = 0` delta against the
  already-booked baseline, while preserving a complete audit trail.

Operationally, this means you should expect exactly one `payout_reversal` row per reversed payout
idempotency domain and at most one successful `payout_execution` row per successful retry payout
execution attempt key.

---

## 8) Reserve withholding & release

### Sign convention

Reserve withholding produces two ledger row types per settlement cycle:

| Entry type     | `amountCents` sign | Meaning                                         |
| -------------- | ------------------ | ----------------------------------------------- |
| `reserve`      | Negative           | Amount withheld from the seller's net payout.   |
| `reserve_used` | Positive           | Amount applied against the outstanding reserve. |

`reserve_used` is always stored as a **positive** amount so that it nets correctly against the
negative `reserve` row. The release job computes remaining balance as `|reserve| − reserve_used` and
caps each release at that net remaining balance to prevent over-release if a reserve was partially
consumed in a prior cycle.

### Reserve release job behavior

Eligibility for release requires:

- `type='reserve'`
- `payoutStatus='reserved'`
- `releaseScheduledAt <= now`

Any reserve with an elapsed `releaseScheduledAt` is eligible regardless of which calendar month it
originated. The prior month-boundary guard (`reserveMonth != currentMonth`) has been removed; same-
month releases are now correctly included when their hold period has elapsed.

- Sellers with `payoutHold: true` are **skipped** by the release job. Releases resume automatically
  once the hold is lifted (e.g., after a COD debt is resolved — see Section 5, COD refund debt
  model).
- The release amount is capped at the net remaining balance; a release can never exceed what was
  originally withheld minus what has already been returned.
- The job processes at most `RESERVE_RELEASE_BATCH_LIMIT` eligible reserve rows per cycle (default
  `500`). Raise this limit if the release backlog consistently approaches the cap; lower it to
  reduce per-cycle DB load in high-volume environments.

---

## 9) Processor fee recovery

When the actual payment processor fee differs from the estimate posted at order time,
`recordProcessorFeeRecovery` posts corrective ledger entries:

- **Upward correction (actual > estimate):** Posts an additional `processor_fee` debit entry for the
  platform and a `processing_fee` debit for the seller.
- **Downward correction (actual < estimate):** Posts a **negative** `processor_fee` entry for the
  platform and a `processing_fee` reversal for the seller, reducing the previously recorded
  estimate.

Both directions are handled in the same ledger write. The corrective entries ensure the net booked
cost always equals the true processor charge regardless of whether the actual fee came in above or
below the original estimate.
