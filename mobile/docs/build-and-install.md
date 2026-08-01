# Android builds: EAS internal APK & production AAB

Muscle-memory reference for producing an **internal sideload APK** and a **Play Console AAB** with
**EAS Build** (Expo **managed** workflow). The native `android/` project is **not** committed — EAS
generates it from `mobile/app.config.js` (`expo prebuild`) at build time. There are **no Gradle
flavors**; the app variant is selected by `APP_VARIANT` (see below).

> TL;DR: `eas build --profile internal` → download APK → `adb install`. For Play:
> `eas build --profile production` → download AAB → upload to the Console.

---

## App variants

`mobile/app.config.js` reads `process.env.APP_VARIANT` and sets a distinct `applicationId`,
deep-link scheme, and launcher name per build. Each `eas.json` profile sets `APP_VARIANT` in its
`env`, so all three apps can be installed on one device at once:

| Variant / profile | applicationId                   | Scheme              | Output             |
| ----------------- | ------------------------------- | ------------------- | ------------------ |
| `production`      | `com.ahmedmonib.eshop`          | `vexflare`          | AAB (Play Console) |
| `internal`        | `com.ahmedmonib.eshop.internal` | `vexflare-internal` | APK (sideload)     |
| `development`     | `com.ahmedmonib.eshop.dev`      | `vexflare-dev`      | local dev-client   |

Locally, pick the variant with the env var, e.g. `APP_VARIANT=development npx expo run:android`.

---

## Prereqs

- Device connected (`adb devices -l` shows it as `device`).
- Java 17 / Android SDK set (see `mobile/custom-dev-setup.md`).
- Logged in to EAS (`eas whoami`), with the upload keystore uploaded to the Expo dashboard.

---

## Internal testing APK (EAS)

```bash
cd mobile
# Build the internal APK on EAS (com.ahmedmonib.eshop.internal, scheme vexflare-internal).
# eas.json's `internal` profile sets APP_VARIANT=internal and buildType apk.
eas build --platform android --profile internal

# When the build finishes, download the APK from the EAS dashboard / build URL, then:
APK_ABS="$(realpath ~/Downloads/<downloaded>.apk)"
adb uninstall com.ahmedmonib.eshop.internal || true   # avoid signature-mismatch on reinstall
adb install -r "$APK_ABS"


# windows install path when apk is at downloads folder
adb install -r "/mnt/c/Users/Admin/Downloads/<downloaded>.apk"
```

> Internal builds run with SSL pinning **ON**, resolved from `mobile/ssl-pins.json` — the same
> canonical source production uses, so the two are byte-identical apart from the application id.
> That holds for local Gradle builds too. Full reference: [`ssl-pinning.md`](./ssl-pinning.md).
>
> The `internal` and `production` profiles also inject the public runtime env in `eas.json`
> (`EXPO_PUBLIC_API_BASE_URL`, `EXPO_PUBLIC_WEB_ASSET_ORIGIN`, and the `EXPO_PUBLIC_FEATURE_*`
> flags), so an EAS build renders categories / product images and feature-gated UI the same as a
> local build that sourced them from `.env.production`. The `development` profile intentionally
> omits them — the dev client gets them from your local `.env` (e.g.
> `npm -w mobile run env:tunnel`).

---

## Production AAB (EAS)

```bash
# 1) Bump `version` and `android.versionCode` in mobile/app.config.js.
# 2) Build the AAB on EAS (com.ahmedmonib.eshop, scheme vexflare). EAS signs it with the
#    keystore you uploaded to the Expo dashboard.
eas build --platform android --profile production

# 3) Download the AAB from the EAS dashboard and upload it to the Play Console.
```

---

## Manual build (local fallback)

EAS is the primary workflow. For an **offline / local** build, generate `android/` with
`expo prebuild` (the `APP_VARIANT` prefix bakes in the application ID + scheme), then run Gradle.
There are **no Gradle flavors**, so the tasks are the plain `assembleRelease` / `bundleRelease` (no
`Internal`/`Prod` infix). The `withReleaseSigning` config plugin injects the release signing config
automatically, so a **release** build needs the keystore exported first.

**Pre-build checklist (every manual build):**

1. **Set the right `.env` first** — unlike EAS (which injects env from `eas.json`), a local build
   bakes in whatever `mobile/.env` currently holds:
   - Vexflare (Dev): `npm -w mobile run env:tunnel` (ngrok API URL, feature flags, etc.)
   - Vexflare (Internal): `npm -w mobile run env:internal` (production API URL + correct feature
     flags). SSL pins are **not** in `.env` — `app.config.js` supplies them from `ssl-pins.json`, so
     a local internal release build is pinned exactly like an EAS one.
2. **Clean prebuild when switching variants** — always regenerate `android/` with `--clean`:
   `APP_VARIANT=development npx expo prebuild --platform android --clean` or
   `APP_VARIANT=internal npx expo prebuild --platform android --clean`.
3. **Uninstall old conflicting apps** before installing the new build:

   ```bash
   adb uninstall com.ahmedmonib.eshop.dev || true
   adb uninstall com.ahmedmonib.eshop.internal || true
   adb uninstall com.ahmedmonib.eshop || true
   ```

Production AAB (→ Play Console):

```bash

export ESHOP_ANDROID_KEYSTORE=~/Dev/secrets/eshop-upload-keystore.jks
export ESHOP_ANDROID_KEYSTORE_PASSWORD='<from Bitwarden>'
export ESHOP_ANDROID_KEY_ALIAS='<from Bitwarden>'
export ESHOP_ANDROID_KEY_PASSWORD='<from Bitwarden>'

# Generate production env

npm -w mobile run env:production

cd mobile
# Wipe Metro's cache
npx expo start --clear &
sleep 5 && kill %1   # Start Metro to clear its cache, then stop it

cd android
./gradlew --stop                     # stop any lingering Gradle daemon
./gradlew cleanBuildCache            # wipe the project-level build cache
cd ..
rm -rf android/.gradle android/build # remove leftover build outputs

# Re-run prebuild and build
APP_VARIANT=production npx expo prebuild --platform android --clean
cd android && ./gradlew bundleRelease
# output: app/build/outputs/bundle/release/app-release.aab
```

Internal sideload APK:

```bash
export ESHOP_ANDROID_KEYSTORE=~/Dev/secrets/eshop-upload-keystore.jks
export ESHOP_ANDROID_KEYSTORE_PASSWORD='<from Bitwarden>'
export ESHOP_ANDROID_KEY_ALIAS='<from Bitwarden>'
export ESHOP_ANDROID_KEY_PASSWORD='<from Bitwarden>'

# Set the internal env FIRST (production API URL + correct feature flags):
npm -w mobile run env:internal

cd mobile
APP_VARIANT=internal npx expo prebuild --platform android --clean

# Wipe Metro's cache
npx expo start --clear &
sleep 5 && kill %1   # Start Metro to clear its cache, then stop it

cd android && ./gradlew assembleRelease
# output: app/build/outputs/apk/release/app-release.apk
adb uninstall com.ahmedmonib.eshop.internal || true
adb install -r app/build/outputs/apk/release/app-release.apk
```

### Pin-enforcement regression APK (`internal-pinfail`)

A build whose SSL pins are **deliberately wrong**, used to prove that pinning is actually enforced
rather than silently inert. Run it after any certificate rotation, any change to the pin set, and
any upgrade of `react-native-ssl-public-key-pinning`.

Why this and not a MITM proxy: apps targeting API 24+ do not trust user-installed CA certificates,
and this app targets API 36 with no `network_security_config.xml`. Installing a mitmproxy/Charles CA
through Settings therefore makes **every** build reject the intercepted connection — pinned or not,
with `ERR_CERT_AUTHORITY_INVALID` from Android's own chain validation, several layers below the
pinning check. A "blocked" result there proves nothing. This test changes only the pin values, so
pinning is the sole variable.

`npm -w mobile run env:internal-pinfail` writes `mobile/.env` **from the `internal-pinfail` profile
in `eas.json`** rather than from a `.env.internal-pinfail` file. `mobile/.env.*` is git-ignored, so
a committed per-profile file is impossible; generating from `eas.json` keeps one copy of the values
and guarantees the local APK and `npm -w mobile run eas:build:pinfail` are identical.

```bash
export ESHOP_ANDROID_KEYSTORE=~/Dev/secrets/eshop-upload-keystore.jks
export ESHOP_ANDROID_KEYSTORE_PASSWORD='<from Bitwarden>'
export ESHOP_ANDROID_KEY_ALIAS='<from Bitwarden>'
export ESHOP_ANDROID_KEY_PASSWORD='<from Bitwarden>'

# Generate .env from the internal-pinfail eas.json profile (intentionally invalid pins).
npm -w mobile run env:internal-pinfail

cd mobile
APP_VARIANT=internal npx expo prebuild --platform android --clean

# Wipe Metro's cache — a stale bundle would still carry the CORRECT pins and the test
# would pass for the wrong reason.
npx expo start --clear &
sleep 5 && kill %1

cd android && ./gradlew assembleRelease
# output: app/build/outputs/apk/release/app-release.apk

adb uninstall com.ahmedmonib.eshop.internal || true
adb install -r app/build/outputs/apk/release/app-release.apk
```

**Expected result — failure is the pass condition:**

| Observation                                                    | Verdict                                     |
| -------------------------------------------------------------- | ------------------------------------------- |
| Every API call fails; login/browse never load                  | ✅ pinning is enforced                      |
| `SSL pinning validation failed for api.vexflare.com` in logcat | ✅ the pin comparison ran and rejected      |
| The app works normally                                         | ❌ **pinning is not being enforced — stop** |

```bash
adb logcat -c && adb logcat | grep -iE "SSL pinning|CertificatePinner|SSLPeerUnverified"
```

Note `chromium … net_error` lines come from the WebView stack, not the OkHttp client the API uses —
an OkHttp pin rejection surfaces as `SSLPeerUnverifiedException: Certificate pinning failure!` plus
the JS listener line above.

**Restore the normal environment afterwards** — the generated `.env` carries a warning banner, but a
forgotten one is otherwise invisible until a later build behaves strangely:

```bash
npm -w mobile run env:internal
cd mobile && APP_VARIANT=internal npx expo prebuild --platform android --clean
cd android && ./gradlew assembleRelease
adb uninstall com.ahmedmonib.eshop.internal || true
adb install -r app/build/outputs/apk/release/app-release.apk
```

The restored build must work normally again — that half matters too, since it confirms the shipped
pins still match the live chain and no lockout is waiting in production.

> **Full runbook:** [`ssl-pinning.md`](./ssl-pinning.md) carries the complete step-by-step
> procedures — environment prerequisites, Charles/mitmproxy setup, per-profile checklists,
> production verification, certificate rotation and troubleshooting. The summaries below are a quick
> reference; that document is authoritative.

### MITM interception test (`internal-mitm` / `internal-mitm-nopin`)

The `internal-pinfail` test above proves the pin comparison runs. This one proves the stronger
property: that pinning rejects a certificate **the device otherwise accepts** — the actual MITM
threat model.

**Why a plain Charles/Proxyman test cannot show this.** Installing the proxy CA through Android
Settings adds it to the _user_ store, and apps targeting API 24+ trust only the _system_ store. The
chain fails ordinary validation with
`SSLHandshakeException: Trust anchor for certification path not found`, and OkHttp evaluates
`CertificatePinner` **only after** the platform TrustManager accepts a chain — so the pinner never
runs. Every build fails identically, pinned or not. The tell that you are hitting this rather than
pinning: unrelated components fail the same way in logcat (`dev.expo.updates`, and Android's own
`NetworkMonitor PROBE_HTTPS`), and the proxy shows **no** flows at all rather than a failed
handshake for the API host.

The `internal-mitm*` profiles set `ANDROID_TRUST_USER_CAS=1`, which activates
`plugins/withUserCaTrust.js` — it emits a `network_security_config.xml` trusting user CAs and wires
it into the manifest. That removes the confound, so the proxy CA becomes genuinely valid and pinning
is the only remaining defence.

> These builds are **deliberately interceptable**. Never distribute one. The plugin throws if
> `ANDROID_TRUST_USER_CAS=1` is combined with `APP_VARIANT=production`, and it is completely inert
> without the flag (no XML, no manifest attribute), so normal builds are unaffected.

Run both halves, changing only the profile:

```bash
# Harness check — unpinned, interceptable. Charles MUST see the traffic.
npm -w mobile run env:internal-mitm-nopin
# …or on EAS: npm -w mobile run eas:build:mitm-nopin

# Pinning check — same build, real pins. Charles must see NOTHING.
npm -w mobile run env:internal-mitm
# …or on EAS: npm -w mobile run eas:build:mitm

# then, for either:
cd mobile && APP_VARIANT=internal npx expo prebuild --platform android --clean
npx expo start --clear & sleep 5 && kill %1      # stale bundle would carry the wrong config
cd android && ./gradlew assembleRelease
adb uninstall com.ahmedmonib.eshop.internal || true
adb install -r app/build/outputs/apk/release/app-release.apk
```

## Verification checklist

Run with the device proxied through Charles and its CA installed via Settings. Steps 1 and 2 are the
differential pair — **step 1 failing to show traffic invalidates step 2**, because it means the
harness itself is not intercepting.

| #   | Build                 | Proxy | Expected                                                                                                                                                                                  |
| --- | --------------------- | ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `internal-mitm-nopin` | on    | Charles **shows** decrypted API traffic to `api.vexflare.com`. App works. Proves the harness intercepts.                                                                                  |
| 2   | `internal-mitm`       | on    | Charles shows **no** API flows. App shows network errors. Logcat: `SSLPeerUnverifiedException` / `Certificate pinning failure!` and `SSL pinning validation failed for api.vexflare.com`. |
| 3   | `internal`            | on    | Blocked (by CA distrust, before pinning). Confirms the shipped build is not interceptable in the field.                                                                                   |
| 4   | `internal`            | off   | App works normally end to end — login, browse, checkout. Confirms the real backend is unaffected.                                                                                         |
| 5   | `production` AAB      | off   | App works normally. Run before any Play upload.                                                                                                                                           |
| 6   | `production` AAB      | on    | Blocked. Same CA-distrust reason as step 3; it is a smoke check, not a pinning proof.                                                                                                     |

```bash
adb logcat -c && adb logcat | grep -iE "SSL pinning|CertificatePinner|SSLPeerUnverified|Trust anchor"
```

Reading the log lines: `Trust anchor for certification path not found` means **Android** rejected
the chain (steps 3 and 6). `Certificate pinning failure!` / `SSLPeerUnverifiedException` means
**pinning** rejected it (step 2). Only the second proves pinning works.

Afterwards, restore with `npm -w mobile run env:internal` and rebuild — the `internal-mitm*` `.env`
carries a warning banner, but a forgotten one leaves you building interceptable APKs.

Local development client (debug build, no keystore needed):

```bash
# Set the dev/tunnel env FIRST (ngrok API URL, feature flags, etc.):
npm -w mobile run env:tunnel

cd mobile
adb devices -l                    # must show one device
adb uninstall com.ahmedmonib.eshop.dev || true      # remove old conflicting builds
adb uninstall com.ahmedmonib.eshop.internal || true
adb uninstall com.ahmedmonib.eshop || true

# Clean prebuild when switching variants:
APP_VARIANT=development npx expo prebuild --platform android --clean

# Build + install (either form):
APP_VARIANT=development npx expo run:android   # installs com.ahmedmonib.eshop.dev
# …or with Gradle directly:
cd android && ./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

> If the `ESHOP_ANDROID_*` vars are missing on a **release** build, the signing plugin fails fast
> with a clear "Missing release keystore configuration" error rather than producing an unsigned
> build. `expo prebuild` overwrites `android/` from `app.config.js` on each run.

---

## OTA updates (EAS Update — JavaScript only)

EAS Update ships **JavaScript-only** fixes over-the-air to installed builds, without a new APK/AAB.

```bash
# Publish a JS-only fix to the internal channel
eas update --channel internal --message "fix: <describe the fix>"

# Publish a JS-only fix to the production channel
eas update --channel production --message "fix: <describe the fix>"
```

Verify the update appears in the Expo dashboard, then relaunch the app on a device on that channel —
the launch-time hook (`src/hooks/useOtaUpdates.js` → `src/ota/otaService.js`) checks on cold launch
and either applies it seamlessly during the launch gate or surfaces an "Update ready" banner. On
dev/internal builds, open the `ota-debug` deep link (e.g. `vexflare-internal://ota-debug`) to
inspect the running channel / runtimeVersion / updateId and the stage log if an update never
applies. See `docs/deployment.md` for the full OTA flow and diagnostic runbook.

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
- The "JS-only" fix turns out to need a native change (works in the dev client but errors on the
  installed binary) — roll back, then ship the native change via `eas build`.
- A rollback is itself a new update pointing at the previous bundle, so users receive it on their
  next launch (it does not instantly remove the bad update). If a correct fix is ready quickly,
  prefer "fix forward" (publish a new good update) — it has the same reach as a rollback.

> ⚠️ OTAs only reach builds sharing the same `runtimeVersion` (currently `appVersion`). If you bump
> the version in `app.config.js`, you must do a fresh store build before OTA updates resume. OTAs
> cannot change native code — for native changes, use `eas build`.

> **First build per channel must be on EAS; after that, local builds work too.** The
> `expo-channel-name` request header is what maps a binary to its update branch. `app.config.js` now
> injects it at prebuild time via `updates.requestHeaders`, keyed off `APP_VARIANT` (`production` →
> `production`, `internal` → `internal`, `development` → none), so a local `APP_VARIANT=production`
> / `APP_VARIANT=internal` build carries its channel and receives OTAs — not EAS-only. But the
> **first** build for a brand-new channel must still run on EAS (`eas build`) — or the channel be
> created via the EAS CLI — to create the channel↔branch mapping on Expo's servers, since a purely
> local build never contacts them. Once the channel exists (as `production` and `internal` do),
> subsequent local builds on it receive updates automatically. (Before this fix, local builds
> omitted the channel entirely and silently received no OTAs — the 2026-07 incident.)

---

## Verify AAB versionCode/versionName (before a Play upload)

Play rejects re-used version codes:

```txt
Version code X has already been used
```

Verify the downloaded AAB before uploading.

### Install bundletool once

```bash
mkdir -p ~/tools && cd ~/tools
wget https://github.com/google/bundletool/releases/download/1.17.2/bundletool-all-1.17.2.jar
```

### Inspect the AAB

```bash
java -jar ~/tools/bundletool-all-1.17.2.jar dump manifest \
  --bundle ~/Downloads/<app>.aab --xpath /manifest/@android:versionCode
java -jar ~/tools/bundletool-all-1.17.2.jar dump manifest \
  --bundle ~/Downloads/<app>.aab --xpath /manifest/@android:versionName
```

If the values are wrong, fix `version` / `android.versionCode` in `app.config.js` and rebuild — EAS
regenerates the native project on every build, so there is no stale native cache to clear.

---

## Common pitfalls & quick fixes

### 1) `INSTALL_FAILED_UPDATE_INCOMPATIBLE: ... signatures do not match`

You're installing over an existing app with the same `applicationId` but a different signing key
(e.g. a local dev build vs the EAS-signed APK).

```bash
adb shell pm list packages | grep eshop
adb uninstall com.ahmedmonib.eshop.internal || true
adb uninstall com.ahmedmonib.eshop          || true
```

> The three variants (`…eshop` / `…eshop.internal` / `…eshop.dev`) have distinct application IDs and
> coexist fine — conflicts only happen when reinstalling the **same** id signed by a different key.

### 2) “Network error” on the internal APK

Internal builds ship with pinning **ON**, so first confirm the configured pins still appear in the
live chain (`npm -w mobile run cert:pins`) and match `eas.json`. Then clear app data and check
runtime logs — a pin failure logs `SSL pinning validation failed for <host>`:

```bash
adb shell pm clear com.ahmedmonib.eshop.internal
adb logcat -c
adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
```

### 3) `adb install` / push path

Always compute an **absolute** path to the downloaded artifact:

```bash
APK_ABS="$(realpath ~/Downloads/<downloaded>.apk)"
adb install -r "$APK_ABS"
```

If `install` hangs, reset ADB or switch to Wi-Fi ADB:

```bash
adb kill-server && adb start-server
# adb pair <ip>:<pair-port> && adb connect <ip>:<debug-port>
```

---

## Useful one-liners

```bash
adb shell pm list packages | grep eshop
adb shell dumpsys package com.ahmedmonib.eshop.internal | sed -n '1,60p'
adb kill-server && adb start-server && adb devices -l
```

---

## Mental model

- **Internal APK** = quick sideload via `eas build --profile internal`, pinning **ON**, id
  `com.ahmedmonib.eshop.internal`. Its distinct id means it coexists with prod.
- **Play AAB** = `eas build --profile production`, signed by the Expo-dashboard keystore, uploaded
  to the Console. Bump `versionCode` for every upload.
- **No Gradle flavors, no committed `android/`** — EAS prebuilds the native project from
  `app.config.js` on each build, and `APP_VARIANT` selects the id/scheme.
