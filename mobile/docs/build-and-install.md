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

> Internal builds run with SSL pinning **OFF** (`EXPO_PUBLIC_SSL_PINNING_CERTS` empty), so no
> certificate is needed. See `docs/deployment.md` → "SSL certificate pinning".
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
     flags)
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


cd mobile
# Wipe Metro's cache
npx expo start --clear &
sleep 5 && kill %1   # Start Metro to clear its cache, then stop it

APP_VARIANT=production npx expo prebuild --platform android --clean
cd android && ./gradlew clean bundleRelease
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
it downloads and applies on the next launch (the launch-time hook in `src/hooks/useOTAUpdates.js`).

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

> **First build per channel must be on EAS.** The very first build for a given channel (e.g.,
> `internal` or `production`) must run on EAS (`eas build`) to create the update branch on Expo's
> servers. After that first EAS build, the branch exists permanently. Subsequent builds — whether on
> EAS or built locally with `expo prebuild` + `./gradlew` — will receive OTA updates automatically
> on that channel.

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

Internal builds ship with pinning **OFF**. Clear app data and check runtime logs:

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

- **Internal APK** = quick sideload via `eas build --profile internal`, pinning **OFF**, id
  `com.ahmedmonib.eshop.internal`. Its distinct id means it coexists with prod.
- **Play AAB** = `eas build --profile production`, signed by the Expo-dashboard keystore, uploaded
  to the Console. Bump `versionCode` for every upload.
- **No Gradle flavors, no committed `android/`** — EAS prebuilds the native project from
  `app.config.js` on each build, and `APP_VARIANT` selects the id/scheme.
