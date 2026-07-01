# Internal testing APK & production AAB — install & SSL pinning

Practical reference for building the **internal sideload APK** and the **production Play Console
AAB**, installing them on a device, and keeping TLS certificate pinning working. It complements
[`build-and-install.md`](./build-and-install.md) (the EAS command reference) and
[`../../docs/deployment.md`](../../docs/deployment.md) (the full runbook); this page focuses on the
**certificate / pinning** mechanics and the **on-device install** gotchas.

Three things that commonly trip people up:

1. **Order matters:** `cert:pull` → `env:<profile>` (which syncs the cert into place) → build.
2. **Path matters:** install the APK with an **absolute** path, or `adb` fails with `cannot stat`.
3. **Pinning must be OFF for internal builds** — otherwise a cert/env mismatch surfaces at runtime
   as a "Network error".

> **Managed-workflow note.** EAS regenerates `android/` from `app.config.js` on every build, and the
> `withSslPinningCerts` config plugin copies `mobile/certs/*.cer` into the app assets at prebuild.
> The `internal` EAS profile ships with pinning **off** (no cert needed); production pinning needs
> the cert made available to EAS — see [Production pinning on EAS](#production-pinning-on-eas).

---

## How the certificate gets into a build

- `npm -w mobile run cert:pull -- --host api.vexflare.com --name eshop_api` → saves the DER leaf
  certificate to `mobile/certs/eshop_api.cer` from the live server. The `.cer` is **git-ignored** —
  a fetched artifact, not committed.
- `npm -w mobile run env:<profile>` → runs `scripts/use-env.sh`, which:
  - copies `mobile/.env.<profile>` → `mobile/.env`,
  - runs `scripts/sync-ssl-certs.js` (copies every `.cer` into the local
    `android/app/src/main/assets/`, the correct location for `react-native-ssl-pinning` on Android),
  - validates that the names listed in `EXPO_PUBLIC_SSL_PINNING_CERTS` exist under `mobile/certs`.
- On an **EAS** build, the `withSslPinningCerts` config plugin performs the equivalent copy at
  prebuild — but only if a `.cer` is present in the build (it is not on EAS unless you provide it;
  see below).

When present, the packaged cert lands at `assets/eshop_api.cer` in an APK and
`base/assets/eshop_api.cer` in an AAB.

---

## Canonical flows

### A) Internal testing APK (pinning OFF)

```bash
# 1) (optional) refresh the cert — harmless even with pinning off
npm -w mobile run cert:pull -- --host api.vexflare.com --name eshop_api

# 2) Build the APK on EAS (internal profile → com.ahmedmonib.eshop.internal, pinning off).
#    EAS prebuilds android/ from app.config.js and signs with the dashboard keystore.
eas build --platform android --profile internal

# 3) Download the APK from the EAS dashboard, then install with an ABSOLUTE path:
APK_ABS="$(realpath ~/Downloads/<downloaded>.apk)"
adb devices -l
adb uninstall com.ahmedmonib.eshop.internal || true   # avoid a signature-mismatch on reinstall
adb install -r "$APK_ABS"

# WSL, APK sitting in the Windows Downloads folder:
# adb install -r "/mnt/c/Users/<you>/Downloads/<downloaded>.apk"
```

> If install hangs: `adb kill-server && adb start-server`, switch to Wi-Fi ADB, or re-plug USB. If
> it fails with `INSTALL_FAILED_UPDATE_INCOMPATIBLE`, a build with the same id but a different
> signing key is already installed — uninstall it first (shown above).

The `internal` EAS profile runs with pinning off (`EXPO_PUBLIC_SSL_PINNING_CERTS` unset) and injects
the public runtime env (`EXPO_PUBLIC_API_BASE_URL`, `EXPO_PUBLIC_WEB_ASSET_ORIGIN`, and the
`EXPO_PUBLIC_FEATURE_*` flags), so the app talks to the production API and renders
categories/product images.

### B) Production AAB (pinning ON)

```bash
# 1) Refresh the cert
npm -w mobile run cert:pull -- --host api.vexflare.com --name eshop_api

# 2) Build the AAB on EAS (production profile → com.ahmedmonib.eshop, signed by the dashboard keystore)
eas build --platform android --profile production

# 3) Download the AAB from the EAS dashboard and upload it to the Play Console.
#    The R8 mapping.txt is attached to the same build (Artifacts panel) — keep it for deobfuscation.
```

#### Production pinning on EAS

`mobile/certs/*.cer` is git-ignored, so EAS clones the repo without it and the pinning plugin no-ops
— a production AAB built on EAS ships **without** the pinned cert unless you provide it. To pin in
production, either:

- add an `eas-build-post-install` hook that runs `cert:pull` before prebuild, or
- upload the cert as an EAS **file** secret and have the plugin/hook place it under `mobile/certs/`.

For a fully **local** production build, `npm -w mobile run env:production` copies the cert into
assets before you run the Gradle `bundleRelease` (see `build-and-install.md` → "Manual build").

---

## Local env profiles

`use-env.sh` swaps `mobile/.env` (API base URL, pinning certs, feature flags). All profiles already
exist as npm scripts:

```bash
npm -w mobile run env:internal     # production API, pinning OFF — for on-device internal testing
npm -w mobile run env:production   # production API, pinning ON  — EXPO_PUBLIC_SSL_PINNING_CERTS=eshop_api
npm -w mobile run env:tunnel       # ngrok API — the daily dev client
npm -w mobile run env:emu          # emulator (10.0.2.2)
npm -w mobile run env:lan          # physical device on the LAN
```

`mobile/.env.internal` keeps pinning off (a blank cert list disables pinning):

```dotenv
EXPO_PUBLIC_API_BASE_URL=https://api.vexflare.com
EXPO_PUBLIC_SSL_PINNING_CERTS=          # blank = pinning OFF
```

> `sync-ssl-certs.js` needs no changes — it already copies `.cer` files into `assets/`, the correct
> Android location.

---

## Verifications

- **Is the cert packaged?** (only relevant when pinning is on)

  ```bash
  unzip -l ~/Downloads/<downloaded>.apk | grep -i 'assets/eshop_api\.cer'
  unzip -l ~/Downloads/<downloaded>.aab | grep -i 'assets/eshop_api\.cer'
  ```

- **Runtime logs** — tap "Login" and watch for TLS/pinning failures:

  ```bash
  adb logcat -c
  adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
  ```

- **"Network error" on the internal APK** → the build has pinning on but no matching cert. Confirm
  it was built from the `internal` profile (pinning off), then clear app data and retry:

  ```bash
  adb shell pm clear com.ahmedmonib.eshop.internal
  ```

---

## "cannot stat …apk" when installing

`adb push` / `adb install` with a **relative** path only resolves from the directory the artifact
lives in, so running it from the repo root fails with `cannot stat`. Always compute an absolute path
first (`APK_ABS="$(realpath …)"`) — then it works from anywhere, including installing a
Windows-downloaded APK from WSL via `/mnt/c/Users/<you>/Downloads/…`.

---

## TL;DR

- Order: **`cert:pull` → `env:<profile>` (syncs the cert) → `eas build`**.
- **Internal** = pinning OFF (`env:internal`), quick sideload APK, `com.ahmedmonib.eshop.internal`.
- **Production** = pinning ON, AAB to Play; on EAS the cert must be provided (post-install hook or
  file secret) or pinning silently no-ops.
- Install with an **absolute** APK path; uninstall a conflicting package first if the signing key
  differs.
