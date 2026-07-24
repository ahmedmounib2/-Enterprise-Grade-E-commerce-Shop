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
   - Settlement scheduler health: verify cron cadence, new batch creation, and payout
     execution/manual fallback queue metrics after release.
   - Any newly added MongoDB indexes: on large collections, index builds can increase CPU/IO and
     temporarily slow writes. Prefer rolling out during lower traffic windows and monitor Mongo
     operations until the build completes.
5. If an issue arises, redeploy the previous container image from the Railway "Deployments" tab or
   roll back the Git commit triggering the automated build.

---

## Mobile (Expo) — EAS Build + EAS Update (managed workflow)

The mobile app uses the **Expo managed workflow**: the native `android/` project is **not**
committed to Git (it is `.gitignore`d). **EAS Build generates it from `mobile/app.config.js` via
`expo prebuild` at build time**, then compiles it. JavaScript-only over-the-air (OTA) patches ship
via **EAS Update**.

### App variants (three side-by-side installs)

There are no Gradle flavors. Instead, `app.config.js` reads `process.env.APP_VARIANT` and sets a
distinct `applicationId`, deep-link scheme, and launcher label per build. Each `eas.json` profile
sets `APP_VARIANT` in its `env`:

| Profile       | `APP_VARIANT` | applicationId                   | Display name          | Scheme              | Output | Channel      |
| ------------- | ------------- | ------------------------------- | --------------------- | ------------------- | ------ | ------------ |
| `production`  | `production`  | `com.ahmedmonib.eshop`          | `Vexflare`            | `vexflare`          | AAB    | `production` |
| `internal`    | `internal`    | `com.ahmedmonib.eshop.internal` | `Vexflare (Internal)` | `vexflare-internal` | APK    | `internal`   |
| `development` | `development` | `com.ahmedmonib.eshop.dev`      | `Vexflare (Dev)`      | `vexflare-dev`      | APK    | `internal`   |

Because the three application IDs differ, all three apps can be installed on one device at once.

The `internal` and `production` profiles also set the public runtime env in `eas.json`
(`EXPO_PUBLIC_API_BASE_URL`, `EXPO_PUBLIC_WEB_ASSET_ORIGIN`, and the `EXPO_PUBLIC_FEATURE_*` flags,
mirroring `mobile/.env.production`) so EAS builds render categories / product images and
feature-gated UI. The `development` profile omits them on purpose — the dev client reads them from
your local `.env` (`npm -w mobile run env:tunnel`).

OTA matching is governed by `runtimeVersion` (policy `appVersion`), so an OTA only reaches builds
sharing the same `version` string. Bumping `version` requires a fresh store build before OTA resumes
— intentional, prevents a JS bundle running against incompatible native code. The launch-time
check/fetch/reload is owned by `mobile/src/hooks/useOtaUpdates.js` (native auto-check is `NEVER`;
the imperative engine lives in `mobile/src/ota/otaService.js`).

**Launch flow.** On cold launch the hook runs while `App.js` is still showing its pre-mount
placeholder: it checks for an update (time-boxed to ~2.5 s, and the whole launch OTA phase is
hard-capped at ~4 s so a slow network never wedges startup). If an update downloads inside that
window it is applied seamlessly with `reloadAsync()` before the first frame. If the window has
already passed, the update downloads in the background and a themed **"A new version is ready"**
banner (`OtaUpdateBanner`) invites the user to apply it. A user-triggered reload is suppressed while
the user is in a reload-unsafe screen (checkout, payment, auth, disputes, profile edit — see
`CRITICAL_ROUTES` in `mobile/src/stores/useOtaStore.js`), so unsaved work is never destroyed. A
best-effort rollback marker (Expo SecureStore) lets the next launch log whether an applied update
actually stuck.

### SSL certificate pinning

`mobile/plugins/withSslPinningCerts.js` (a local config plugin) copies `mobile/certs/*.cer` into the
generated `android/app/src/main/assets/` during prebuild — the managed-workflow replacement for the
old `copyPinnedCerts` Gradle task. The `.cer` files are **git-ignored** (fetched on demand with
`npm -w mobile run cert:pull`), so they are absent on EAS unless provided:

- **Internal / development** builds run with pinning **off** (`EXPO_PUBLIC_SSL_PINNING_CERTS`
  empty), so no cert is needed — the plugin safely no-ops.
- **Production** pinning needs the cert present at prebuild. Provide it on EAS via an
  `eas-build-post-install` hook that runs `cert:pull`, or supply it as an EAS file secret. (The
  plugin copies whatever `.cer` files exist and skips silently when none are present.)

### Signing

EAS Build signs with the **upload keystore you uploaded to the Expo dashboard** (Play App Signing
re-signs on Google's side) — there is no Gradle signing block to maintain. For a **local** release
build you must run `expo prebuild` first and then provide signing yourself (the generated project
has no committed signing config).

### Native release (EAS Build)

1. Increment `version` and `android.versionCode` in `mobile/app.config.js`.
2. Build the artifact:

   ```bash
   eas build --platform android --profile production   # AAB → Play Console
   eas build --platform android --profile internal      # APK → sideload testing
   ```

3. Download the artifact, smoke test on a device, then upload the AAB to the Google Play Console.
4. Update the Play Console release notes and roll out to the desired track (internal, beta,
   production).
5. Monitor Sentry and Play Console vitals for regressions.

Rollback: promote the previous build in the Play Console, or ship a corrective OTA (below) if the
regression is JavaScript-only.

### OTA update (EAS Update — JavaScript only)

1. Confirm the change is JS-only (no native dependency or config change) and that the target builds
   share the current `runtimeVersion`.
2. Publish to a channel:

   ```bash
   # Publish a JS-only fix to the internal channel
   eas update --channel internal --message "fix: <describe the fix>"

   # Publish a JS-only fix to the production channel
   eas update --channel production --message "fix: <describe the fix>"
   ```

3. Verify the update appears in the Expo dashboard, then relaunch the app on a device on that
   channel to confirm it downloads and applies on the next launch.

**Rollback:**

```bash
# Rollback internal
eas update:rollback --channel internal

# Rollback production
eas update:rollback --channel production
```

When you need a rollback:

- The published OTA introduced a crash, white screen, or broken flow that reaches users on their
  next app launch — roll back immediately to restore the last good JS bundle without cutting a new
  build.
- A regression was caught on the `internal` channel before promoting to production — roll back
  `internal`, fix, and re-publish.
- The "JS-only" fix turns out to need a native change (it works in the dev client but errors on the
  installed binary) — roll back, then ship the native change via `eas build`.
- A rollback is itself a new update pointing at the previous bundle, so users receive it on their
  next launch (it does not instantly remove the bad update). If a correct fix is ready quickly,
  prefer "fix forward" (publish a new good update) — it has the same reach as a rollback.

> ⚠️ OTAs only reach builds sharing the same `runtimeVersion` (currently `appVersion`). If you bump
> the version in `app.config.js`, you must do a fresh store build before OTA updates resume. OTAs
> cannot change native code — for native changes, use `eas build`.

> **Local internal/production builds now receive OTAs — but the first build on a channel must still
> be EAS.** The channel (`expo-channel-name` request header) is what maps a binary to its update
> branch. EAS Build injects it automatically; a plain local `expo prebuild` + `./gradlew` build used
> to omit it, so locally-built binaries had `Updates.channel = null` and never received
> channel-scoped OTAs (the confirmed root cause of the 2026-07 "OTA never applies" incident —
> v1.51.0 on the Play Store was built locally with no `production` channel). This is now fixed:
> `mobile/app.config.js` declares `updates.requestHeaders: { 'expo-channel-name': <channel> }` keyed
> off `APP_VARIANT` (`production` → `production`, `internal` → `internal`, `development` → none), so
> `expo prebuild` bakes the channel into local builds too. A local `APP_VARIANT=production` /
> `APP_VARIANT=internal` build therefore receives OTAs on its channel.
>
> The **first** build for a brand-new channel must still run on EAS (`eas build`) — or the channel/
> branch must be created via the EAS CLI (`eas channel:create` / `eas update`) — because a purely
> local build never contacts Expo's servers and so cannot create the channel↔branch mapping. Once
> that mapping exists (as it does for `production` and `internal`), subsequent local builds on that
> channel receive updates. Development builds stay channel-less and never attempt OTA.

### OTA diagnostics — "an update was published but never applies"

`eas update:list --branch <name>` showing an update proves it exists on the **branch**, but the app
requests updates by **channel** and `runtimeVersion`. Work the pipeline outside-in — the most common
causes are server-side (channel↔branch mapping) or a build-identity mismatch, not the client code.

Step 1 — zero-code checks (no build needed):

```bash
eas update:list --branch production --json   # latest update's runtimeVersion == the build's (1.51.0)? platforms include yours? isRollBackToEmbedded false?
eas channel:view production                    # is channel `production` linked to the branch that holds the update?  <-- most common cause
eas branch:view production                     # updates actually present on this branch?
eas build:list --platform android --limit 5    # did the installed build come from `--profile production` (channel production, runtimeVersion 1.51.0)? not a local `expo export`?
adb logcat | grep -iE 'expo-updates|EXUpdates' # cold-launch the app; the native module logs its fetch/skip decision
```

A missing/wrong channel→branch mapping, a runtimeVersion/platform mismatch, or an update that was
only ever produced by a local `expo export` (never `eas update`-published) will each make the client
correctly find "no update available."

Step 2 — reproduce on an internal build (the client is instrumented there):

The OTA Debug surface is compiled only into dev/internal builds (`isOtaDebugEnabled` =
`__DEV__ || Updates.channel !== 'production'`); production stays silent. To inspect the exact client
code path, publish the same update to `internal` and open the debug screen:

```bash
eas update --channel internal --message "ota diag"
# install the internal build, then open the deep link:
#   vexflare-internal://ota-debug
```

The screen shows the running `channel`, `runtimeVersion`, `updateId`, `isEmbeddedLaunch`,
`isEnabled`, and a live stage log
(`checking → available/none/checkError → downloading → downloaded → reloading → rollbackDetected/updateApplied`).
The manual **Check / Download / Reload** buttons surface the real `checkError`/`downloadError` that
the old flow used to swallow. Whatever fails here fails identically on production.

### Manual build (local fallback)

Since `android/` is no longer committed, a local native build must **generate it first** with
`expo prebuild` (the `APP_VARIANT` selects which application ID / scheme is baked in). There are no
Gradle flavors, so the tasks are the plain `assembleRelease` / `bundleRelease`. The
`withReleaseSigning` config plugin injects the release signing config, so export the keystore first
for a **release** build:

```bash
export ESHOP_ANDROID_KEYSTORE=~/Dev/secrets/eshop-upload-keystore.jks
export ESHOP_ANDROID_KEYSTORE_PASSWORD='<from Bitwarden>'
export ESHOP_ANDROID_KEY_ALIAS='<from Bitwarden>'
export ESHOP_ANDROID_KEY_PASSWORD='<from Bitwarden>'

# Local builds bake in mobile/.env — set the right profile FIRST:
npm -w mobile run env:internal   # internal build: production API + correct feature flags
# (dev build instead: npm -w mobile run env:tunnel)

# Uninstall old conflicting apps before installing a new build:
adb uninstall com.ahmedmonib.eshop.dev || true
adb uninstall com.ahmedmonib.eshop.internal || true
adb uninstall com.ahmedmonib.eshop || true

cd mobile

# Wipe Metro's cache
npx expo start --clear &
sleep 5 && kill %1   # Start Metro to clear its cache, then stop it


APP_VARIANT=internal npx expo prebuild --platform android --clean   # clean prebuild when switching variants
cd android && ./gradlew assembleRelease                             # → app/build/outputs/apk/release/app-release.apk
adb install -r app/build/outputs/apk/release/app-release.apk
# Play AAB: APP_VARIANT=production prebuild --clean, then ./gradlew bundleRelease
# Dev client (debug, no keystore): env:tunnel, then APP_VARIANT=development npx expo run:android
#   (or ./gradlew assembleDebug && adb install -r app/build/outputs/apk/debug/app-debug.apk)
```

> Note: `expo prebuild` overwrites `android/`. There are no product flavors — each build's
> application ID comes from `APP_VARIANT`. Missing `ESHOP_ANDROID_*` on a release build fails fast
> with a clear error (never an unsigned build). See `mobile/docs/android-release-play-store.md` and
> `mobile/docs/build-and-install.md` for device install and signing details.

> **OTA on local builds:** because `app.config.js` now injects `updates.requestHeaders`
> (`expo-channel-name`) keyed off `APP_VARIANT`, a local `APP_VARIANT=production` or
> `APP_VARIANT=internal` build carries its channel and **does** receive OTA updates — no longer
> EAS-only (as long as that channel already exists on Expo's servers; see the note in the OTA
> section above). `APP_VARIANT=development` builds carry no channel and never attempt OTA.

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

## Settlement retry operator note

- Reversed settlement payouts are moved to `retryable` and are reprocessed through the same retry
  lane as failed payouts.
- Batch retry endpoint accepts both `failed` and `retryable` payouts; if your environment has a
  single-payout retry endpoint, it also accepts both statuses.
- Expected post-retry progression is `failed|retryable` -> `processing` -> `paid` (or
  `manual_required` when payout prerequisites are still missing).
