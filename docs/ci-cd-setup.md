# CI/CD Setup

## Workflows

### `.github/workflows/ci.yml` – Continuous Integration

Triggers on every push to `main` and every pull request targeting `main`.

| Job                      | What it does                                                        |
| ------------------------ | ------------------------------------------------------------------- |
| `lint`                   | Runs `npm run lint` across all workspaces                           |
| `test-backend`           | Runs `npm test` in `backend/` (Jest + mongodb-memory-server)        |
| `test-frontend`          | Runs `npm test` in `frontend/` (Jest + Testing Library)             |
| `test-mobile`            | Runs `npm test` in `mobile/` (Jest; no device, emulator or network) |
| `architecture-integrity` | Runs `npm run test:architecture` — see `docs/architecture-page.md`  |
| `build`                  | Runs `npm run build` in `frontend/` (Vite production build)         |

All jobs use `actions/setup-node@v4` with `cache: npm` to reuse `~/.npm` across runs. A
`concurrency` group per branch cancels in-progress runs when a new push arrives.

#### Backend test environment variables

The `test-backend` job sets a minimal set of fake secrets inline in the workflow so that:

- `cache/index.js` does not throw `No Redis config found` at module load (requires `REDIS_URL`).
- JWT signing functions in `auth.utils.js` receive a non-empty secret string.

A real **Redis service container** (`redis:7`) runs alongside the test job. The `REDIS_URL`
environment variable points at `redis://localhost:6379` so cache-layer tests that require an actual
Redis client work without mocks. All other external services (MongoDB, Stripe, Cloudinary, email)
are still mocked by Jest. The inline secret values are intentionally fake and must **never** be used
outside of CI test runs.

No additional repository secrets are needed for the base CI workflow to pass.

#### What `test-mobile` protects

The mobile suite is where the certificate-pinning guarantees are asserted, so it is a security gate
rather than only a regression check:

- `sslPinningAdapter.test.js` — pinning fails **closed** in production when the native module is
  missing, no request escapes before native pinning finishes initialising, and a network error never
  downgrades the connection to an unpinned one.
- `sslPinResolution.test.js` — the pin hashes exist in **one** file only, `mobile/ssl-pins.json`
  (checked across every tracked file); `internal` and `production` resolve byte-identical pinning
  configuration; no shippable EAS profile trusts user-installed CA certificates.

None of it touches the network or a device — `app.config.js` is resolved in a child process. The
live certificate chain is checked separately, by `ssl-pins.yml` below.

### `.github/workflows/ssl-pins.yml` – Mobile Certificate Pin Validation

| Trigger                                                                                     | Checks                                           |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Schedule, Mondays 06:00 UTC                                                                 | Stale pins **and** pin-set expiry within 90 days |
| `workflow_dispatch`                                                                         | Stale pins                                       |
| PR / push to `main` touching `mobile/ssl-pins.json`, `compute-ssl-pins.js`, `app.config.js` | Stale pins                                       |

It runs `npm -w mobile run cert:pins -- --check`, which opens a real TLS connection to the API host
and compares the presented chain against `mobile/ssl-pins.json`. It **fails** when no configured pin
appears in the live chain — a build released in that state could not reach the API at all — and, on
the scheduled run, when the pin set's `expirationDate` is within 90 days, after which pin validation
switches itself off in every installed build.

Both failures matter disproportionately because **pinning configuration cannot be delivered over the
air**: an app that cannot reach the API also cannot download an OTA update to fix itself, so
recovery requires a store release.

Two deliberate design points:

- **It is not part of `ci.yml`.** The check needs a live handshake against the API, so a backend
  outage would otherwise fail unrelated pull requests. Keeping it separate means a red run here
  always means something about the pins, never something about the backend's uptime.
- **A scheduled failure opens a GitHub issue** (label `ssl-pins`, one open at a time, reused via a
  comment). A cron run has no pull request to annotate, so the failure would otherwise be a red mark
  nobody sees.

No secrets or configuration are required: the host comes from the `host` field of `ssl-pins.json`,
and `GITHUB_TOKEN` (with `issues: write`) is supplied automatically. The job carries
`timeout-minutes: 10`, and the script caps its own TLS handshake at 15 seconds.

Neither workflow proves pinning is **enforced** — that needs a physical device. See
`mobile/docs/ssl-pinning.md` §4.2 (`internal-pinfail`) and §4.3–4.4 (the MITM pair).

### `.github/workflows/security.yml` – Security Scanning

Triggers on every push (all branches).

| Job           | What it does                                                                                 |
| ------------- | -------------------------------------------------------------------------------------------- |
| `secret-scan` | Runs [Gitleaks](https://github.com/gitleaks/gitleaks) on full git history (`fetch-depth: 0`) |
| `audit`       | Runs `npm audit --audit-level=high` in `backend/` and `frontend/`                            |

The audit steps use `continue-on-error: true` because high-severity advisories currently exist in
transitive dependencies. This keeps the pipeline green while they are tracked. See
`docs/peer-dependency-conflicts.md` for the resolution plan. Remove `continue-on-error` once all
high-severity advisories are resolved.

### `.github/dependabot.yml` – Dependency Updates

Monthly PRs for npm packages in `/`, `/backend`, `/frontend`, `/mobile`, and GitHub Actions in `/`.
Limit of 5 open PRs per ecosystem to avoid noise.

---

## Enabling Branch Protection on `main`

1. Go to **GitHub repo → Settings → Branches → Add rule**.
2. Branch name pattern: `main`.
3. Enable:
   - **Require a pull request before merging** (at least 1 approval)
   - **Require status checks to pass** — add `lint`, `test-backend`, `test-frontend`, `test-mobile`,
     `architecture-integrity`, `build`. Do **not** add the SSL pin check: it only runs on some
     pushes, and a required check that does not run blocks merges indefinitely.
   - **Require branches to be up to date before merging**
   - **Do not allow bypassing the above settings**
4. Save.

---

## Enabling Secret Scanning and Dependabot Alerts

1. **Secret scanning:** Go to **Settings → Security → Code security and analysis** → enable **Secret
   scanning** and **Push protection**.
2. **Dependabot alerts:** Enable **Dependabot alerts** and **Dependabot security updates** on the
   same page. The `dependabot.yml` file controls update frequency; alerts are enabled separately.

---

## Secrets required in GitHub Actions

These secrets must be added at **Settings → Secrets and variables → Actions** for the non-test
workflows to work. The `test-backend` and `test-frontend` jobs use inline fake values and do not
need any of these.

| Secret name         | Required by           | Notes                                      |
| ------------------- | --------------------- | ------------------------------------------ |
| `SENTRY_AUTH_TOKEN` | `frontend-sentry.yml` | From Sentry → Settings → Auth Tokens       |
| `SENTRY_ORG`        | `frontend-sentry.yml` | Your Sentry organization slug              |
| `SENTRY_PROJECT`    | `frontend-sentry.yml` | Your Sentry project slug                   |
| `VITE_SENTRY_DSN`   | `frontend-sentry.yml` | Optional; disables Sentry client if absent |

---

## Vercel proxy double-egress observation

The `vercel.json` rewrite rules route all non-`/assets/` paths to `/index.html` (correct for the
SPA) and proxy `/api/:match*` to the Railway backend. This creates a round-trip where every API call
travels:

```
user → Vercel → Railway → Vercel → user
```

API responses are forwarded through Vercel on the way back, which doubles outbound egress costs
relative to a direct client-to-API path.

### Future options

1. **Serve the frontend from Railway directly** — remove Vercel entirely. The Express server serves
   the Vite build as static files; API and frontend share the same origin, eliminating the proxy
   hop.
2. **Keep Vercel for the SPA; call the Railway API directly** — set `VITE_API_BASE_URL` (or
   `SERVER_URL`) to the Railway API origin in the frontend build, and add the appropriate CORS
   headers on the backend (`PRODUCTION_ORIGIN`). The SPA is delivered by Vercel's CDN but API
   requests bypass it entirely.

**Out of scope — revisit during networking/cost review.**

---

## Railway deployment checklist

Railway reads environment variables from the service's **Variables** panel. The backend will print a
clear error and exit immediately if any of the following are missing in production (enforced by
`backend/src/utils/checkRequiredEnv.js`):

```
MONGO_URI
SESSION_SECRET
ACCESS_TOKEN_SECRET
REFRESH_TOKEN_SECRET
FIELD_ENCRYPTION_KEY
OAUTH_BRIDGE_SECRET
MOBILE_DASHBOARD_BRIDGE_SECRET
SELLER_DOC_VIEW_TOKEN_SECRET
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
SERVER_URL
CLIENT_URL
PRODUCTION_ORIGIN
MONITORING_METRICS_TOKEN
```

Refer to `backend/.env.example` for generation instructions and recommended values for each
variable. Optional integrations (Cloudinary, SendGrid/Resend, Chimoney, Sentry) are not enforced at
boot but will degrade gracefully when absent.
