# Monitoring Metrics Access — Token Strategy

## Current approach (single shared token)

The Prometheus-compatible scrape endpoint (`GET /api/monitoring/metrics`) is protected by a single
static bearer token:

```
Authorization: Bearer <MONITORING_METRICS_TOKEN>
```

`MONITORING_METRICS_TOKEN` is set in the backend environment. All scrapers — Prometheus, Grafana
Cloud, Datadog, or any other consumer — must present this same token. The endpoint returns
`404 Not Found` when the variable is absent, preventing accidental metric exposure.

This is sufficient for a single Prometheus instance and has no operational overhead.

## Limitation with multiple scrapers

If a second scraper is added (e.g. Grafana Cloud alongside an existing Prometheus instance), both
must share the same token. Rotating it requires updating **every** scraper simultaneously, which
introduces a coordination window where at least one scraper loses access.

## Future improvement — per-scraper token map

When multiple scrapers are required, replace the single token with a JSON map:

```
MONITORING_METRICS_TOKENS={"prometheus":"tok1","grafana":"tok2","datadog":"tok3"}
```

Each scraper sends `Authorization: Bearer <its-own-token>`. The backend looks up the presented token
in the map, accepts the request if any entry matches, and logs the **scraper name** on successful
authentication (useful for audit trails and debugging).

Migration steps:

1. Add the `MONITORING_METRICS_TOKENS` env var alongside (not replacing) the existing
   `MONITORING_METRICS_TOKEN`. The middleware should try the map first, then fall back to the
   single-token var for backwards compatibility.
2. Update each scraper's configuration to use its own token.
3. Rotate individual tokens independently via the map — no coordination needed.
4. Once all scrapers are migrated, remove `MONITORING_METRICS_TOKEN`.

**This is a future improvement — no code changes are made today.** The current single-token approach
remains in effect.

## See also

- `docs/security/webhook-rotation.md` — rotation procedure for webhook secrets
- `docs/security/access-controls.md` — broader access control policy
- README.md → _Monitoring_ section for endpoint details
