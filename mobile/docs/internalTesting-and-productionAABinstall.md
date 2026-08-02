# Internal testing APK & production AAB — install & SSL pinning

Practical reference for building the **internal sideload APK** and the **production Play Console
AAB**, installing them on a device, and keeping TLS certificate pinning working. It complements
[`build-and-install.md`](./build-and-install.md) (the EAS command reference) and
[`../../docs/deployment.md`](../../docs/deployment.md) (the full runbook); this page focuses on the
**certificate / pinning** mechanics and the **on-device install** gotchas.

Three things that commonly trip people up:

1. **Pins are strings, not files.** Nothing has to be fetched or copied before a build. Their single
   canonical source is `mobile/ssl-pins.json`, which `app.config.js` injects into `extra` for both
   EAS and local Gradle builds. Full reference: [`ssl-pinning.md`](./ssl-pinning.md).
2. **Path matters:** install the APK with an **absolute** path, or `adb` fails with `cannot stat`.
3. **Internal builds pin too.** That is deliberate: the internal APK is the vehicle for validating
   pinning on a real device ahead of each store release. Production pinning is enabled and has been
   verified end to end from Google Play (see [`ssl-pinning.md`](./ssl-pinning.md) §10).

> **Managed-workflow note.** EAS regenerates `android/` from `app.config.js` on every build. Because
> the pins are plain environment values rather than binary assets, EAS needs no file secret and no
> `eas-build-post-install` hook — a fresh clone builds a correctly pinned app.

---

## How the pins get into a build

- `npm -w mobile run cert:pins` reads the live certificate chain for the API host and prints the
  base64 SHA-256 SPKI hash of every certificate in it, plus the exact comma-separated value to use.
  It **excludes the leaf on purpose**: Railway renews the Let's Encrypt certificate about every 60
  days with a new key, so a leaf pin would lock out every installed app at the next renewal. The
  intermediates and root are pinned instead.
- `npm -w mobile run env:<profile>` copies `mobile/.env.<profile>` → `mobile/.env`. There is no
  certificate sync step any more.
- On an **EAS** build, `EXPO_PUBLIC_SSL_PINNING_HASHES` comes from the profile's `env` block in
  `mobile/eas.json` and is baked into the JS bundle at prebuild.

Nothing is packaged as an asset, so there is no `assets/*.cer` to look for in the APK or AAB. Verify
instead from the EAS build page (**Build details → Environment variables**) that
`EXPO_PUBLIC_SSL_PINNING_HASHES` is present.

---

## Canonical flows

### A) Internal testing APK (pinning ON)

```bash
# 1) (optional) confirm the configured pins still appear in the live chain
npm -w mobile run cert:pins

# 2) Build the APK on EAS (internal profile → com.ahmedmonib.eshop.internal, pinning ON).
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

The `internal` EAS profile sets `EXPO_PUBLIC_SSL_PINNING_HASHES` and injects the public runtime env
(`EXPO_PUBLIC_API_BASE_URL`, `EXPO_PUBLIC_WEB_ASSET_ORIGIN`, and the `EXPO_PUBLIC_FEATURE_*` flags),
so the app talks to the production API over a pinned connection and renders categories/product
images.

### B) Production AAB

```bash
# 1) Confirm the configured pins still appear in the live chain
npm -w mobile run cert:pins

# 2) Build the AAB on EAS (production profile → com.ahmedmonib.eshop, signed by the dashboard keystore)
eas build --platform android --profile production

# 3) Download the AAB from the EAS dashboard and upload it to the Play Console.
#    The R8 mapping.txt is attached to the same build (Artifacts panel) — keep it for deobfuscation.
```

#### Production pinning on EAS

Pinning in production is controlled by one line: `EXPO_PUBLIC_SSL_PINNING_HASHES` in the
`production` profile's `env` in `mobile/eas.json`. Present → pinned; absent → unpinned. No file
secret and no `eas-build-post-install` hook is involved, because the pins are plain strings that
live in the repository (they are hashes of public keys, not secrets).

Removing that line and rebuilding is also the **rollback**. Note it requires a new store build: a
binary that cannot reach the API also cannot download an OTA update, so pinning cannot be switched
off over the air. The `DEFAULT_PIN_EXPIRATION_DATE` constant in
`packages/api-client/sslPinningAdapter.native.js` is the last-resort net for builds already in the
field — on that date pin validation disables itself.

For a fully **local** production build, `npm -w mobile run env:production` puts the same hashes into
`mobile/.env` before you run the Gradle `bundleRelease` (see `build-and-install.md` → "Manual
build").

---

## Local env profiles

`use-env.sh` swaps `mobile/.env` (API base URL, pin hashes, feature flags). All profiles already
exist as npm scripts:

```bash
npm -w mobile run env:internal     # production API, pinning OFF locally — for dev-client work
npm -w mobile run env:production   # production API, pinning ON — EXPO_PUBLIC_SSL_PINNING_HASHES set
npm -w mobile run env:tunnel       # ngrok API — the daily dev client
npm -w mobile run env:emu          # emulator (10.0.2.2)
npm -w mobile run env:lan          # physical device on the LAN
```

These local `.env` files govern **local** builds and the dev client. An EAS build ignores them and
uses the matching profile's `env` block in `mobile/eas.json`, where the `internal` profile _does_
set the pins.

`mobile/.env.internal` keeps pinning off locally (a blank hash list disables pinning):

```dotenv
EXPO_PUBLIC_API_BASE_URL=https://api.vexflare.com
EXPO_PUBLIC_SSL_PINNING_HASHES=         # blank = pinning OFF
```

> The emulator, LAN and tunnel profiles need no pinning setting at all: the app skips pinning
> automatically whenever the API host is `localhost`, a raw IPv4 address, or a known tunnel domain.

---

## Verifications

> For the complete reproducible procedures — including proxy setup, expected console and logcat
> output, pass/fail criteria and troubleshooting — follow [`ssl-pinning.md`](./ssl-pinning.md) §4.
> The checks below are the short form.

- **Are the pins in the build?** Open the EAS build page → **Build details → Environment variables**
  and confirm `EXPO_PUBLIC_SSL_PINNING_HASHES` is listed with the expected comma-separated hashes.
  There is no packaged asset to inspect.

- **Is pinning actually enforced?** Build the `internal-pinfail` profile
  (`npm -w mobile run eas:build:pinfail`) and install it. It is byte-for-byte the `internal` profile
  except that `EXPO_PUBLIC_SSL_PINNING_HASHES` holds two deliberately non-matching hashes, so:

  | Build              | Expected                                                                                 |
  | ------------------ | ---------------------------------------------------------------------------------------- |
  | `internal`         | app works normally — the shipped pins match the live chain                               |
  | `internal-pinfail` | **every** API call fails; `SSL pinning validation failed for api.vexflare.com` in logcat |

  Both builds share the `com.ahmedmonib.eshop.internal` application ID, so install one at a time.
  This is the differential test to trust: it needs no proxy and no CA, and it isolates the pinning
  comparison as the only variable. The same build can be produced locally with Gradle —
  `npm -w mobile run env:internal-pinfail` generates `mobile/.env` from this very profile; see
  [`build-and-install.md`](./build-and-install.md) → "Pin-enforcement regression APK".

  > **A MITM-proxy test cannot serve as the control on a modern Android build.** Apps targeting API
  > 24+ do not trust user-installed CA certificates — only the system store — and this app targets
  > API 36 with no `network_security_config.xml`. So installing a mitmproxy/Charles CA through
  > Settings makes _every_ build reject the intercepted connection, pinned or not, with
  > `ERR_CERT_AUTHORITY_INVALID`. That failure comes from Android's CA validation, several layers
  > below the pinning check, so a "blocked" result proves nothing about pinning. Making the proxy
  > test meaningful requires installing the CA into the **system** store (rooted device or an
  > emulator with a writable system partition), or shipping a throwaway build whose network security
  > config trusts user CAs. Note also that `chromium … net_error -202` lines in logcat come from the
  > WebView stack, not from the OkHttp client the API uses — an OkHttp pin rejection surfaces as
  > `SSLPeerUnverifiedException: Certificate pinning failure!` plus the JS listener line above.

- **Runtime logs** — tap "Login" and watch for TLS/pinning failures:

  ```bash
  adb logcat -c
  adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
  ```

- **"Network error" on every request** → none of the configured pins matches the live chain, or
  initialization failed. Re-run `npm -w mobile run cert:pins` and compare with the values in
  `mobile/eas.json`; `adb logcat` will show `SSL pinning validation failed for <host>`. Then clear
  app data and retry:

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

- No cert-fetching step: pins are strings in `mobile/eas.json`. Regenerate with `cert:pins`.
- **Internal** = pinned sideload APK, `com.ahmedmonib.eshop.internal` — the device-validation
  vehicle ahead of each store release.
- **Production** = pinned AAB to Play. Turning pinning on or off is one line in `eas.json` plus a
  new build; it cannot be changed over the air.
- Install with an **absolute** APK path; uninstall a conflicting package first if the signing key
  differs.
