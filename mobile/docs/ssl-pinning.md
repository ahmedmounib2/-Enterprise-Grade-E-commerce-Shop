# Mobile SSL certificate pinning — operational runbook

Canonical reference for how pinning works, how to verify it, and how to rotate certificates.

Written to be followed from a **fresh clone with no prior context**. Every command is given in full.
Where a step depends on something that was not captured during the original verification, that is
called out explicitly rather than guessed.

- [0. Environment preparation](#0-environment-preparation)
- [1. Architecture](#1-architecture)
- [2. Build matrix](#2-build-matrix)
- [3. Proxy setup (Charles / mitmproxy)](#3-proxy-setup-charles--mitmproxy)
- [4. Test procedures](#4-test-procedures)
- [5. Production release verification](#5-production-release-verification)
  - [Release verification checklist (< 5 min)](#release-verification-checklist--5-minutes-before-uploading)
  - [Post-upload verification checklist](#post-upload-verification-checklist)
- [6. Certificate rotation](#6-certificate-rotation)
- [7. Maintenance notes](#7-maintenance-notes)
- [8. Verification command reference](#8-verification-command-reference)
- [9. Troubleshooting](#9-troubleshooting)
- [10. Final acceptance checklist](#10-final-acceptance-checklist)

---

## 0. Environment preparation

### 0.1 Required software

| Tool                           | Version                                 | Notes                                                                       |
| ------------------------------ | --------------------------------------- | --------------------------------------------------------------------------- |
| Node.js                        | 20.x                                    | Verified on v20.19.4. Matches the Vercel build baseline.                    |
| npm                            | 10.x                                    | Workspaces are required; do not use yarn/pnpm here.                         |
| Java JDK                       | **17**                                  | Required by Gradle 8.14.3 / AGP 8.11. Java 21 is not supported by this AGP. |
| Android SDK                    | **platform 36**, build-tools **36.0.0** | The app targets API 36. Installing only 35 will fail the build.             |
| Android platform-tools         | any recent                              | Provides `adb`.                                                             |
| Expo CLI                       | bundled                                 | Invoked as `npx expo`; not installed globally.                              |
| EAS CLI                        | `>= 12.0.0`                             | Only needed for cloud builds. `npm i -g eas-cli`.                           |
| Charles Proxy **or** mitmproxy | any recent                              | Only for the MITM procedures (4.3 / 4.4).                                   |

Install the Android pieces (see `mobile/custom-dev-setup.md` for full first-time setup):

```bash
yes | sdkmanager --licenses
yes | sdkmanager "platform-tools" "platforms;android-36" "build-tools;36.0.0" "ndk;27.1.12297006"
```

Verify the toolchain before anything else:

```bash
node -v                  # v20.x
npm -v                   # 10.x
java -version            # openjdk version "17.x"
adb version              # Android Debug Bridge
npx expo --version
eas whoami               # only if you intend to build on EAS
```

### 0.2 Required environment variables

**Shell variables** — only the release signing set. Nothing about pinning is configured by hand.

| Variable                            | Needed for                  | Source                      |
| ----------------------------------- | --------------------------- | --------------------------- |
| `ESHOP_ANDROID_KEYSTORE`            | any **local release** build | Path to the upload keystore |
| `ESHOP_ANDROID_KEYSTORE_PASSWORD`   | any local release build     | Bitwarden                   |
| `ESHOP_ANDROID_KEY_ALIAS`           | any local release build     | Bitwarden                   |
| `ESHOP_ANDROID_KEY_PASSWORD`        | any local release build     | Bitwarden                   |
| `ANDROID_HOME` / `ANDROID_SDK_ROOT` | Gradle                      | Your SDK install            |
| `JAVA_HOME`                         | Gradle                      | The JDK 17 install          |

Omitting the `ESHOP_ANDROID_*` set on a release build fails fast with
`Missing release keystore configuration` — by design, so an unsigned release is never produced
silently. They are **not** needed for `assembleDebug` or for EAS builds (EAS injects the dashboard
keystore).

**Build-profile variables** — set by `eas.json`, never by you:

| Variable                         | Meaning                                                                       |
| -------------------------------- | ----------------------------------------------------------------------------- |
| `APP_VARIANT`                    | `production` \| `internal` \| `development`. Selects app id, scheme, channel. |
| `ANDROID_TRUST_USER_CAS=1`       | Diagnostic only. Trusts user-installed CAs (`internal-mitm*`).                |
| `MOBILE_SSL_PINNING_DISABLED=1`  | Diagnostic only. Ships no pins (`internal-mitm-nopin`).                       |
| `EXPO_PUBLIC_SSL_PINNING_HASHES` | Diagnostic override only (`internal-pinfail`). **Never set by hand.**         |

> **Do not put pin hashes in any `.env` file.** Their canonical source is `mobile/ssl-pins.json`;
> `app.config.js` injects them. Setting them by hand reintroduces exactly the drift that once left
> local Gradle builds unpinned while EAS builds were pinned.

### 0.3 Prerequisites before running any test

```bash
git clone <repo> && cd <repo>
npm install                       # workspaces: installs backend, frontend, mobile, shared, packages
npm -w mobile test                # must be green before you start
npm -w mobile run cert:pins       # must show pins matching mobile/ssl-pins.json
```

For any on-device procedure:

- A physical Android device with **USB debugging** enabled (Settings → About phone → tap Build
  number 7×, then Developer options → USB debugging).
- `adb devices -l` lists it as `device` (not `unauthorized` — accept the RSA prompt on the phone).
- The device is **not** already running another `com.ahmedmonib.eshop.internal*` build. All four
  `internal*` profiles share one application id, so install one at a time.

---

## 1. Architecture

### Implementation

Pinning uses **`react-native-ssl-public-key-pinning`** — OkHttp's `CertificatePinner` on Android,
TrustKit on iOS. It installs pinning **globally at the platform networking layer**, so every request
made through standard React Native networking is covered, not just axios calls.

It replaced `react-native-ssl-pinning@1.6.0`, which was a legacy bridge module and the only
dependency blocking the New Architecture. The replacement ships a TurboModule codegen spec
(`RNSslPublicKeyPinningSpec`) and resolves through `TurboModuleRegistry.get` when
`global.__turboModuleProxy` exists, falling back to `NativeModules` otherwise — so it works under
both architectures.

### The adapter boundary

`packages/api-client/sslPinningAdapter.native.js` is the compatibility boundary. Nothing else in the
app imports the pinning library, so swapping libraries again is a change to that one file.

Metro selects it by platform extension: `./sslPinningAdapter` resolves to
`sslPinningAdapter.native.js` in React Native and `sslPinningAdapter.js` (a throwing stub) on web
and under Node.

Because the library pins globally rather than performing the request itself, the adapter is a **gate
that delegates**:

```
api.get('/orders')
  → axios dispatch → api.defaults.adapter  (the pinning adapter)
      → await initializeSslPinning({ host: { publicKeyHashes, expirationDate } })   ← once
      → stockAdapter(config)               (axios' own XHR adapter)
          → React Native networking → OkHttp (pinned) / URLSession (TrustKit)
```

Two details that matter:

- The stock adapter is resolved from the **global** `axios.defaults` _before_ the swap. Reading the
  instance default instead would make the adapter call itself.
- The `await` is what closes a fail-open window. Native initialization is async; without the gate a
  request issued between app mount and initialization would go out unpinned.

### CertificatePinner flow, and trust manager vs pinning

**This distinction is the single most important thing to understand before testing.** Two rounds of
verification during this migration produced inconclusive results because of it.

```
TLS handshake
  1. Platform TrustManager validates the chain against the device trust store
        ✗ fails → SSLHandshakeException: "Trust anchor for certification path not found"
                  ← ANDROID rejected it. Pinning never ran.
        ✓ passes ↓
  2. OkHttp CertificatePinner compares the SPKI hashes of the accepted chain to our pins
        ✗ no match → SSLPeerUnverifiedException: "Certificate pinning failure!"
                     ← PINNING rejected it.
        ✓ match → request proceeds
```

**Pinning runs only after the trust manager accepts the chain.** A certificate the device does not
trust is rejected at step 1 and never reaches step 2 — so a proxy CA installed through Android
Settings cannot test pinning at all (see [section 3](#3-proxy-setup-charles--mitmproxy)).

### Where the pin hashes come from

**`mobile/ssl-pins.json` is the single canonical source.** Nothing else in the repository holds the
hashes — enforced by `src/tests/config/sslPinResolution.test.js`.

```
mobile/ssl-pins.json                    ← edit here, and only here
   │
   └─ mobile/app.config.js  resolveSslPinningHashes()
         │   overrides (diagnostic profiles only):
         │     MOBILE_SSL_PINNING_DISABLED=1        → ship no pins
         │     EXPO_PUBLIC_SSL_PINNING_HASHES=<...> → override with these
         │
         └─ expo.extra.EXPO_PUBLIC_SSL_PINNING_HASHES
            expo.extra.EXPO_PUBLIC_SSL_PINNING_EXPIRES
               │
               └─ packaged by Expo at build time (EAS and local Gradle alike),
                  read at runtime via Constants.expoConfig.extra
```

> Because the app reads these through `Constants` rather than an inlined `process.env` value, the
> hashes **do not appear in `assets/index.android.bundle`**. Grepping the JS bundle for a hash is
> therefore not a valid check. Verify by resolving `app.config.js` for the target profile
> ([Phase A](#phase-a--build-time-verification-run-before-every-release)) rather than by unpacking
> the artefact — Expo's packaging layout is an internal detail that has moved between SDK versions.

Feeding both build paths from `app.config.js` is what makes a local Gradle build and an EAS build
identical. Previously `eas.json` carried its own copy of the hashes while `.env` did not, so local
internal builds shipped **unpinned** while EAS internal builds shipped pinned.

**The hashes are not secrets.** They are SHA-256 hashes of _public_ keys, computable by anyone who
connects to the host. They belong committed in `ssl-pins.json` — never in EAS Secrets, Bitwarden,
Railway variables or GitHub Secrets, where they would only be harder to rotate.

### How the app consumes them

`mobile/src/auth/AuthProvider.js`, in its mount effect:

```js
const pinningHashes = (readStringEnv("EXPO_PUBLIC_SSL_PINNING_HASHES") || "")
  .split(",")
  .map((v) => v.trim())
  .filter(Boolean);

if (pinningHashes.length) {
  if (shouldSkipPinningForHost(hostname)) return; // localhost / IP literal / ngrok / trycloudflare
  configureSslPinning({
    publicKeyHashes: pinningHashes,
    hostname,
    timeout: 15000,
    expirationDate: readStringEnv("EXPO_PUBLIC_SSL_PINNING_EXPIRES"),
  });
}
```

`readStringEnv` (`src/config/env.js`) reads `process.env` first, then `Constants.expoConfig.extra`.
A release build only reliably exposes `extra`, so reading raw `process.env` here would silently
yield nothing in production.

`configureSslPinning` returns `true` when pinning is active and `false` when it was intentionally
skipped, and **throws** when pinning was required but could not be installed. That throw is
deliberately not caught — swallowing it is what previously let the app continue unpinned.

There is no way to disable pinning at runtime. An earlier implementation restored the unpinned
adapter and replayed the request whenever a pinned request produced a network error, which meant any
failure — including a genuine pin rejection — silently downgraded the connection for the rest of the
process. Pinning is now install-once.

### Why the MITM builds exist

`internal-mitm` and `internal-mitm-nopin` are the only way to prove pinning rejects a certificate
the device **accepts**. They set `ANDROID_TRUST_USER_CAS=1`, which activates
`mobile/plugins/withUserCaTrust.js` to emit a `network_security_config.xml` trusting user-installed
CAs. Without that, Android rejects the proxy certificate at step 1 above and pinning is never
exercised.

---

## 2. Build matrix

| Profile               | Pinning                 | User CAs | Application id                  | Env source                                  | Purpose                                                   |
| --------------------- | ----------------------- | -------- | ------------------------------- | ------------------------------------------- | --------------------------------------------------------- |
| `production`          | **ON** (canonical)      | off      | `com.ahmedmonib.eshop`          | `eas.json` → `production`                   | Play Store AAB.                                           |
| `internal`            | **ON** (canonical)      | off      | `com.ahmedmonib.eshop.internal` | `eas.json` → `internal`                     | Sideload APK. Identical to production but for the app id. |
| `development`         | ON, but skipped by host | off      | `com.ahmedmonib.eshop.dev`      | local `.env` (`env:tunnel` / `emu` / `lan`) | Dev client. Pinning auto-skips localhost/IP/tunnel hosts. |
| `internal-pinfail`    | **deliberately wrong**  | off      | `com.ahmedmonib.eshop.internal` | `eas.json` → `internal-pinfail`             | Proves the pin comparison executes. Must fail every call. |
| `internal-mitm`       | **ON** (canonical)      | **ON**   | `com.ahmedmonib.eshop.internal` | `eas.json` → `internal-mitm`                | Proves pinning blocks a trusted MITM certificate.         |
| `internal-mitm-nopin` | **OFF**                 | **ON**   | `com.ahmedmonib.eshop.internal` | `eas.json` → `internal-mitm-nopin`          | Control: proves the MITM harness actually intercepts.     |

`internal` and `production` resolve to byte-identical pinning configuration; a test asserts this.

| Profile               | EAS build                                | Local `.env` generation                     |
| --------------------- | ---------------------------------------- | ------------------------------------------- |
| `internal`            | `npm -w mobile run eas:build:internal`   | `npm -w mobile run env:internal`            |
| `production`          | `npm -w mobile run eas:build:production` | `npm -w mobile run env:production`          |
| `internal-pinfail`    | `npm -w mobile run eas:build:pinfail`    | `npm -w mobile run env:internal-pinfail`    |
| `internal-mitm`       | `npm -w mobile run eas:build:mitm`       | `npm -w mobile run env:internal-mitm`       |
| `internal-mitm-nopin` | `npm -w mobile run eas:build:mitm-nopin` | `npm -w mobile run env:internal-mitm-nopin` |

`env:internal-pinfail` / `env:internal-mitm*` run `scripts/use-eas-env.js`, which writes
`mobile/.env` **from the matching `eas.json` profile** rather than from a `.env.<profile>` file —
`mobile/.env.*` is git-ignored, so a committed per-profile file is impossible and a fresh clone
could not reproduce the build.

---

## 3. Proxy setup (Charles / mitmproxy)

### 3.1 Why a naive proxy test proves nothing

Apps targeting API 24+ trust only the **system** CA store. A proxy CA installed through Settings
goes to the **user** store. Android therefore rejects the intercepted chain during ordinary
validation and `CertificatePinner` never runs — every build fails identically, pinned or not.

Symptoms of hitting this rather than pinning:

- The proxy shows **no flows at all** for the API host — not a failed handshake, nothing.
- Logcat shows `SSLHandshakeException` / `Trust anchor for certification path not found`.
- **Unrelated components fail the same way** — `dev.expo.updates`, and Android's own
  `NetworkMonitor PROBE_HTTPS`. The OS connectivity probe has nothing to do with this app, so its
  failure is conclusive evidence that the CA is untrusted device-wide.
- No `SSLPeerUnverifiedException`, no `CertificatePinner`, no `SSL pinning validation failed` lines.

Only the `internal-mitm*` profiles remove this confound.

### 3.2 Install Charles (Windows)

1. Download from **<https://www.charlesproxy.com/download/>** and run the installer with defaults.
2. Launch Charles.

**Normal launch or Administrator?** Launch it **normally**. Administrator is only required when
Charles reconfigures the _Windows system_ proxy for you (Proxy → Windows Proxy) or installs its root
certificate into the Windows trust store. This runbook never uses either: the phone is pointed at
Charles manually, and the certificate is installed on the **phone**, not on Windows.

> Not recorded during the original verification: whether Charles was launched elevated. It should
> not matter for the manual-proxy flow described here. If Windows blocks the listening socket, an
> elevated launch plus the firewall step in 3.3 resolves it.

### 3.3 Windows Defender Firewall

The **most common reason the phone cannot reach the proxy at all.**

On first launch Windows shows _"Allow Charles Proxy to communicate on these networks"_. Tick
**Private networks** and allow. Public may be left unticked.

If you dismissed that prompt, add the rule manually:

1. Start → **Windows Defender Firewall with Advanced Security**.
2. **Inbound Rules** → **New Rule…** → **Program** → browse to the Charles executable (typically
   `C:\Program Files\Charles\Charles.exe`).
3. **Allow the connection** → tick **Private** only → name it `Charles Proxy`.

Confirm the network is classified Private: Settings → Network & Internet → Wi-Fi → your network →
Network profile type = **Private**. A network marked Public blocks inbound connections regardless of
the rule.

### 3.4 Find the correct Windows IP

```powershell
ipconfig
```

Use the **IPv4 Address** of the adapter that carries your Wi-Fi — usually
`Wireless LAN adapter Wi-Fi`. Example:

```
Wireless LAN adapter Wi-Fi:
   IPv4 Address. . . . . . . . . . . : 192.168.1.42     ← use this
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

Ignore: `127.0.0.1`, any `169.254.x.x` (no DHCP), WSL / Hyper-V / VirtualBox / VMware adapters
(often `172.x.x.x`), and Ethernet if you are on Wi-Fi. Picking a virtual adapter's address is a
frequent cause of "phone cannot reach the proxy".

**Verify phone and PC are on the same network** — the phone's IP must share the subnet and gateway:

```bash
adb shell ip route          # e.g. "192.168.1.0/24 ... src 192.168.1.57"
adb shell ping -c 3 192.168.1.42
```

Three replies means the phone can reach the PC. If ping fails, the two are on different networks
(guest Wi-Fi, 5 GHz vs 2.4 GHz on isolated SSIDs, or **AP/client isolation** enabled on the router).

### 3.5 Configure Charles

1. **Proxy → Proxy Settings**
   - HTTP Proxy **Port: `8888`** (Charles' default).
   - Tick **Enable transparent HTTP proxying**.
2. **Proxy → SSL Proxying Settings → SSL Proxying tab → Add**
   - Host: `api.vexflare.com` Port: `443`
   - Tick **Enable SSL Proxying**.
   - **Without this Charles tunnels TLS opaquely and decrypts nothing** — you would see a CONNECT
     entry with no readable body and wrongly conclude pinning blocked it.
   - A wildcard `*` also works and is easier while debugging.
3. **Proxy → Access Control Settings**
   - Add your LAN subnet, e.g. `192.168.1.0/24`. Otherwise Charles refuses the phone's connection.
4. **Proxy → Start Recording** (usually on by default — the red button must be active).

**Common mistakes:** forgetting the SSL Proxying host entry; leaving Access Control at localhost
only; a stale Wi-Fi proxy left configured on the phone from a previous session; recording stopped.

> **mitmproxy equivalent.** `mitmweb --listen-port 8080`. The web UI port is printed at startup —
> read it from mitmweb's own output rather than assuming, as it increments when a port is busy
> (`8081` by default; `8082` was observed during this migration). The CA lives at
> `~/.mitmproxy/mitmproxy-ca-cert.cer` and is also served from `mitm.it` while the proxy is active.

### 3.6 Configure the Android device

Settings → Network & internet → Wi-Fi → long-press your network → **Modify network** → Advanced →
**Proxy: Manual**

- **Proxy hostname:** your Windows IPv4 from 3.4 (e.g. `192.168.1.42`)
- **Proxy port:** `8888` (Charles) or `8080` (mitmproxy)
- Leave the bypass list empty.
- Save.

### 3.7 Install the proxy CA on the device

1. With the proxy active, open Chrome on the phone and visit **`chls.pro/ssl`** (Charles) or
   **`mitm.it`** (mitmproxy), then download the certificate.
2. Settings → Security & privacy → More security settings → **Encryption & credentials** → **Install
   a certificate** → **CA certificate** → **Install anyway** → select the downloaded file.
3. Android shows a "your network may be monitored" warning. That is expected.

**Verify it is installed:** Settings → Security & privacy → More security settings → Encryption &
credentials → **Trusted credentials** → **USER** tab. The Charles/mitmproxy entry must appear there.
If the USER tab is empty, the certificate did not install and the MITM procedures cannot work.

**Remove it afterwards** (do this when you finish testing): same screen → **User credentials** or
Trusted credentials → USER → tap the entry → **Remove** / **Disable**. Also set the Wi-Fi proxy back
to **None**.

### 3.8 Verify Charles is actually intercepting — before testing pinning

**Do this first, every time.** If this does not work, any pinning result is meaningless, because the
proxy is not intercepting anything.

1. Bring the Charles window to the front. The left pane is the flow list (Structure/Sequence tabs).
2. On the phone, open Chrome and load any HTTPS site, e.g. `https://example.com`.
3. Confirm **entries appear in Charles** as you browse. If nothing appears, the phone is not routing
   through the proxy — recheck 3.3, 3.4 and 3.6.
4. Now launch the app build under test and use it.
5. Confirm **`api.vexflare.com` appears** in the flow list.
6. Expand it and confirm individual requests are listed with methods and status codes — e.g.
   `GET /api/products  200`. Seeing only a greyed `CONNECT` entry means SSL Proxying is not
   configured for this host (step 3.5.2).
7. Select a request and check the **Request** and **Response** tabs. You must be able to read the
   JSON bodies. Readable bodies are the proof that TLS is being decrypted rather than tunnelled.

With mitmweb, the equivalent is the flow table at the URL mitmweb printed at startup; the same three
checks apply (flows appear, the API host appears, bodies are readable).

> **If step 5–7 fail on `internal-mitm-nopin`, stop.** That build has no pinning at all, so a
> failure there is a harness problem, not a pinning result.

---

## 4. Test procedures

### 4.0 Shared building blocks

**[LOGCAT]** — run in a second terminal before launching the app:

```bash
adb logcat -c
adb logcat | grep -iE "SSL pinning|CertificatePinner|SSLPeerUnverified|Trust anchor"
```

**[SIGNING]** — required for every local release build:

```bash
export ESHOP_ANDROID_KEYSTORE=~/Dev/secrets/eshop-upload-keystore.jks
export ESHOP_ANDROID_KEYSTORE_PASSWORD='<from Bitwarden>'
export ESHOP_ANDROID_KEY_ALIAS='<from Bitwarden>'
export ESHOP_ANDROID_KEY_PASSWORD='<from Bitwarden>'
```

**[BUILD]** — the local Gradle sequence, run from the repo root:

```bash
cd mobile
APP_VARIANT=internal npx expo prebuild --platform android --clean
npx expo start --clear & sleep 5 && kill %1     # wipe Metro cache; a stale bundle carries old config
cd android && ./gradlew assembleRelease
cd ../..
adb uninstall com.ahmedmonib.eshop.internal || true
adb install -r mobile/android/app/build/outputs/apk/release/app-release.apk
```

Expected tail of a successful build:

```
BUILD SUCCESSFUL in 9m 51s
792 actionable tasks: ...
```

> The Metro cache wipe is **load-bearing**, not hygiene. A stale bundle carries the previous
> profile's configuration, so the app can pass or fail for the wrong reason entirely.

---

### 4.1 Internal release verification

Proves the shipped configuration works against the real backend. **Proxy OFF.**

```bash
# 1. Select the internal environment.
npm -w mobile run env:internal
```

Expected console:

```
Switched mobile/.env -> .env.internal
```

```bash
# 2. Confirm .env carries NO pin hashes — app.config.js supplies them.
grep -c SSL_PINNING_HASHES mobile/.env        # → 0

# 3. Confirm the resolved config has the canonical pins.
cd mobile && APP_VARIANT=internal node -e "
  const c = require('./app.config.js').expo;
  console.log(c.extra.EXPO_PUBLIC_SSL_PINNING_HASHES);
" && cd ..
```

Expected: the three comma-separated hashes from `mobile/ssl-pins.json`.

```bash
# 4. Confirm they still match the live chain.
npm -w mobile run cert:pins
```

Then **[SIGNING]** and **[BUILD]**. Prebuild must print **no** `[withUserCaTrust]` warning.

Start **[LOGCAT]**, launch the app.

| Check         | Expected                                                                                                                                              |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| App behaviour | Login, browse, product detail, add to cart, checkout, logout all succeed.                                                                             |
| Session       | Survives force-quit and relaunch.                                                                                                                     |
| Logcat        | **Silent** — no pinning lines at all.                                                                                                                 |
| **PASS**      | Everything works and logcat is silent.                                                                                                                |
| **FAIL**      | Any `Certificate pinning failure!` → the shipped pins no longer match the live chain. Run `cert:pins`, compare with `ssl-pins.json`. **Do not ship.** |

EAS equivalent: `npm -w mobile run eas:build:internal`, then download and install the APK.

---

### 4.2 `internal-pinfail` — enforcement regression test

The cheap check: no proxy, no CA, one build. **Run this after every pin change.**

```bash
npm -w mobile run env:internal-pinfail
```

Expected console:

```
[use-eas-env] Wrote mobile/.env from the "internal-pinfail" eas.json profile (9 variables).
[use-eas-env] ⚠  SSL pinning hashes are intentionally invalid in this profile.
[use-eas-env]    Every API call in the resulting build must fail. That is the expected result.
[use-eas-env]    Restore the normal environment afterwards with: npm -w mobile run env:internal
```

Then **[SIGNING]** and **[BUILD]**, proxy **OFF**. Start **[LOGCAT]**, launch the app.

| Check         | Expected                                                                           |
| ------------- | ---------------------------------------------------------------------------------- |
| App behaviour | **Every** API call fails. Login impossible, no product data loads.                 |
| Logcat        | `SSL pinning validation failed for api.vexflare.com: Certificate pinning failure!` |
| **PASS**      | Total API failure **is** the pass condition.                                       |
| **FAIL**      | The app working normally means pinning is inert. **Stop and investigate.**         |

---

### 4.3 `internal-mitm-nopin` — MITM harness control

Proves the proxy actually intercepts. **Must pass before 4.4 means anything.**

```bash
npm -w mobile run env:internal-mitm-nopin
```

Expected console:

```
[use-eas-env] Wrote mobile/.env from the "internal-mitm-nopin" eas.json profile (10 variables).
[use-eas-env] ⚠  This profile TRUSTS USER-INSTALLED CA CERTIFICATES (ANDROID_TRUST_USER_CAS=1).
[use-eas-env]    The resulting build is deliberately interceptable. Never distribute it.
```

Run the prebuild step of **[BUILD]**; it must print:

```
[withUserCaTrust] ⚠  DIAGNOSTIC BUILD: user-installed CA certificates are TRUSTED.
```

Confirm the native output before compiling:

```bash
cat mobile/android/app/src/main/res/xml/network_security_config.xml
grep -o 'android:networkSecurityConfig="[^"]*"' mobile/android/app/src/main/AndroidManifest.xml
```

Expected:

```xml
<certificates src="system" />
<certificates src="user" />
```

```
android:networkSecurityConfig="@xml/network_security_config"
```

Finish **[BUILD]**. Set up the proxy (section 3), **proxy ON, CA installed**, and run the
interception checks in [3.8](#38-verify-charles-is-actually-intercepting--before-testing-pinning).

| Check         | Expected                                                                                                               |
| ------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Charles       | `api.vexflare.com` appears; individual requests show `GET … 200`; bodies readable.                                     |
| App behaviour | Works normally.                                                                                                        |
| **PASS**      | Decrypted API traffic is visible.                                                                                      |
| **FAIL**      | No flows → the harness is not intercepting. Fix section 3 before continuing. **4.4 is meaningless until this passes.** |

---

### 4.4 `internal-mitm` — the pinning proof

Same build, real pins. **Proxy stays ON, CA stays installed.**

```bash
npm -w mobile run env:internal-mitm
```

Then **[SIGNING]** and **[BUILD]**. Start **[LOGCAT]**, launch the app.

| Check         | Expected                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------- |
| App behaviour | Every API request fails.                                                                        |
| Charles       | **No** `api.vexflare.com` flows.                                                                |
| Logcat        | `SSL pinning validation failed for api.vexflare.com: Certificate pinning failure!` — repeated   |
| **PASS**      | The above. Android accepted the chain (it trusts the user CA) and **pinning** then rejected it. |
| **FAIL**      | Charles showing decrypted traffic means pinning is **not** enforced. **Stop. Do not ship.**     |

The pairing of 4.3 and 4.4 is the whole proof: identical builds, identical proxy, one variable
changed (pins present or absent), opposite outcomes.

---

### 4.5 Restore

**Do not skip.** The diagnostic `.env` files leave you building interceptable or broken APKs.

```bash
npm -w mobile run env:internal
```

Then **[BUILD]** again and confirm the app works normally (4.1). Also on the device:

- Wi-Fi proxy → **None**.
- Remove the proxy CA (section 3.7).

---

### 4.6 Production verification

See [section 5](#5-production-release-verification).

---

### 4.7 What production enablement did and did not touch

Audited when production pinning was switched on. Everything lives in the repository; **no external
system needed a change.**

| System                        | Change required | Why                                                                        |
| ----------------------------- | --------------- | -------------------------------------------------------------------------- |
| **Railway** (backend)         | **None**        | Pinning is entirely client-side. The server serves the same certificate.   |
| **Backend code**              | **None**        | No pinning references exist in `backend/src`.                              |
| **Expo secrets**              | **None**        | The pins are not secrets and are committed.                                |
| **EAS secrets**               | **None**        | `eas.json` references no secrets. Values come from `ssl-pins.json`.        |
| **EAS build profiles**        | Repo only       | `production` inherits the canonical pins; nothing to set in the dashboard. |
| **Android gradle.properties** | **None**        | Pinning is library-level. No network security config in shipping builds.   |
| **GitHub Actions**            | **None**        | No workflow references pinning or certificates.                            |
| **Play Console**              | **None**        | No new permissions and no manifest changes in shipping builds.             |
| **Apple Developer**           | **None**        | TrustKit needs no entitlement.                                             |

**Certificate rotation touches exactly one file: `mobile/ssl-pins.json`.**

---

## 5. Production release verification

Every release — internal or production — passes three separate phases. They are kept apart on
purpose: each answers a different question, and only the first is fully automatable.

| Phase               | Answers                                            | Where            | Time    |
| ------------------- | -------------------------------------------------- | ---------------- | ------- |
| **A. Build-time**   | Is the configuration correct before we build?      | Your machine     | < 5 min |
| **B. Play Console** | Did the right artefact reach the right track?      | Play Console/EAS | ~5 min  |
| **C. Runtime**      | Does the shipped binary actually work on a device? | Device           | ~15 min |

> **Scope note.** These phases verify the _inputs_ to the build (canonical pins, resolved config,
> version numbers) rather than unpacking the finished binary. Artefact-unpacking checks were removed
> because they depended on Expo's internal packaging layout, which is not stable across SDK
> versions. The compensating controls are `src/tests/config/sslPinResolution.test.js`, which asserts
> the resolution rules, and the fact that the build is deterministic from `app.config.js`.

---

### Phase A — Build-time verification (run before every release)

Copy-paste block. Every command is reproducible and exits non-zero on failure.

```bash
# 1. Clean tree — nothing uncommitted can leak into the build.
git status --porcelain && echo "^ must be empty"

# 2. Canonical pins still match the live certificate chain. Exits 1 if none match.
npm -w mobile run cert:pins -- --check

# 3. Config-level guarantees: single source, matrix, internal==production.
npm -w mobile test

# 4. Resolved config for the profile you are about to build.
cd mobile && APP_VARIANT=production node -e "
  const c = require('./app.config.js').expo;
  const canonical = require('./ssl-pins.json');
  const expected = canonical.pins.map((p) => p.hash).join(',');
  const pins = c.extra.EXPO_PUBLIC_SSL_PINNING_HASHES;
  console.log('applicationId :', c.android.package);
  console.log('versionName   :', c.version);
  console.log('versionCode   :', c.android.versionCode);
  console.log('pin expiry    :', c.extra.EXPO_PUBLIC_SSL_PINNING_EXPIRES);
  console.log('pins match    :', pins === expected ? 'YES' : 'NO  <-- STOP');
  console.log('expiry future :', new Date(c.extra.EXPO_PUBLIC_SSL_PINNING_EXPIRES) > new Date() ? 'YES' : 'NO  <-- STOP');
"; cd ..

# 5. No shippable profile trusts user CAs (must list ONLY the two -mitm profiles).
node -e "
  const b = require('./mobile/eas.json').build;
  console.log(Object.entries(b).filter(([, p]) => p.env && p.env.ANDROID_TRUST_USER_CAS).map(([n]) => n).join(', '));
"

# 6. Build.
npm -w mobile run eas:build:production      # or eas:build:internal
```

Expected values for step 4:

| Field           | `production`                         | `internal`                      |
| --------------- | ------------------------------------ | ------------------------------- |
| `applicationId` | `com.ahmedmonib.eshop`               | `com.ahmedmonib.eshop.internal` |
| `pins match`    | `YES`                                | `YES`                           |
| `expiry future` | `YES`                                | `YES`                           |
| `versionCode`   | greater than the last uploaded build | —                               |

**Stop conditions.** Any of these means do not build or upload:

- `cert:pins --check` exits 1 — no configured pin is in the live chain; the build would be unable to
  reach the API.
- `pins match: NO` — `app.config.js` is not resolving the canonical file.
- `expiry future: NO` — pin validation would be disabled in the field on install.
- Step 5 lists anything other than `internal-mitm-nopin, internal-mitm`.
- `versionCode` not incremented — Play rejects the upload.

#### No user-CA trust in the artefact

`ANDROID_TRUST_USER_CAS` is never set for shippable profiles (step 5), and `withUserCaTrust.js`
throws if it is combined with `APP_VARIANT=production`. To confirm on the generated project after a
local prebuild — these are Android-defined paths, stable across Expo versions:

```bash
ls mobile/android/app/src/main/res/xml/network_security_config.xml   # must NOT exist
grep -c networkSecurityConfig mobile/android/app/src/main/AndroidManifest.xml   # must be 0
```

And on a downloaded AAB, if you want artefact-level confirmation:

```bash
unzip -Z1 app-release.aab | grep -c 'res/xml/network_security_config'   # must be 0
```

---

### Phase B — Play Console verification

1. **EAS build page → Build details → Environment variables.** Confirm
   `EXPO_PUBLIC_SSL_PINNING_HASHES` is **absent**. That is correct — it is supplied by
   `app.config.js` from `ssl-pins.json`, not by `eas.json`. Its presence would mean someone
   reintroduced a second source.
2. Download the `.aab`, and keep `mapping.txt` from the Artifacts panel for crash deobfuscation.
3. Play Console → **Testing → Internal testing → Create new release**. Upload the `.aab`, add
   release notes.
4. Confirm the **version code** shown by Play matches step 4 of Phase A.
5. **Save → Review release → Start rollout to Internal testing.** Do **not** promote to Production.
6. Play Console → **App integrity** — confirm the upload was accepted and re-signed by Play App
   Signing without warnings.

---

### Phase C — Post-upload runtime verification

Install **from Google Play**, not from the locally downloaded AAB: Play App Signing re-signs the
upload, so the binary users receive is not the one you built.

1. Play Console → Internal testing → **Testers** tab → copy the opt-in URL.
2. Open it on the device with a tester account, accept, then **Download it on Google Play**.

Then, with the proxy **OFF**:

| #   | Check                   | Steps                                             | Expected                        |
| --- | ----------------------- | ------------------------------------------------- | ------------------------------- |
| 1   | Install from Play       | Opt-in URL → Play → Install                       | Installs; correct version shown |
| 2   | Login                   | Cold start → sign in                              | Succeeds                        |
| 3   | Browse products         | Home → category → product detail                  | Lists and images load           |
| 4   | Checkout                | Add to cart → COD order, then a Stripe card order | Both complete                   |
| 5   | Restart app             | Force-quit → relaunch                             | Still signed in (SecureStore)   |
| 6   | Background / foreground | Background 10+ min → foreground → act             | Session alive, no forced logout |
| 7   | OTA update              | Publish to the channel, relaunch twice            | Update downloads and applies    |
| 8   | Logcat                  | `adb logcat \| grep -i "SSL pinning"`             | **Silent**                      |

Optional proxy smoke test (proxy **ON**, CA installed — section 3):

| Check   | Expected                                        |
| ------- | ----------------------------------------------- |
| App     | All API calls fail                              |
| Charles | No `api.vexflare.com` flows                     |
| Logcat  | `Trust anchor for certification path not found` |

> This is blocked by **CA distrust**, not by pinning — a production build does not trust user CAs,
> so the chain is rejected before the pinner runs. It confirms the app is not interceptable in the
> field, but it is **not** a pinning proof. The pinning proof is
> [4.4](#44-internal-mitm--the-pinning-proof) on `internal-mitm`, which is why that is run on the
> internal build rather than here.

Afterwards: remove the CA and set the device proxy back to None.

---

### Promote, or roll back

Promote to Production only after all of Phase C passes.

| Situation                    | Action                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------- |
| Pins wrong, not yet promoted | Fix `ssl-pins.json`, rebuild, upload a new AAB. Nothing is in users' hands.                 |
| Pins wrong, already promoted | Play Console → Production → **Halt rollout** immediately, then ship a corrected build.      |
| Need pinning off entirely    | Set `MOBILE_SSL_PINNING_DISABLED=1` in the `production` profile env in `eas.json`, rebuild. |
| Whole migration bad          | `git revert` the SSL commits, then `npx expo prebuild --clean`, rebuild.                    |

⚠ **Pinning cannot be turned off over the air.** A binary that cannot reach the API also cannot
download an OTA update. Every rollback above needs a new store build. The `expirationDate` in
`ssl-pins.json` is the last-resort net for binaries already in the field: on that date pin
validation disables itself, so a stale build recovers without user action.

### Release verification checklist (< 5 minutes, before uploading)

Build-time only. Tick every box before the AAB leaves your machine.

```
[ ] git status --porcelain            → empty
[ ] npm -w mobile run cert:pins -- --check   → "OK: every configured pin is present" (exit 0)
[ ] npm -w mobile test                → all suites pass
[ ] app.config resolve (Phase A #4)   → pins match YES, expiry future YES
[ ] applicationId                     → com.ahmedmonib.eshop (production)
[ ] versionCode                       → incremented vs the last Play upload
[ ] versionName                       → intended release version
[ ] user-CA profiles (Phase A #5)     → only internal-mitm-nopin, internal-mitm
[ ] build succeeds                    → EAS build finishes green
```

### Post-upload verification checklist

Runtime, on a device, installed **from Play Internal Testing**.

```
[ ] installed from Play Internal Testing (not the local AAB)
[ ] login succeeds
[ ] browse products — lists and images load
[ ] checkout — one COD order and one Stripe card order complete
[ ] restart app — still signed in
[ ] background 10+ min → foreground — session alive, no forced logout
[ ] OTA update downloads and applies
[ ] logcat silent — no "SSL pinning" lines
[ ] (optional) proxy smoke test — all API calls fail, "Trust anchor" in logcat
```

Any unticked box → do not promote to Production. See [Promote, or roll back](#promote-or-roll-back).

---

## 6. Certificate rotation

`api.vexflare.com` is served by **Railway**, which auto-renews its Let's Encrypt certificate roughly
every 60 days with a **new private key each time**. Because we pin the CA layer and not the leaf, a
routine renewal changes nothing and **no action is required**. Rotation is needed only when the
public key behind a pinned certificate changes — a CA re-keying an intermediate under the same name
counts, as does dropping a cross-signed path or moving to a different CA — or ahead of the pin set's
own `expirationDate`. Only `mobile/ssl-pins.json` is edited; no environment variable changes.

### Procedure

```bash
# 0. Is rotation needed at all? Exits non-zero when no configured pin still
#    appears in the live chain. A routine leaf renewal will not trip it.
npm -w mobile run cert:pins -- --check

# 1. Regenerate the canonical file from the live chain.
npm -w mobile run cert:pins -- --write

# 2. Review the diff. Removed pins are the dangerous part — any build in the field
#    relying solely on them is locked out until users install a new binary.
git diff mobile/ssl-pins.json

# 3. Config-level checks.
npm -w mobile test

# 4. Re-run the verification suite:
#      4.2  internal-pinfail   (enforcement)
#      4.3  internal-mitm-nopin (harness control)
#      4.4  internal-mitm      (pinning proof)
#      4.5  restore
#      4.1  internal           (works against the real backend)

# 5. Ship and verify production (section 5).
```

`--write` prints a warning listing any removed pins and reminds you to re-verify.

### Where hashes are updated

**`mobile/ssl-pins.json`, and nowhere else.** `eas.json`, `.env*`, `app.config.js` and the app code
all derive from it. A test fails if a copy reappears elsewhere.

### Keeping the expiry current

Also review `expirationDate` in `ssl-pins.json` at each rotation. Too near and pinning silently
switches off in the field; too far and a stale build stays locked out longer. Roughly 12 months out,
moved forward each release, is the intent.

---

## 7. Maintenance notes

### The MITM profiles must never ship

`internal-mitm` and `internal-mitm-nopin` trust **user-installed CA certificates**, which makes them
interceptable by anyone who can add a CA to the device. `internal-mitm-nopin` additionally has no
pinning at all.

Guards in place:

- `plugins/withUserCaTrust.js` **throws** if `ANDROID_TRUST_USER_CAS=1` is combined with
  `APP_VARIANT=production`.
- Without the flag the plugin is completely inert — no XML written, no manifest attribute.
- A test asserts no non-`-mitm` profile sets `ANDROID_TRUST_USER_CAS`.
- Generated `.env` files carry a "NEVER distribute it" banner, and the generator prints one.

Neither profile is wired to `eas submit`, and both use the internal application id, so neither can
reach the Play production track by accident.

### Internal and production must stay behaviourally identical

The internal APK is the vehicle for validating pinning before a store release. That is only sound if
it differs from production **solely** in application id and update channel. Both resolve pins from
the same canonical file, and `src/tests/config/sslPinResolution.test.js` asserts identical pins and
expiry with differing package names. Do not add pinning-relevant env to one profile and not the
other.

### When to re-run the verification suite

| Trigger                                                                | Run                                          |
| ---------------------------------------------------------------------- | -------------------------------------------- |
| `ssl-pins.json` changed                                                | Full suite: 4.2, 4.3, 4.4, 4.5, 4.1, then 5. |
| `react-native-ssl-public-key-pinning` upgraded                         | Full suite.                                  |
| Expo SDK / React Native upgrade                                        | Full suite — networking internals move.      |
| New Architecture enabled                                               | Full suite.                                  |
| `sslPinningAdapter.native.js` or the AuthProvider mount effect changed | Full suite.                                  |
| Routine feature work                                                   | `npm -w mobile test` only.                   |

---

## 8. Verification command reference

Every check in one place. Run from the repo root unless noted.

**Resolved Expo config, per profile**

```bash
cd mobile
APP_VARIANT=internal node -e "
  const c = require('./app.config.js').expo;
  console.log('package :', c.android.package);
  console.log('pins    :', c.extra.EXPO_PUBLIC_SSL_PINNING_HASHES || '(none)');
  console.log('expires :', c.extra.EXPO_PUBLIC_SSL_PINNING_EXPIRES);
"
# Diagnostic profiles: prefix with the same env eas.json sets, e.g.
#   ANDROID_TRUST_USER_CAS=1 MOBILE_SSL_PINNING_DISABLED=1 APP_VARIANT=internal node -e ...
```

**Canonical pins**

```bash
node -e "console.log(require('./mobile/ssl-pins.json').pins.map(p => p.hash).join(','))"
npm -w mobile run cert:pins                 # live chain, print only
npm -w mobile run cert:pins -- --write      # live chain, update ssl-pins.json
```

**Confirm no duplicate source exists**

```bash
grep -rn "$(node -e "console.log(require('./mobile/ssl-pins.json').pins[0].hash)")" \
  --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=android --exclude-dir=ios .
# Expect only mobile/ssl-pins.json (and test fixtures).
```

**Generated native config** (after `expo prebuild`)

```bash
cat mobile/android/app/src/main/res/xml/network_security_config.xml   # only for internal-mitm*
grep -o 'android:networkSecurityConfig="[^"]*"' mobile/android/app/src/main/AndroidManifest.xml
grep -n "applicationId" mobile/android/app/build.gradle
grep -n "newArchEnabled" mobile/android/gradle.properties
```

**Inspect a built artefact**

Only Android-defined paths are checked here. Expo's own packaging layout (e.g. where the resolved
config is stored) is an internal detail that has changed between SDK versions, so it is **not** used
for verification — resolve the config from `app.config.js` instead (above), which is reproducible.

```bash
APK=mobile/android/app/build/outputs/apk/release/app-release.apk

# User-CA trust config must be absent from any shippable build.
unzip -Z1 "$APK" | grep -c 'res/xml/network_security_config'          # must be 0

# Same check on a downloaded bundle.
unzip -Z1 app-release.aab | grep -c 'res/xml/network_security_config' # must be 0

# Is the pinning native module compiled in?
for d in $(unzip -Z1 "$APK" | grep '^classes'); do
  unzip -p "$APK" "$d" | strings | grep -m1 -i sslpublickeypinning && break
done
```

**Logcat filters**

```bash
adb logcat -c && adb logcat | grep -iE "SSL pinning|CertificatePinner|SSLPeerUnverified|Trust anchor"
adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
adb logcat | grep -i "dev.expo.updates"       # OTA + a useful CA-trust canary
```

**Device / network**

```bash
adb devices -l
adb shell ip route
adb shell ping -c 3 <windows-ip>
adb shell pm list packages | grep ahmedmonib
adb shell pm clear com.ahmedmonib.eshop.internal
adb uninstall com.ahmedmonib.eshop.internal
```

**Tests**

```bash
npm -w mobile test
npx jest src/tests/config/sslPinResolution.test.js --forceExit          # from mobile/
npx jest src/tests/api-client/sslPinningAdapter.test.js --forceExit     # from mobile/
```

---

## 9. Troubleshooting

Real issues encountered during this migration, plus their signatures.

### `Trust anchor for certification path not found` instead of a pinning error

**Not a pinning result.** Android rejected the chain before `CertificatePinner` ran. Expected for
any non-`-mitm` profile under a proxy, because apps targeting API 24+ do not trust user-installed
CAs.

Tell-tale: `dev.expo.updates` and Android's own `NetworkMonitor PROBE_HTTPS` fail the same way. The
OS connectivity probe has nothing to do with this app, so its failure proves the CA is untrusted
device-wide.

To test pinning, use `internal-mitm` / `internal-mitm-nopin` (4.3, 4.4).

### `chromium … net_error -202 (ERR_CERT_AUTHORITY_INVALID)`

WebView stack, not the API client. Same root cause as above. Ignore it when assessing pinning.

### Charles shows no flows at all

Work through, in order:

1. Recording is started (Proxy → Start Recording).
2. Phone Wi-Fi proxy is set to Manual with the right host and port (3.6).
3. The IP is the **Wi-Fi adapter's** IPv4, not WSL/Hyper-V/VirtualBox (3.4).
4. `adb shell ping -c 3 <windows-ip>` replies.
5. Windows Defender Firewall allows Charles on **Private** networks, and the network is classified
   Private (3.3).
6. Charles Access Control includes your subnet (3.5.3).

### Charles shows a `CONNECT` entry but no readable requests

SSL Proxying is not configured for the host. Proxy → SSL Proxying Settings → add
`api.vexflare.com:443` (3.5.2). Without it Charles tunnels TLS without decrypting, which looks
deceptively like pinning blocking the connection.

### Proxy CA not installed / not trusted

Trusted credentials → **USER** tab must list the entry (3.7). If the phone downloaded the file but
never prompted to install, open Settings → Encryption & credentials → Install a certificate → CA
certificate and pick it from Downloads manually.

### App works when it should fail (e.g. during `internal-pinfail`)

Almost always a **stale Metro bundle** — the APK carries the previous profile's configuration.
Rebuild with the cache wipe:

```bash
cd mobile && npx expo start --clear & sleep 5 && kill %1
APP_VARIANT=internal npx expo prebuild --platform android --clean
cd android && ./gradlew assembleRelease
```

Then confirm which configuration the build was made from — resolve it the same way the build did:

```bash
cd mobile && APP_VARIANT=internal node -e "
  console.log(require('./app.config.js').expo.extra.EXPO_PUBLIC_SSL_PINNING_HASHES || '(none)');
"; cd ..
```

Run this with the same `.env` in place as the build used (`head -12 mobile/.env` shows which profile
generated it).

Second possibility: you installed an older APK. `adb uninstall` first — all `internal*` profiles
share one application id.

### API flows visible in the proxy when they should not be

On `internal-mitm` this means pinning is **not** enforced. Check the packaged pins with the command
above; if they are empty you built `internal-mitm-nopin` by mistake. If they are correct and traffic
is still visible, stop and investigate before shipping anything.

### App fails after restoring `env:internal`

The restore needs a **rebuild**, not just an env switch — the configuration is baked in at build
time. Re-run **[BUILD]** after `npm -w mobile run env:internal`.

If it still fails, check for a leftover diagnostic `.env`:

```bash
head -12 mobile/.env      # a GENERATED FILE banner means a diagnostic profile is still active
```

And confirm the pins still match the live chain (`npm -w mobile run cert:pins`) — a genuine issuer
rotation produces the same symptom.

### Pinning silently absent in a release build but fine in dev

A value read via raw `process.env` instead of `readStringEnv`. Release builds only reliably expose
`app.config.js` `extra`; the dev client masks the omission because Metro loads `.env` dynamically.

### `INSTALL_FAILED_UPDATE_INCOMPATIBLE`

A build with the same application id but a different signing key is installed.
`adb uninstall com.ahmedmonib.eshop.internal` first.

### `Missing release keystore configuration`

The `ESHOP_ANDROID_*` variables are not exported in the shell running Gradle (4.0 **[SIGNING]**).
Deliberate — it prevents producing an unsigned release.

### iOS pinning appears to do nothing in a dev build

`expo-build-properties` sets `ios.networkInspector: false` for exactly this reason; the dev network
inspector proxies URLSession and defeats TrustKit.

---

## 10. Final acceptance checklist

Everything that had to be true for this migration to be considered complete. This records what was
actually validated.

### Implementation

- [x] `react-native-ssl-pinning` removed; `react-native-ssl-public-key-pinning` installed.
- [x] `sslPinningAdapter.native.js` keeps its filename, `createPinningAdapter` export, options
      shape, `null` cases and fail-closed throw.
- [x] The downgrade-and-replay path removed — pinning cannot be disabled at runtime.
- [x] The fail-closed throw is no longer swallowed by `AuthProvider`.
- [x] Pins read via `readStringEnv`, so release builds resolve them from `extra`.
- [x] Obsolete plumbing deleted: `mobile/certs/`, `withSslPinningCerts`, `withSSLPinningFix`,
      `sync-ssl-certs.js`, `fetch-ssl-cert.js`.

### Single source of truth

- [x] `mobile/ssl-pins.json` is the only place the hashes appear (test-enforced).
- [x] No `.env*` file and no `eas.json` profile carries the hashes.
- [x] `internal` and `production` resolve byte-identical pinning config (test-enforced).
- [x] Local Gradle and EAS builds resolve identically — verified by building the internal APK
      locally and by `src/tests/config/sslPinResolution.test.js`.

### Automated checks

- [x] `npm run lint` — 0 errors.
- [x] `npm -w mobile test` — 33 suites, 204 passed, 1 todo, 0 failed.
- [x] `npm -w mobile run cert:pins` reproduces the chain in `ssl-pins.json`.
- [x] `npx expo prebuild --clean` succeeds; no diagnostic warning for shippable profiles.
- [x] `./gradlew :app:assembleRelease` succeeds **without** the old `verifyReleaseResources`
      workaround.

### Build artefact verification

- [x] Local internal APK: `applicationId com.ahmedmonib.eshop.internal`, **no**
      `res/xml/network_security_config`, pinning native module present.
- [x] `internal-pinfail` APK carries the wrong pins and none of the real ones.

### On-device functional verification (internal build)

- [x] Login / browse / background / foreground / logout / relaunch
- [x] Token refresh
- [x] Offline → reconnect recovery
- [x] Airplane-mode recovery
- [x] COD order
- [x] Stripe order
- [x] Google OAuth
- [x] Facebook OAuth
- [x] OTA update
- [x] Development client unaffected

### On-device pinning verification

- [x] `internal-mitm-nopin`: user-CA trust enabled, `network_security_config` generated, manifest
      attribute present, **Charles decrypted `api.vexflare.com`**, app worked — harness confirmed.
- [x] `internal-mitm`: all API requests failed, **no** `api.vexflare.com` traffic in Charles, logcat
      showed `SSL pinning validation failed for api.vexflare.com: Certificate pinning failure!` —
      proving the pinner ran _after_ Android accepted the chain.

### Production

- [x] Production profile pinned from the canonical source.
- [x] Enablement audit: no Railway, backend, Expo/EAS secret, Gradle, CI or store-console change
      required (4.7).
- [ ] **Outstanding —** production AAB built, uploaded to Internal testing, installed from Play, and
      verified per [section 5](#5-production-release-verification) (Phases A, B and C) before
      promotion to the Production track.

### Documentation

- [x] This runbook.
- [x] `docs/deployment.md`, `mobile/env.md`, `mobile/docs/build-and-install.md`,
      `mobile/docs/internalTesting-and-productionAABinstall.md`, `CLAUDE.md`, `README.md` updated
      and cross-linked.
- [x] `ArchitecturePage.jsx` claims corrected.

### Deferred, tracked elsewhere

- [ ] New Architecture enablement — no remaining blockers, but a separate migration
      (`mobile/docs/docs/mobile/android-build-config.md`).
- [ ] `mobile/scripts/release-pipeline.sh` calls `:app:assembleInternalRelease` /
      `:app:bundleProdRelease`, which do not exist (no Gradle flavors). Pre-existing, unrelated to
      pinning.
