Totally fine to keep your **existing commands** — they already do the right things. The only
“gotchas” that were tripping you up were:

1. **Order matters** (pull → env → build).
2. **Path matters** when installing the APK from repo root.
3. Make sure **pinning is OFF for internal builds** (otherwise you’ll get “Network error” if the
   cert name/env don’t match).

Below is a clean, repeatable flow using your scripts, plus two tiny improvements.

---

# ✅ What your commands already do

- `npm -w mobile run cert:pull -- --host ... --name eshop_api` → saves `mobile/certs/eshop_api.cer`
  (DER leaf) from your server.

- `npm -w mobile run env:production` → runs `scripts/use-env.sh production` which:
  - copies `mobile/.env.production` → `mobile/.env`
  - runs `scripts/sync-ssl-certs.js` which **copies all `.cer` files to**
    `mobile/android/app/src/main/assets/` (this is the correct place for `react-native-ssl-pinning`
    on Android).
  - also validates that `EXPO_PUBLIC_SSL_PINNING_CERTS` names exist in `mobile/certs`.

- Gradle build (APK or AAB) then packages the cert into:
  - APK: `assets/eshop_api.cer`
  - AAB: `base/assets/eshop_api.cer`

You can see that in your logs already, so the copying step works 👍

---

# 🧭 Canonical flows (memorize these)

## A) Internal testing APK (pinning OFF)

```bash
# 1) (Optional but safe) refresh cert
npm -w mobile run cert:pull -- --host api.ahmedmonib-eshop-demo.com --name eshop_api

# 2) Use an internal env with pinning OFF (see “Add this script” below)
npm -w mobile run env:internal

# 3) Build APK
cd mobile/android && ./gradlew clean :app:assembleInternalRelease && cd -

# 4) Install (from repo root, use absolute path to avoid “cannot stat”)
APK_ABS="$(realpath mobile/android/app/build/outputs/apk/internal/release/app-internal-release.apk)"
adb devices -l
adb push "$APK_ABS" /data/local/tmp/eshop-internal.apk
adb shell pm install -r /data/local/tmp/eshop-internal.apk
```

> If install hangs: try `adb kill-server && adb start-server`, switch to Wi-Fi ADB, or unplug/replug
> USB.

## B) Production AAB (pinning ON)

```bash
# 1) Refresh cert
npm -w mobile run cert:pull -- --host api.ahmedmonib-eshop-demo.com --name eshop_api

# 2) Production env (pinning ON) — this copies the cert into assets
npm -w mobile run env:production

# 3) Build AAB
cd mobile/android && ./gradlew clean :app:bundleProdRelease && cd -

# 4) Upload these to Play Console
ls -lah mobile/android/app/build/outputs/bundle/prodRelease/app-prod-release.aab
ls -lah mobile/android/app/build/outputs/mapping/prodRelease/mapping.txt
```

---

# 🧩 Add this small improvement (one-time)

You already have `env:emu`, `env:lan`, `env:tunnel`, `env:production`. Add an **internal** profile
with pinning OFF so your test APK never fails on pinning.

### 1) `mobile/.env.internal` (new)

```dotenv
EXPO_PUBLIC_API_BASE_URL=https://api.ahmedmonib-eshop-demo.com
# Turn OFF pinning for internal testing (leave blank)
EXPO_PUBLIC_SSL_PINNING_CERTS=
```

### 2) `mobile/package.json` (add script)

```diff
  "scripts": {
+   "env:internal": "bash ./scripts/use-env.sh internal",
    "env:emu": "bash ./scripts/use-env.sh emu",
    "env:lan": "bash ./scripts/use-env.sh lan",
    "env:tunnel": "bash ./scripts/use-env.sh tunnel",
    "env:production": "bash ./scripts/use-env.sh production",
```

> You don’t need to change `sync-ssl-certs.js` — it already copies `.cer` files to **assets**, which
> is correct for Android.

---

# 🧪 Quick verifications

- **APK/AAB contains the cert**:

  ```bash
  unzip -l mobile/android/app/build/outputs/apk/internal/release/app-internal-release.apk | grep -i 'assets/eshop_api\.cer'
  unzip -l mobile/android/app/build/outputs/bundle/prodRelease/app-prod-release.aab | grep -i 'assets/eshop_api\.cer'
  ```

- **Runtime logs** (when you tap “Login”), look for pinning failures:

  ```bash
  adb logcat -c
  adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
  ```

- **If internal APK says “Network error”**:
  - Ensure you built with `npm -w mobile run env:internal` (pinning OFF).
  - Clear app data and retry:

    ```bash
    adb shell pm clear com.ahmedmonib.eshop
    ```

---

# 🔎 Why you saw “cannot stat …apk”

You ran `adb push` from the repo root while using a **relative path** that only exists if you’re
inside `mobile/android`. Use the absolute path trick shown above (`APK_ABS="$(realpath …)"`), and it
won’t matter where you run the command from.

---

# TL;DR

- **Keep using your commands** — they’re good.
- The correct order is **cert:pull → env:<profile> (which syncs certs) → build**.
- Use a dedicated **`env:internal`** (pinning OFF) for the APK you test on-device.
- Use **`env:production`** (pinning ON, `EXPO_PUBLIC_SSL_PINNING_CERTS=eshop_api`) for your Play
  AAB.

If you still hit “Network error” with the **internal** build, we’ll inspect logs next and check the
fetch/axios layer to ensure your pinning toggle is only applied when `EXPO_PUBLIC_SSL_PINNING_CERTS`
is set.
