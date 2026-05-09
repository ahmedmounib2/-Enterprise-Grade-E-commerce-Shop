# Monitoring & Alert Rules

## Metrics exposure

- Prometheus scrape endpoint: `GET /api/monitoring/metrics`
- Datadog metric forwarding: emitted as structured `datadog.metric` logs for existing metric events.

## New operational metrics

- `settlements_payouts_retryable_count` (counter): number of retry-eligible payouts discovered by the payout retry cron.
- `settlements_payouts_retryable_attempt` (counter): number of retry attempts executed.
- `cron_settlement_scheduler_run` (counter with `status` label): scheduler lifecycle outcomes (`started`, `success`, `failed`).
- `cron_settlement_scheduler_last_run_epoch_ms` (counter-style snapshot): last scheduler run timestamp in epoch milliseconds.

## Recommended alert rules

### Payout retry failure-rate alert

Use job health (or `cron_settlement_scheduler_run{status="failed"}` equivalents) to trigger when failure rate exceeds threshold.

Example condition:

- Over last 20 payout retry runs, failed ratio >= 0.30
- Fire if persisted for 10 minutes

### Stale scheduler alert

Trigger if scheduler has not successfully completed in the expected freshness window.

Example condition:

- `now() - max(cron_settlement_scheduler_last_run_epoch_ms) > 3600000`
- Fire if persisted for 15 minutes

These mirror existing backend stale/failure checks in `jobHealth.service.js` and provide external observability hooks for Prometheus/Datadog dashboards.
