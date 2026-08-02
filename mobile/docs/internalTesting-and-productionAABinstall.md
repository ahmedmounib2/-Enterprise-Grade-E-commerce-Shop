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
  certificate sync step any more, and **no `.env` file carries pin hashes**.
- On **every** build — EAS or local Gradle — `app.config.js` reads `mobile/ssl-pins.json` and
  injects `EXPO_PUBLIC_SSL_PINNING_HASHES` and `EXPO_PUBLIC_SSL_PINNING_EXPIRES` into the Expo
  config's `extra` block, which is baked into the JS bundle at prebuild. `eas.json` supplies only
  `APP_VARIANT` and the public runtime env.

Nothing is packaged as an asset, so there is no `assets/*.cer` to look for in the APK or AAB.

> **Verify from the EAS build page that `EXPO_PUBLIC_SSL_PINNING_HASHES` is _absent_ from
> Environment variables.** That is the correct state: the pins come from `ssl-pins.json` via
> `app.config.js`, not from the profile's `env`. Seeing it listed means someone reintroduced a
> second source, which is exactly the drift that once left local Gradle builds unpinned while EAS
> builds were pinned. The one exception is `internal-pinfail`, whose whole purpose is to override
> the pins with invalid ones.
>
> To check what a build will actually use, resolve the config the same way the build does:
>
> ```bash
> cd mobile && APP_VARIANT=internal node -e "
>   console.log(require('./app.config.js').expo.extra.EXPO_PUBLIC_SSL_PINNING_HASHES);
> "; cd ..
> ```

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

The `internal` EAS profile injects the public runtime env (`EXPO_PUBLIC_API_BASE_URL`,
`EXPO_PUBLIC_WEB_ASSET_ORIGIN`, and the `EXPO_PUBLIC_FEATURE_*` flags), and `app.config.js` adds the
pins from `ssl-pins.json`, so the app talks to the production API over a pinned connection and
renders categories/product images.

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

Production is pinned from `mobile/ssl-pins.json`, the same file the internal build uses — there is
nothing to switch on per profile. No file secret and no `eas-build-post-install` hook is involved,
because the pins are plain strings that live in the repository (they are hashes of public keys, not
secrets).

**Rollback** is one line in the `production` profile's `env` in `mobile/eas.json`:

```json
"MOBILE_SSL_PINNING_DISABLED": "1"
```

`app.config.js` then injects no pins and the app runs unpinned. It requires a new store build: a
binary that cannot reach the API also cannot download an OTA update, so pinning cannot be switched
off over the air.

The last-resort net for builds already in the field is `expirationDate` in `ssl-pins.json`
(currently `2027-08-01`), injected as `EXPO_PUBLIC_SSL_PINNING_EXPIRES` — on that date pin
validation disables itself without user action. The `DEFAULT_PIN_EXPIRATION_DATE` constant in
`packages/api-client/sslPinningAdapter.native.js` is only the fallback used when that value is
missing.

A fully **local** production build needs no pinning setup at all: `app.config.js` resolves the pins
identically for Gradle. Run `npm -w mobile run env:production` for the API URL and feature flags,
then `bundleRelease` (see `build-and-install.md` → "Manual build").

---

## Local env profiles

`use-env.sh` swaps `mobile/.env` (API base URL, feature flags, theme defaults). It does **not**
touch pinning — no `.env` file contains pin hashes. All profiles already exist as npm scripts:

```bash
npm -w mobile run env:internal     # production API — pinned, because app.config.js supplies the pins
npm -w mobile run env:production   # production API — same pins, production app id
npm -w mobile run env:tunnel       # ngrok API — the daily dev client
npm -w mobile run env:emu          # emulator (10.0.2.2)
npm -w mobile run env:lan          # physical device on the LAN
```

These local `.env` files govern **local** builds and the dev client; an EAS build ignores them and
uses the matching profile's `env` block in `mobile/eas.json`. Either way the pins come from
`ssl-pins.json`, which is what makes a local Gradle build and an EAS build of the same profile
byte-identical in pinning configuration (asserted by
`mobile/src/tests/config/sslPinResolution.test.js`).

> **`cert:pins` reads the API host from the active `mobile/.env`.** Run it after `env:emu` and it
> will try to read a certificate chain from `10.0.2.2` and fail. Switch to `env:internal` or
> `env:production` first, or pass the host explicitly:
>
> ```bash
> npm -w mobile run cert:pins -- --host api.vexflare.com
> ```
>
> On a **fresh clone** this matters more than it looks: `mobile/.env` is gitignored, so it does not
> exist until you run an `env:*` script, and `cert:pins` exits 1 with
> `No host provided. Pass --host <api-host> or set EXPO_PUBLIC_API_BASE_URL.` until then.

> The emulator, LAN and tunnel profiles need no pinning setting at all: the app skips pinning
> automatically whenever the API host is `localhost`, a raw IPv4 address, or a known tunnel domain.

---

## Verifications

> For the complete reproducible procedures — including proxy setup, expected console and logcat
> output, pass/fail criteria and troubleshooting — follow [`ssl-pinning.md`](./ssl-pinning.md) §4.
> The checks below are the short form.

- **Are the pins in the build?** There is no packaged asset to inspect, and the EAS build page
  should show `EXPO_PUBLIC_SSL_PINNING_HASHES` as **absent** (see the callout above). Resolve
  `app.config.js` instead — that is the same path the build takes.

- **Is pinning actually enforced?** Build the `internal-pinfail` profile
  (`npm -w mobile run eas:build:pinfail`) and install it. It is byte-for-byte the `internal` profile
  except that `EXPO_PUBLIC_SSL_PINNING_HASHES` in its `env` block overrides the canonical pins with
  two deliberately non-matching hashes, so:

  | Build              | Expected                                                                                 |
  | ------------------ | ---------------------------------------------------------------------------------------- |
  | `internal`         | app works normally — the shipped pins match the live chain                               |
  | `internal-pinfail` | **every** API call fails; `SSL pinning validation failed for api.vexflare.com` in logcat |

  Both builds share the `com.ahmedmonib.eshop.internal` application ID, so install one at a time.
  This is the differential test to trust: it needs no proxy and no CA, and it isolates the pinning
  comparison as the only variable. The same build can be produced locally with Gradle —
  `npm -w mobile run env:internal-pinfail` generates `mobile/.env` from this very profile; see
  [`build-and-install.md`](./build-and-install.md) → "Pin-enforcement regression APK".

  > **The invalid hashes must still be well-formed.** Both are 44-character base64 strings decoding
  > to exactly 32 bytes, like any real SHA-256 SPKI digest — they simply match nothing in the live
  > chain. Do not "simplify" them to a short placeholder: a malformed pin can make the native
  > library reject the configuration outright rather than accept it and fail the comparison, and an
  > app that fails for that reason looks identical from the outside. That would turn this test — the
  > one check whose entire job is to catch pinning having gone inert — into a false pass.

  > **A MITM-proxy test cannot serve as the control on a modern Android build.** Apps targeting API
  > 24+ do not trust user-installed CA certificates — only the system store — and this app targets
  > API 36 with no `network_security_config.xml`. So installing a mitmproxy/Charles CA through
  > Settings makes _every_ build reject the intercepted connection, pinned or not, with
  > `ERR_CERT_AUTHORITY_INVALID`. That failure comes from Android's CA validation, several layers
  > below the pinning check, so a "blocked" result proves nothing about pinning.
  >
  > **That is what the `internal-mitm` / `internal-mitm-nopin` profiles are for.** They set
  > `ANDROID_TRUST_USER_CAS=1`, which makes `plugins/withUserCaTrust.js` emit a
  > `network_security_config.xml` trusting the user store, so the proxy CA becomes genuinely valid
  > and the pinner is reached. Running the pair — identical builds, identical proxy, pins present
  > versus absent — is the actual MITM proof; see [`ssl-pinning.md`](./ssl-pinning.md) §4.3 and
  > §4.4. Neither profile is shippable, and the plugin throws if the flag is combined with
  > `APP_VARIANT=production`.
  >
  > Note also that `chromium … net_error -202` lines in logcat come from the WebView stack, not from
  > the OkHttp client the API uses — an OkHttp pin rejection surfaces as
  > `SSLPeerUnverifiedException: Certificate pinning failure!` plus the JS listener line above.

- **Runtime logs** — tap "Login" and watch for TLS/pinning failures:

  ```bash
  adb logcat -c
  adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
  ```

- **"Network error" on every request** → none of the configured pins matches the live chain, or
  initialization failed. Run `npm -w mobile run cert:pins -- --check`, which compares
  `mobile/ssl-pins.json` against the live chain and exits 1 if no configured pin appears in it;
  `adb logcat` will show `SSL pinning validation failed for <host>`. Then clear app data and retry:

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

- No cert-fetching step: pins are strings in `mobile/ssl-pins.json`, the single canonical source for
  EAS and local Gradle alike. Regenerate with `cert:pins -- --write`, gate releases with
  `cert:pins -- --check`.
- **Internal** = pinned sideload APK, `com.ahmedmonib.eshop.internal` — the device-validation
  vehicle ahead of each store release.
- **Production** = pinned AAB to Play, from the same pins as internal. Turning pinning **off** is
  `MOBILE_SSL_PINNING_DISABLED=1` in the profile's `env` plus a new build; it cannot be changed over
  the air.
- Install with an **absolute** APK path; uninstall a conflicting package first if the signing key
  differs.
- Verification here is **Android-only**. See [`ssl-pinning.md`](./ssl-pinning.md) §0.4 for what that
  does and does not cover on iOS.
