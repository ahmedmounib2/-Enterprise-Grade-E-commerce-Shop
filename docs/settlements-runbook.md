# Settlement & Payout Operations Runbook

This document covers settlement scheduling, payout execution behavior, Stripe Connect prerequisites,
and manual fallback procedures.

---

## 1) Double-entry ledger behavior and settlement lifecycle

The settlement engine aggregates immutable seller ledger entries. Every entry carries an integer
`amountCents`, currency, and source references (order/refund).

| Type                       | Trigger                                                                                                                                                        | Merchant-scoped sign                                                                             | Platform-scoped sign                                                | Idempotency-key pattern                                                                                                                                                                                                                       |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sale`                     | Order marked delivered (`recordDeliveryLedger`)                                                                                                                | Positive (seller revenue)                                                                        | —                                                                   | `delivered:sale:{orderId}:{sellerId}`                                                                                                                                                                                                         |
| `commission`               | Same delivery event as `sale`                                                                                                                                  | Negative (platform fee deducted)                                                                 | —                                                                   | `delivered:commission:{orderId}:{sellerId}`                                                                                                                                                                                                   |
| `platform_fee`             | Same delivery event as `sale`/`commission`                                                                                                                     | —                                                                                                | Positive (mirrors commission credit)                                | `delivered:platform_fee:{orderId}:{sellerId}:platform`                                                                                                                                                                                        |
| `tax`                      | Same delivery event; two rows posted                                                                                                                           | Negative (seller tax share)                                                                      | Positive (platform collects)                                        | `delivered:tax:{orderId}:{sellerId}:seller` / `:platform`                                                                                                                                                                                     |
| `shipping_fee`             | Same delivery event; two rows posted                                                                                                                           | Negative (seller shipping share)                                                                 | Positive (platform collects)                                        | `delivered:shipping_fee:{orderId}:{sellerId}:seller` / `:shipping`                                                                                                                                                                            |
| `refund`                   | `refund.updated` or `charge.refunded` webhook (`recordRefundLedger`)                                                                                           | Negative (revenue reversed)                                                                      | —                                                                   | `refunded:refund:{orderId}:{sellerId}:{refundKeyPart}`                                                                                                                                                                                        |
| `commission_reversal`      | Same refund event as `refund`                                                                                                                                  | Positive (commission returned to seller)                                                         | Negative (platform gives back fee)                                  | `refunded:commission:{orderId}:{sellerId}:{refundKeyPart}`                                                                                                                                                                                    |
| `tax_refund`               | Same refund event; two rows posted (**prepaid only** — COD refunds post no tax rows)                                                                           | Positive (tax refunded to seller)                                                                | Negative (platform returns tax)                                     | `refunded:tax_refund:{orderId}:{sellerId}:seller:{refundKeyPart}` / `:platform:{refundKeyPart}`                                                                                                                                               |
| `tax_remittance`           | Admin records a tax payment to a tax authority (`POST /api/admin/platform-financials/tax-remittances`); gated by `FEATURE_TAX_REMITTANCE`                      | —                                                                                                | Negative (platform pays tax authority; debits `PLATFORM_SELLER_ID`) | `tax_remittance:{taxRemittanceId}`                                                                                                                                                                                                            |
| `shipping_refund`          | Same refund event; two rows posted (**prepaid only** — COD refunds post no shipping rows)                                                                      | Positive (shipping refunded to seller)                                                           | Negative (platform returns shipping)                                | `refunded:shipping_refund:{orderId}:{sellerId}:seller:{refundKeyPart}` / `:shipping:{refundKeyPart}`                                                                                                                                          |
| `shipping_label_cost`      | Carrier label purchased (`POST /seller/orders/:id/shipments` with `source:'api'`); gated by `FEATURE_SHIPPING_LABEL_LEDGER`                                    | —                                                                                                | Negative (carrier label cost; debits `SHIPPING_SELLER_ID`)          | `shipping_label_cost:{shipmentId}`                                                                                                                                                                                                            |
| `shipping_revenue`         | Same delivery event; flat/tiered shipping (`order.shippingMode !== 'carrier'`) when `FEATURE_SELLER_SHIPPING_PAYOUT=true`                                      | Positive (seller keeps shipping as revenue)                                                      | —                                                                   | `delivered:shipping_revenue:{orderId}:{sellerId}`                                                                                                                                                                                             |
| `shipping_revenue_refund`  | Same refund event; reverses `shipping_revenue`                                                                                                                 | Negative (seller revenue reversed on refund)                                                     | —                                                                   | `refunded:shipping_revenue_refund:{orderId}:{sellerId}:{refundKeyPart}`                                                                                                                                                                       |
| `processor_fee`            | Stripe balance transaction resolved (`recordPaymentProcessorFeeLiabilities`)                                                                                   | —                                                                                                | Negative (processor cost to platform)                               | `processor_fee:{orderId}`                                                                                                                                                                                                                     |
| `processing_fee`           | Same event as `processor_fee`; per-seller row                                                                                                                  | Negative (seller's share of processor cost)                                                      | —                                                                   | `processing_fee:{orderId}:{sellerId}`                                                                                                                                                                                                         |
| `processor_fee_recovery`   | Corrective entry when actual processor fee differs from estimate (`recordProcessorFeeRecovery`)                                                                | Positive or negative (delta vs estimate, per seller)                                             | Positive or negative (platform-side delta)                          | `processor_fee_recovery:{orderId}:delta:{deltaKey}` / `:{sellerId}`                                                                                                                                                                           |
| `cod_refund_disbursement`  | COD refund paid out to customer via provider (Chimoney webhook / reconciliation) — amount = full refund (merch + tax + shipping)                               | —                                                                                                | Negative (platform disburses the full amount)                       | `refunded:cod_refund_disbursement:{orderId}:{sellerId}:{refundKeyPart}:platform`                                                                                                                                                              |
| `cod_refund_reimbursement` | Same COD refund event — amount = `refundAmountCents + taxRefundAmountCents + shippingRefundAmountCents`. COD refunds post NO tax/shipping rows (see Section 5) | Negative (seller reimburses platform for the full customer refund: merchandise + tax + shipping) | —                                                                   | `refunded:cod_refund_reimbursement:{orderId}:{sellerId}:{refundKeyPart}:seller`                                                                                                                                                               |
| `dispute_fee`              | LOST Stripe chargeback (`charge.dispute.closed`) — fee from the event's `balance_transactions`, fallback `DISPUTE_FEE_CENTS`                                   | —                                                                                                | Negative (platform pays Stripe's dispute fee)                       | `dispute_fee:{disputeId}`                                                                                                                                                                                                                     |
| `adjustment`               | Manual operator correction                                                                                                                                     | Variable                                                                                         | Variable                                                            | Caller-supplied; no standard shape                                                                                                                                                                                                            |
| `payout_reversal`          | Admin reversal endpoint (`POST /api/admin/settlements/payouts/:id/reverse`)                                                                                    | Negative (offsets original payout credit)                                                        | —                                                                   | `payout_reversal:{payoutId}:{reversalId}`                                                                                                                                                                                                     |
| `payout_execution`         | Successful retry execution of a `retryable` payout                                                                                                             | Positive (re-credited after reversal)                                                            | —                                                                   | `settlement:{batchId}:{sellerId}:{currency}:retry:{retrySeq}:{timestamp}`                                                                                                                                                                     |
| `manual_payout`            | Manual payout via `manualSettlementPayout.service.js`                                                                                                          | Positive (payout credit)                                                                         | —                                                                   | `manual_payout:{payoutId}:{transactionId}`                                                                                                                                                                                                    |
| `reserve`                  | Reserve withholding job at settlement cycle                                                                                                                    | Negative (amount withheld from net payout)                                                       | —                                                                   | Caller-supplied or derived from `reserve_release:{sellerId}:{currency}:{amount}:{reason}:{actor}:{reserveMonth}`                                                                                                                              |
| `reserve_release`          | Release job after hold period expires, or refund-time reserve application (`applied_to_refund`). **Payout-eligible**: swept into the next settlement batch     | Positive (withheld amount returned to payable)                                                   | —                                                                   | Job: `reserve_release:{sellerId}:{currency}:{reserveMonth}:{reason}:{marker}:{rowsHash}` — identifies the operation (the exact rows released), never the amount. Refund path: `refunded:reserve_release:{orderId}:{sellerId}:{refundKeyPart}` |
| `reserve_used`             | Reserve consumed from the reserved pool at refund-time application (audit trail; never settles)                                                                | Positive (cancels outstanding reserve debit)                                                     | —                                                                   | `refunded:reserve_used:{orderId}:{sellerId}:{refundKeyPart}`                                                                                                                                                                                  |

Every entry carries an integer `amountCents`, currency, and source references (order/refund).
Entries start with `payoutStatus=pending` and move through these phases:

1. **Pending**: eligible for upcoming settlement periods.
2. **Scheduled**: attached to a `SettlementBatch` and frozen for payout execution.
3. **Paid**: finalized after transfer success.
4. **Retryable**: a previously paid payout was reversed and is queued for re-execution with a
   regenerated payout idempotency key.

This pending → scheduled → paid → retryable lifecycle applies to **payout-eligible** types only. The
**pass-through** types (`tax`, `tax_refund`, `tax_remittance`, `shipping_fee`, `shipping_refund`,
`shipping_label_cost` — `PASS_THROUGH_TYPES` in `settlementClassification.js`) are collected on the
platform's behalf and never settled to the seller, so seller- and admin-facing ledger views
(`normalizeLedgerEntry`) report their `payoutStatus` as `'collected'` regardless of the value stored
on the row, and `SellerFinancialsPanel` shows a dedicated "Collected" badge instead of "Pending
payout" / "Scheduled" / "Paid".

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
   - Overlapping-period guard (M-2): the batch unique index includes `configHash`, so a
     `SETTLEMENT_PERIOD_DAYS`/`SETTLEMENT_HOLD_DAYS` change could otherwise create a second batch
     over an already-settled calendar range. Before creating a batch, the scheduler checks for ANY
     existing batch (same currency, configHash ignored) whose period overlaps the computed one and
     skips the cycle with `created: false, reason: 'overlapping_period'` plus a warning log. After
     a config change, wait until the new config's next computed window clears the last settled
     period (or reverse the conflicting batches) before expecting a new batch.

---

## 2) Status references

### Settlement batch statuses (`SettlementBatch.status`)

- `calculated`: pre-scheduled or informational calculation state.
- `scheduled`: payouts queued and pending execution.
- `paid`: all payouts terminally paid.
- `failed`: at least one payout failed and requires operator attention.

### Settlement payout statuses (`SettlementPayout.status`)

- `scheduled`: queued for execution. **Aged-scheduled expiry:** a payout still `scheduled` after
  `SETTLEMENT_PAYOUT_EXPIRE_SCHEDULED_HOURS` (default 72) is handled by the payout-retry job —
  executed immediately when `SETTLEMENT_AUTO_EXECUTE=true`, otherwise escalated to `manual_required`
  (reason `aged_scheduled_payout`) with an alert. No payout can sit in `scheduled` indefinitely.
- `processing`: claimed by a worker mid-execution. A payout stuck here longer than
  `SETTLEMENT_PAYOUT_STALE_PROCESSING_MINUTES` (crashed worker) is resolved by the payout-retry
  job's **stale-processing reaper**: a Stripe transfer matching the payout metadata exists →
  finalized `paid` (methodSnapshot notes `finalizedVia: stale_processing_reaper`); none → released
  to `failed` (reason `stale_processing_reaped`) for normal retry. The reconciliation scan also
  reports stale rows as `stale_processing` in case the reaper is disabled.
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
SETTLEMENT_PAYOUT_STALE_PROCESSING_MINUTES=60  # reaper window for crashed `processing` claims (dev: 5)
SETTLEMENT_EXECUTION_NEGATIVE_DELTA_CENTS=0    # M8 preflight; 0 = any post-scheduling negative activity blocks
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

### Distributed lock keys and RUN_ON_BOOT behavior (all cron jobs)

All cron jobs in the settlement and payout subsystem acquire a Redis distributed lock before
executing so that concurrent instances in a multi-replica deployment do not run the same job
simultaneously. Each job uses a distinct lock key:

| Job                       | Lock key                         | Default TTL (seconds) | RUN_ON_BOOT env var                                           |
| ------------------------- | -------------------------------- | --------------------- | ------------------------------------------------------------- |
| Settlement scheduler      | `lock:settlement_scheduler`      | 120                   | _(scheduler is boot-triggered via `SETTLEMENT_AUTO_EXECUTE`)_ |
| Settlement reconciliation | `lock:settlement_reconciliation` | 120                   | `SETTLEMENT_RECONCILIATION_RUN_ON_BOOT`                       |
| Payout retry              | `lock:settlement_payout_retry`   | 120                   | `PAYOUT_RETRY_RUN_ON_BOOT`                                    |
| COD payout reconciliation | `lock:cod_payout_reconciliation` | 120                   | `COD_PAYOUT_RECONCILIATION_RUN_ON_BOOT`                       |
| Subscription renewal      | `lock:subscription_renewal`      | 120                   | `SUBSCRIPTION_RENEWAL_RUN_ON_BOOT`                            |
| Liability reconciliation  | `liability:reconciliation:lock`  | 1800                  | `LIABILITY_RECONCILIATION_RUN_ON_BOOT`                        |

**RUN_ON_BOOT semantics:** each `*_RUN_ON_BOOT` variable defaults to `false`. When `false`, the job
only runs on its configured cron schedule — safe for multi-replica deployments where process startup
is not synchronized. Set to `true` only in single-instance environments (e.g., local dev or one-off
migration deploys) where you need an immediate first run on server start.

All jobs use `SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY` semantics: `skip_cycle` (default/production) or
`run_unlocked` (dev/test convenience). Keep `skip_cycle` in production for all financial jobs.

---

### Scheduled reconciliation job

A dedicated reconciliation cron validates recent settlement integrity:

- Batch checks: `SettlementBatch.totalAmountCents` vs summed payout and ledger totals.
- Payout checks: internal payout state/amount vs Stripe transfer state/amount.
- Stale-processing checks: `processing` payouts older than
  `SETTLEMENT_PAYOUT_STALE_PROCESSING_MINUTES` are reported with discrepancy code `stale_processing`
  (report-only — no Stripe comparison and no quarantine; the payout-retry reaper owns the resolution
  path).
- Scan window: controlled by `SETTLEMENT_RECONCILIATION_SCAN_DAYS`.

Recommended baseline:

```env
SETTLEMENT_RECONCILIATION_ENABLED=true
SETTLEMENT_RECONCILIATION_REPORT_ROLES=admin,staff
SETTLEMENT_RECONCILIATION_CRON=15 2 * * *
SETTLEMENT_RECONCILIATION_TIMEZONE=UTC
SETTLEMENT_RECONCILIATION_SCAN_DAYS=2
SETTLEMENT_RECONCILIATION_MAX_REPORT_RECIPIENTS=5  # caps the email recipient list (default 5)
SETTLEMENT_RECONCILIATION_RUN_ON_BOOT=false         # set true only on single-instance deploys
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

**COD refund accounting (two rows, full amounts).** The platform never holds COD tax or shipping —
the seller collected the full order amount in cash at the door. A COD refund therefore posts exactly
two rows per seller, both sized at the **full** refund
(`refundAmountCents + taxRefundAmountCents + shippingRefundAmountCents`):

- `cod_refund_disbursement` (operator/platform side, negative) — the platform's actual Chimoney
  outflow to the customer.
- `cod_refund_reimbursement` (merchant side, negative) — the full clawback from the seller, who
  keeps the original cash.

No `tax_refund` or `shipping_refund` rows are posted for COD refunds: the tax/shipping liability
pools never received COD money, so debiting them corrupted liability reconciliation, and the former
seller-side pass-through credits never settled (they only produced misleading "collected" rows in
seller statements). To verify a COD refund, confirm both rows' `|amountCents|` equal the
`RefundRecord`'s `totalRefundCents` and that the seller's payout-eligible balance moved by exactly
`-totalRefundCents`.

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
- `post_scheduling_negative_activity` — the execution preflight found more than
  `SETTLEMENT_EXECUTION_NEGATIVE_DELTA_CENTS` of new negative pending ledger activity
  (refunds/reversals) for the seller created after the batch. Review the seller's recent refund
  entries; either let the next cycle net everything (cancel this payout via reversal-free
  re-schedule) or approve manually for the amount you judge correct.

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

### Model: reserve is a deferral of payout-eligible value

The reserve is withheld by reducing the payout amount at batch creation, and **returned as a
payout-eligible `reserve_release` credit** (`payoutStatus: 'pending'`) that the next settlement
batch sweeps into `netPayable` like any other entry. The reserved-scope rows (`reserve`,
`reserve_used`, `payoutStatus: 'reserved'`) are the audit trail that tracks the reserved pool — they
never settle directly.

| Entry type        | `amountCents` sign | Scope             | Meaning                                                                                                                                                                                                  |
| ----------------- | ------------------ | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reserve`         | Negative           | reserved→released | Amount withheld from the seller's net payout. Flipped to the terminal `payoutStatus: 'released'` (with `releasedAt`) when its value is returned — a released row can never be counted or released again. |
| `reserve_used`    | Positive           | reserved          | Amount consumed from the reserved pool (audit trail).                                                                                                                                                    |
| `reserve_release` | Positive           | pending           | Payout-eligible credit; swept by the next settlement.                                                                                                                                                    |

`reserve_used` is always stored as a **positive** amount so that it nets correctly against the
negative `reserve` row. Both release paths compute the remaining pool as `|reserve| − reserve_used`
and cap the release at that net balance, so a release can never exceed what was withheld minus what
was already returned.

### Two release paths

1. **Scheduled release (cron job):** posts `reserve_release` (reason `scheduled_monthly_release`)
   once `releaseScheduledAt` elapses. Sellers with a **negative pending balance still get the
   release** — because the credit is payout-eligible, the next batch nets it against the debt
   automatically. (The former "debt branch", which posted a second `reserve_used` instead of
   releasing, was removed: it consumed the reserve without ever crediting the seller — a double
   charge.)
2. **Refund-time application (`applyReserveToRefund`):** an early release. In the same seller
   transaction as the refund reversal entries, it posts `reserve_used` (+applied, reserved — shrinks
   the pool) AND `reserve_release` (+applied, pending, reason `applied_to_refund`, idempotency key
   `refunded:reserve_release:{orderId}:{sellerId}:{refundKeyPart}`). The refund debit and this
   credit net in the next batch, so the seller is charged exactly once, funded by the reserve.

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
- The job processes at most `RESERVE_RELEASE_BATCH_LIMIT` eligible reserve rows per cycle (default
  `500`). Raise this limit if the release backlog consistently approaches the cap; lower it to
  reduce per-cycle DB load in high-volume environments.
- **Terminal flip:** after posting the `reserve_release` credit, the job flips the released
  `reserve` rows to `payoutStatus: 'released'` (+`releasedAt`). This is what prevents re-release: a
  released row leaves the eligibility set, the reserved-pool aggregates, and the admin reserve
  summary permanently. Crash ordering is safe — the credit is posted first under an
  operation-identity idempotency key
  (`reserve_release:{sellerId}:{currency}:{reserveMonth}:{reason}:{marker}:{rowsHash}`, where
  `rowsHash` hashes the exact row IDs being released), so a retry over the same rows dedupes instead
  of double-crediting, then completes the flip. The key deliberately excludes the amount: keys
  identify operations, not payloads.
- If a month group's remaining pool is zero (fully consumed by `reserve_used`), the rows are flipped
  to `released` with **no** credit — their value already funded refunds, and flipping stops them
  from retrying forever.

**Verify a release reached the seller:** after the next scheduler run, the `reserve_release` row
should carry `payoutStatus: 'scheduled'` (then `'paid'` after execution) and its amount should be
included in that batch's `grossEligibleAmountCents` for the seller.

---

## 9) Payout and refund notifications

When `FEATURE_IN_APP_NOTIFICATIONS=true`, settlement and refund operations fire in-app notifications
**in addition to** the transactional emails described in the main README.

| Event                             | Email function                                       | Notification type  | Recipient         |
| --------------------------------- | ---------------------------------------------------- | ------------------ | ----------------- |
| Settlement payout marked `paid`   | `sendPayoutCompletedEmail`                           | `payout_completed` | Seller            |
| Settlement payout marked `failed` | `sendPayoutFailedEmail`                              | `payout_failed`    | Seller            |
| Refund approved                   | `sendRefundApprovedCustomerEmail` / `...SellerEmail` | `refund_approved`  | Customer + Seller |
| Refund rejected                   | `sendRefundRejectedCustomerEmail` / `...SellerEmail` | `refund_rejected`  | Customer + Seller |

Both the email and the notification are fire-and-forget: a failure in one never suppresses the
other. Notification delivery is handled by `fireNotification()` in
`backend/src/services/notification/notification.service.js`.

---

## 10) Processor fee recovery

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

---

## 11) Shipping liability reconciliation

The platform holds collected shipping fees in the Shipping Liability pool (`SHIPPING_SELLER_ID`).
This pool should net to zero (or a small positive float) when carrier label costs are properly
recorded. Verify the pool balance with:

> **`FEATURE_SELLER_SHIPPING_PAYOUT` (default off):** when enabled, only **carrier**-mode shipping
> flows into this liability pool. **Flat/tiered** (self-fulfilled) shipping is instead posted as
> payout-eligible seller revenue (`shipping_revenue` / `shipping_revenue_refund`) and never touches
> `SHIPPING_SELLER_ID`. The flag is **forward-only** — it changes how deliveries are posted while it
> is on; historical flat/tiered rows already in the pool are left untouched (no backfill).

> **Per-region classification.** `order.shippingMode` is the _effective_ mode for that order's
> destination (`resolveEffectiveShippingMode`): a seller can set `shipping.regions[].mode` to
> `carrier` for some regions and `flat`/`tiered` for others (or leave a region to inherit the
> store-level mode). The carrier-vs-flat/tiered split above is therefore evaluated **per order, by
> destination** — not per store. A single seller can have some deliveries post to the Shipping
> Liability pool (carrier-region orders) and others post as `shipping_revenue` (flat/tiered-region
> orders, when `FEATURE_SELLER_SHIPPING_PAYOUT=true`) within the same period.

```
# 1. Query the shipping liability seller's ledger entries for a period
GET /api/admin/platform-financials/ledger
  ?sellerId=<SHIPPING_SELLER_ID>&type=shipping_fee,shipping_refund,shipping_label_cost
  &from=<YYYY-MM-01>&to=<YYYY-MM-01 next month>
```

**Expected signs:**

| Type                  | Sign     | Meaning                              |
| --------------------- | -------- | ------------------------------------ |
| `shipping_fee`        | Positive | Collected from customers on delivery |
| `shipping_refund`     | Negative | Returned to customers on refund      |
| `shipping_label_cost` | Negative | Paid to Shippo for carrier labels    |

**Net formula:** `shipping_fee + shipping_refund + shipping_label_cost` (sum of signed values).

> **Carrier-only.** With `FEATURE_SELLER_SHIPPING_PAYOUT` on, flat/tiered shipping is **not** in
> this net — it is posted as `shipping_revenue` / `shipping_revenue_refund` (seller revenue) and
> surfaced separately on the snapshot as `sellerRetainedShippingCents`
> (`shippingRevenueCents + shippingRevenueRefundCents`). It is reported for visibility only and
> never affects the carrier-liability `status`. Both `marketplaceMetrics.shippingLiabilityCents`
> (this carrier net) and `marketplaceMetrics.sellerRetainedShippingCents` are exposed by
> `/api/admin/platform-financials` — never blend the two.

A **positive net** means the platform collected more in shipping fees than it paid in labels —
reasonable if the store's shipping prices exceed carrier cost. A **negative net** means outflows
exceeded collections (e.g. labels were purchased for free/discounted shipping orders). The monthly
`ReconciliationSnapshot` sets `status: 'review'` and logs a warning in this case.

**Procedure when net is negative:**

1. Query `ReconciliationSnapshot.findOne({ period, currency, status: 'review' })`.
2. Identify whether the shortfall is from missing `shipping_label_cost` entries (e.g.
   `FEATURE_SHIPPING_LABEL_LEDGER` was off when shipments were created), excess refunds, or
   underpriced shipping.
3. For missing historical entries, perform a backfill by calling `recordShippingLabelExpense` for
   each affected shipment with `labelCostCents > 0` and `labelCostSource === 'api'`.
4. Re-run reconciliation (`LIABILITY_RECONCILIATION_PERIOD=<period>`) after the backfill.

---

## 12) Tax remittance procedure

When the platform pays collected tax to a tax authority, record the remittance via the admin panel
or API before the next reconciliation run.

**Admin panel:** Navigate to Admin → Marketplace Financials → Tax remittances and click **Record tax
remittance**. The modal collects Period, Jurisdiction, Amount, Currency, Date filed, and Notes —
fill in the period, amount, and currency, and optionally the jurisdiction, a backdated date filed,
and filing notes. On success the document is filed and the ledger entry is posted automatically; the
modal closes, a toast confirms the result, and the remittances table and liability totals refresh.

**API:**

```
POST /api/admin/platform-financials/tax-remittances
Authorization: admin/staff session cookie

{
  "period":       "2026-05",
  "amountCents":  42500,
  "currency":     "USD",
  "jurisdiction": "US-Federal",
  "notes":        "Q1 2026 quarterly payment — EIN 12-3456789"
}
```

The response returns the `TaxRemittance` document with `status: 'filed'` and the `ledgerEntryId` of
the posted `tax_remittance` debit.

> **`FEATURE_TAX_LIABILITY_SELLER` (default off):** when enabled,
> `resolveTaxLiabilityLedgerSellerId()` posts the operator-side mirror of every `tax`, `tax_refund`,
> and `tax_remittance` entry — including this remittance debit — against the dedicated Tax-Liability
> seller (`TAX_SELLER_ID`, slug `tax-liability`) instead of `PLATFORM_SELLER_ID`. The flag is
> **forward-only** — it changes where new entries post while it is on; historical
> `tax`/`tax_refund`/`tax_remittance` rows already against `PLATFORM_SELLER_ID` are left untouched
> (no backfill). See `docs/tax-book.md` for the full cutover and reconciliation impact.

**Verify:** After recording, query the tax liability pool to confirm the debit landed. Use
`TAX_SELLER_ID` if `FEATURE_TAX_LIABILITY_SELLER=true`, otherwise `PLATFORM_SELLER_ID`:

```
GET /api/admin/platform-financials/ledger
  ?sellerId=<TAX_SELLER_ID or PLATFORM_SELLER_ID>&type=tax,tax_refund,tax_remittance
  &from=<period start>&to=<period end>
```

The `taxNetLiabilityCents` in the next `ReconciliationSnapshot` for that period should decrease by
`amountCents`.

**Rollback:** If a remittance was entered with an incorrect amount, record a corrective entry in the
opposite direction using the `adjustment` endpoint (`POST /api/admin/ledger/adjustments`) against
the same seller the remittance was posted to (`TAX_SELLER_ID` if
`FEATURE_TAX_LIABILITY_SELLER=true`, otherwise `PLATFORM_SELLER_ID`), then create a new remittance
with the correct amount. Do not delete the original `TaxRemittance` document — it is the audit
trail.

**Feature gate:** Both the admin UI panel and the API endpoints return `404` when
`FEATURE_TAX_REMITTANCE=false`. Enable in `backend/.env` before using.

---

## 13) Chargeback (dispute) accounting

When Stripe closes a chargeback (`charge.dispute.closed`), the funds have already moved — a LOST
dispute pulled the disputed amount plus the dispute fee from the platform balance. The webhook
records that movement (audit H6):

- **Lost:** under the order's refund lock, a `provider: 'stripe_dispute'` completed refund record is
  pushed (clamped by the normal refund plan so prior refunds are respected), `refundedAmountCents`
  advances (future refund plans clamp against the chargeback), the standard refund reversal entries
  are posted with identity `dispute:{disputeId}`, and a `dispute_fee` entry (platform scope,
  `dispute_fee:{disputeId}`) records the fee — from the event's `balance_transactions`, falling back
  to `DISPUTE_FEE_CENTS` (default 1500). **No restock** — the customer keeps the goods. Replays are
  idempotent: the Redis event dedup, the existing `stripe_dispute` record check under the lock, and
  the ledger idempotency keys each stop a double-posting.
- **Won:** dispute status update only. (If your Stripe account is charged a fee even on won
  disputes, record it manually via the adjustment endpoint — automatic fee-on-won posting is a known
  follow-up.)
- **No resolvable order** (no internal dispute link and no PaymentIntent match): the ledger is NOT
  posted and an `alert: true` error is logged — reconcile manually against the Stripe dashboard.

---

## 14) Seller debt monitoring (negative balances)

Carry-forward netting recovers refund/chargeback debt automatically when a seller keeps selling, but
a seller who stops selling never repays through settlements. The `seller_debt_monitor` cron (default
daily 06:00 UTC, Redis lock `lock:seller_debt_monitor`) is the detection layer:

- **Report:** sellers whose unpaid payout-eligible balance is below
  `−SELLER_DEBT_ALERT_THRESHOLD_CENTS` (default 5000) with the oldest unpaid negative entry at least
  `SELLER_DEBT_ALERT_AGE_DAYS` (default 7) old are logged with `alert: true` and emailed to the
  reconciliation report recipients.
- **Auto-hold:** below `−SELLER_DEBT_AUTO_HOLD_THRESHOLD_CENTS` (default 20000) the job sets
  `payoutHold` with reason `negative_balance_auto_hold` (history entry, `adminId: null`). It never
  overwrites an existing hold.
- **Release:** `maybeReleaseSellerPayoutHold` clears a hold when the balance recovers — but ONLY for
  automated reasons (`cod_refund_pending`, `negative_balance_auto_hold`). Admin-set holds are never
  auto-released.
- **Collection remains manual/legal** by design: use the report to open ops tickets, invoice the
  seller, or pursue the KYC/terms process. Record any recovered funds via the adjustment endpoint
  (payout-eligible `adjustment`, positive) so the ledger closes the debt.

## 15) Financial invariant monitor & job heartbeats

Two monitoring crons close the observability gaps identified in the July 2026 finance audit (H-3
account-level invariants, L-2 job liveness). Both use the standard Redis-lock pattern and report
into the JobHealth store like every other finance cron.

### Financial invariant monitor (`financial_invariant_monitor`)

Daily sweep (default 03:00 UTC, lock `lock:financial_invariant_monitor`) over the JournalEntry
shadow ledger. Every individual journal entry balances by construction, so corruption only ever
appears at the ACCOUNT level — the reserve-release loop kept 13 entries perfectly balanced while
manufacturing seller credit. Checks, per seller/currency where applicable:

- **`reserve_balance_positive` (critical):** a per-seller `Reserve` balance (debits − credits) must
  never be positive — positive means more reserve released than withheld.
- **`seller_payable_drift` (high):** journal `Seller_Payable` (credits − debits) vs the flat
  ledger's outstanding payout-eligible rows (`pending` + `scheduled`), beyond
  `INVARIANT_SELLER_PAYABLE_TOLERANCE_CENTS` (default 100).
- **`cash_position_mismatch` (high):** journal `Cash` per currency vs a reconstruction from
  independent sources: delivered prepaid order totals − refunds − processor fees − paid payouts −
  tax remittances − shipping label costs. v1 limitations: COD cash flows and payout reversals are
  outside the reconstruction (COD sale journals post no Cash, so prepaid-only keeps the comparison
  symmetric; an unre-executed payout reversal will surface here). NOTE: until the processor-fee
  recovery journal mapping (audit finding M-1) is fixed and historical data cleaned, this check is
  EXPECTED to alert with a delta of ~1,016 cents per prepaid order — that is the M-1 misstatement
  being detected, not a false positive.
- **`duplicate_idempotency_key` (critical):** duplicate JournalEntry idempotency keys — the guard
  against unique-index drift.

Violations are logged with `alert: true` and emailed to the reconciliation report recipients
(`SETTLEMENT_RECONCILIATION_REPORT_EMAILS` / `..._TO` / `SUPPORT_EMAIL`).

### Job heartbeat monitor (`heartbeat_monitor`)

Daily watchdog (default 04:00 UTC, lock `lock:heartbeat_monitor`) over the JobHealth store — the
single source of truth for job liveness (no separate heartbeat collection). All finance crons report
start/completion via `markJobStarted`/`markJobCompleted`: settlement scheduler, payout retry, seller
debt monitor (pre-existing), reserve release, settlement reconciliation, and the invariant monitor
(added with this feature). The watchdog alerts when a job's last SUCCESSFUL completion is older than
`HEARTBEAT_{JOB}_INTERVAL_MINUTES` (defaults: 26 h for daily jobs so a late cron doesn't flap; 2 h
for the hourly payout retry), or when an enabled job has never reported. Jobs disabled via their own
`*_ENABLED` env are skipped. A crash-looping job still pages once its last success ages out (the
failed-run timestamp does not reset the clock).

There is no standalone settlement-execution cron: payouts execute inside the scheduler cycle
(`SETTLEMENT_AUTO_EXECUTE`) and the payout-retry cycle, so execution liveness is covered by those
two heartbeats.

## 16) Journal (double-entry) shadow ledger — design reference

Feature-flagged by `JOURNAL_LEDGER_ENABLED`. A `JournalEntry` (`models/journalEntry.model.js`)
mirrors every financial event as ONE balanced double-entry document alongside the flat `LedgerEntry`
rows; the flat ledger remains the operational source of truth that settlement aggregates.

### One entry per EVENT — never one per order

An entry is "one thing that happened to money at one moment": an `eventType`, a `reference`, and a
`postings[]` array where Σdebits must equal Σcredits (enforced in `pre('validate')` and re-checked
in `createJournalEntry`). Order-level grouping is impossible by design: settlement withholds,
reserve releases, and payouts are batch/seller-level events that span many orders and happen weeks
apart, and an append-only document cannot keep absorbing later events without mutating. Per-order
views are aggregations on top (the admin tab's lifecycle timeline), not the storage shape.

### Idempotency keys

All keys are `journal:`-prefixed and identify the event's canonical id, never its amount:

| Event                    | Key pattern                                        |
| ------------------------ | -------------------------------------------------- |
| Sale (delivery)          | `journal:delivered:{orderId}`                      |
| Processor fee / recovery | `journal:processor_fee[_recovery]:{orderId}`       |
| Refund                   | `journal:refunded:{orderId}:{refundKeyPart}`       |
| Settlement withhold      | `journal:settlement_reserve:{batchId}:{currency}`  |
| Reserve release          | `journal:` + the flat release row's operation key  |
| Payout execution         | `journal:payout_execution:{payoutId}:{transferId}` |

Dual-writes run post-commit via `postJournalEntrySafely` (never inside the ledger transaction, never
throws); a lost write self-heals on the next replay of the same event.

### Expected entries for a typical prepaid order (settlement + reserve)

About **6 entries touch the order** across its lifecycle — only the first three are order-scoped;
the rest are shared with every other order in the same batch/seller cycle:

| #   | Event                      | Scope         |
| --- | -------------------------- | ------------- |
| 1   | `processor_fee` (estimate) | Order         |
| 2   | `processor_fee` (recovery) | Order         |
| 3   | `sale` (at delivery)       | Order         |
| 4   | `settlement` (withhold)    | Batch         |
| 5   | `reserve_release`          | Seller        |
| 6   | `payout_execution`         | Seller/Payout |

Materially more entries for one order means refunds/adjustments occurred — or something is wrong
(the July 2026 reserve-release loop produced 13; the invariant monitor now guards this).

### Reading the admin Journal Entries tab

One row per JournalEntry. The scope badge tells you what kind of entry you are looking at: **Order**
(has `reference.orderId`), **Batch** (settlement withhold — spans all sellers in the batch),
**Seller** (release/payout/adjustment — spans all of that seller's orders), **Platform** (tax
remittance, label costs — no seller posting). Filtering by Order ID and toggling the lifecycle
timeline merges the order's own entries with its seller's aggregate events chronologically, with the
running Seller_Payable balance after each event. The collapsible mini trial balance above the table
sums per-account balances from the loaded entries — a positive (debit) Reserve balance is
highlighted as an error because it means more reserve was released than withheld.

### Chart of accounts ↔ ledger entry types

Accounts are defined in `services/ledger/chartOfAccounts.js`; builders in
`services/ledger/journalBuilders.js` map flat entry types onto them:

| Account               | Typical postings from                                          |
| --------------------- | -------------------------------------------------------------- |
| Cash                  | sale (Dr), refunds (Cr), payouts (Cr), remittances/labels (Cr) |
| Seller_Payable        | sale net (Cr), withhold (Dr), release (Cr), payout (Dr)        |
| Reserve               | settlement withhold (Cr), reserve release (Dr) — per seller    |
| Platform_Revenue      | commission (Cr), adjustments                                   |
| Tax_Liability         | tax collected (Cr), tax_remittance (Dr)                        |
| Shipping_Liability    | carrier shipping collected (Cr), shipping_label_cost (Dr)      |
| Processor_Fee_Expense | processor fees not passed through (Dr), recoveries (Cr)        |
| Refunds_Payable       | COD refund obligations                                         |
