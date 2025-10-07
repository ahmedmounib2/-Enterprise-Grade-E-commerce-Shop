# Deployment Runbook

This guide explains how we ship the AhmedMonib E-Shop stack to each surface (frontend, backend API,
and Expo mobile client). Follow these steps whenever you promote code to staging or production so
rollouts remain predictable and auditable.

---

## Prerequisites

- All changes merged into `main` via reviewed pull requests.
- CI pipeline (lint, tests, type checks) green on the merge commit.
- Incident response channel (`#incidents`) informed for planned maintenance windows.
- Required secrets stored in the respective hosting platforms (Vercel, Railway, Expo EAS) as
  described below.

---

## Frontend (Vercel)

1. Ensure `frontend/.env.production` is up to date with values from the secure secret manager.
2. From the repository root, run a smoke build locally if significant UI or dependency changes
   landed:

   ```bash
   npm install
   npm -w frontend run build
   ```

3. Trigger a production deployment:

   ```bash
   npx vercel --cwd frontend --prod --yes
   ```

4. Monitor the Vercel deployment dashboard until the build completes. Validate that:
   - Environment variables match the expected values (particularly `PUBLIC_CLIENT_FALLBACK_URL`).
   - The preview URL serves correctly before promoting it to production (Vercel auto-promotes when
     using `--prod`).
5. Announce completion in the `#deployments` channel and note the commit hash.

Rollback: use the Vercel dashboard to promote the previous successful deployment or run
`npx vercel rollback <deployment-id>`.

---

## Backend API (Railway)

1. Verify the Docker image builds locally:

   ```bash
   docker build -t eshop-backend:latest .
   ```

2. Run the container against staging credentials to smoke test critical endpoints:

   ```bash
   docker run --rm -p 5000:5000 --env-file backend/.env.staging eshop-backend:latest
   ```

   - Hit `/api/health` to confirm readiness.
   - Trigger a sample order flow against the staging Stripe keys if feasible.

3. Push the changes to Railway via `railway up` or rely on the GitHub Actions workflow that watches
   the `main` branch. Confirm the deployment finished by checking the Railway build logs.
4. After the release, monitor:
   - Railway metrics (CPU, memory, HTTP 5xx rate).
   - Sentry alerts for new errors.
   - Stripe webhook deliveries to ensure no events fail.
5. If an issue arises, redeploy the previous container image from the Railway "Deployments" tab or
   roll back the Git commit triggering the automated build.

---

## Mobile (Expo)

### OTA updates (JavaScript only)

1. Update `app.config.ts` version or release channel notes as needed.
2. Publish with Expo Application Services (EAS):

   ```bash
   npm -w mobile run env:lan   # or env:tunnel based on target
   npx expo publish --release-channel production
   ```

3. Verify the release appears in the Expo dashboard and test on a physical device using the
   production channel.

Rollback: republish the last known-good bundle with `expo publish` using the corresponding release
channel tag.

### Native releases (APK / AAB)

1. Increment the `versionCode` and `version` in `app.json`.
2. Build with EAS:

   ```bash
   npx eas build -p android --profile production
   ```

3. Download the generated artifact, smoke test on a device, then upload to the Google Play Console.
4. Update the release notes in the Play Console and roll out to the desired track (internal, beta,
   production).
5. Monitor Crashlytics and Expo error logs for regressions.

Rollback: promote the previous build in the Play Console or submit an emergency update with the
previous working commit.

---

## Post-deployment checklist

- Update the deployment log (see `docs/INCIDENT_LOG.md`) with the date, surfaces deployed, and
  commit hashes.
- Confirm analytics dashboards (Vercel, Railway, Stripe, Datadog) show expected traffic without
  anomalies for the first 30 minutes post-deploy.
- Close the change request ticket or merge request with a summary of user-facing impacts.
- If the deployment introduces a feature flag, ensure there is a documented plan to remove or toggle
  it after validation.

Keep this runbook updated when tooling or procedures change so on-call responders have accurate,
actionable documentation during incidents.
