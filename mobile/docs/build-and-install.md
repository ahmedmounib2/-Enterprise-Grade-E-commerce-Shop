---

# Android builds: Internal APK sideload & Production AAB

Use this as your “muscle-memory” reference for building and installing **internal release APKs** (pinning **OFF**) and **Play Console AABs** (pinning **ON**). It assumes you’re working from the monorepo root.

> TL;DR: **cert:pull → env:<profile> → build**.
> Internal APKs must have **pinning OFF** and you **must uninstall** any debug build that shares the same `applicationId` before installing a release (signature mismatch).

---

## Prereqs

- Device connected (`adb devices -l` shows it as `device`).
- Java 17 / Android SDK set (see `mobile/docs/dev-setup.md`).
- Your env scripts: `env:internal`, `env:production` (details below).

---

## One-time: add **internal** env (pinning OFF)

Only do this if you don’t already have it.

**`mobile/.env.internal`**

```dotenv
EXPO_PUBLIC_API_BASE_URL=https://api.ahmedmonib-eshop-demo.com
# Turn OFF pinning for internal testing
EXPO_PUBLIC_SSL_PINNING_CERTS=
```

**`mobile/package.json`**

```diff
  "scripts": {
+   "env:internal": "bash ./scripts/use-env.sh internal",
    "env:production": "bash ./scripts/use-env.sh production",
    "env:emu": "bash ./scripts/use-env.sh emu",
    "env:lan": "bash ./scripts/use-env.sh lan",
    "env:tunnel": "bash ./scripts/use-env.sh tunnel"
  }
```

> The `use-env.sh` script also runs `sync-ssl-certs.js`, which copies any `mobile/certs/*.cer` into
> `android/app/src/main/assets/` so the **pinned** builds can find them.

---

## Flow A — Internal testing **APK** (pinning **OFF**)

> Build flavor: **internalRelease** Package name: usually `com.ahmedmonib.eshop.internal`

```bash
# 1) Optional: refresh the server cert (safe even if pinning is off)
npm -w mobile run cert:pull -- --host api.ahmedmonib-eshop-demo.com --name eshop_api

# 2) Pinning OFF
npm -w mobile run env:internal

# 3) Build internal release APK
cd mobile/android
./gradlew clean :app:assembleInternalRelease
cd -

# 4) Before installing, remove any existing build with the *same* applicationId
#    (prevents INSTALL_FAILED_UPDATE_INCOMPATIBLE signature errors)
adb uninstall com.ahmedmonib.eshop.internal || true

# 5) Install (use absolute path so it works from repo root)
APK_ABS="$(realpath mobile/android/app/build/outputs/apk/internal/release/app-internal-release.apk)"
adb devices -l
adb push "$APK_ABS" /data/local/tmp/eshop-internal.apk
adb shell pm install -r /data/local/tmp/eshop-internal.apk
```

**Verify APK contains your cert (it may or may not be used when pinning is off):**

```bash
unzip -l "$APK_ABS" | grep -i 'assets/eshop_api\.cer' || echo "No pinned cert in APK (ok for internal)"
```

**If install hangs/fails:**

```bash
adb kill-server && adb start-server
# or switch to Wi-Fi ADB
# adb pair <ip>:<pair-port> && adb connect <ip>:<debug-port>
```

---

## Flow B — Production **AAB** (pinning **ON**)

> Build flavor: **prodRelease** Package name: usually `com.ahmedmonib.eshop`

```bash
# 1) Refresh cert (ensures latest leaf; saved as mobile/certs/eshop_api.cer)
npm -w mobile run cert:pull -- --host api.ahmedmonib-eshop-demo.com --name eshop_api

# 2) Pinning ON (env sets EXPO_PUBLIC_SSL_PINNING_CERTS=eshop_api and syncs cert to assets/)
npm -w mobile run env:production

# 3) Build AAB
cd mobile/android
./gradlew clean :app:bundleProdRelease
cd -

# 4) Upload to Play Console
ls -lah mobile/android/app/build/outputs/bundle/prodRelease/app-prod-release.aab
ls -lah mobile/android/app/build/outputs/mapping/prodRelease/mapping.txt
```

**Sanity check: AAB has the pinned cert in base assets**

```bash
unzip -l mobile/android/app/build/outputs/bundle/prodRelease/app-prod-release.aab | grep -i 'assets/eshop_api\.cer'
```

> Remember to bump `versionCode`/`versionName` for Play uploads (in
> `mobile/android/app/build.gradle` or `gradle.properties` depending on your setup).

---

## Common pitfalls & quick fixes

### 1) `INSTALL_FAILED_UPDATE_INCOMPATIBLE: ... signatures do not match`

You tried to install a **release** over a **debug** (or vice-versa) that shares the same
`applicationId`.

Fix:

```bash
# Choose the package that conflicts and uninstall it, then install again
adb shell pm list packages | grep eshop
adb uninstall com.ahmedmonib.eshop.internal || true   # internal flavor
adb uninstall com.ahmedmonib.eshop          || true   # prod flavor
```

> Tip: Use distinct `applicationId`/`applicationIdSuffix` per flavor so debug/dev client and release
> can coexist.

### 2) “Network error” on internal APK

Internal builds should have pinning **OFF**:

- You **must** build after `npm -w mobile run env:internal`.
- Clear app data and retry:

  ```bash
  adb shell pm clear com.ahmedmonib.eshop.internal
  ```

- Check runtime logs while you hit Login:

  ```bash
  adb logcat -c
  adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
  ```

### 3) Using relative paths with `adb push`

Always compute an **absolute** APK path:

```bash
APK_ABS="$(realpath mobile/android/app/build/outputs/apk/internal/release/app-internal-release.apk)"
adb push "$APK_ABS" /data/local/tmp/eshop-internal.apk
```

### 4) “Where did my cert go?”

- The env script copies `mobile/certs/*.cer` → `android/app/src/main/assets/`
- Verify after build:

  ```bash
  unzip -l "$APK_ABS" | grep -i 'assets/eshop_api\.cer' || true
  unzip -l mobile/android/app/build/outputs/bundle/prodRelease/app-prod-release.aab | grep -i 'assets/eshop_api\.cer'
  ```

---

## Clean build checklist (when switching branches or native deps changed)

```bash
npm ci

# quick sanity that native tooling is present
test -d node_modules/@react-native/gradle-plugin && echo "RN gradle plugin OK"
test -d node_modules/expo-modules-autolinking/android && echo "Expo autolinking OK"
test -d node_modules/expo/packages/expo-gradle-plugin && echo "Expo gradle plugin OK"

cd mobile/android
./gradlew --stop || true
rm -rf ~/.gradle/kotlin .cxx app/build app/.cxx
./gradlew clean

adb kill-server && adb start-server
adb devices -l
```

---

## Useful one-liners

**See packages installed on device:**

```bash
adb shell pm list packages | grep eshop
adb shell dumpsys package com.ahmedmonib.eshop.internal | sed -n '1,60p'
```

**Find Gradle variants / verify tasks:**

```bash
cd mobile/android
./gradlew :app:tasks --all | grep -i -E 'assemble|bundle|install'
```

**Confirm flavor dimensions (if unsure):**

```bash
rg "productFlavors|applicationId" mobile/android/app/build.gradle -n
```

---

## Mental model

- **Internal APK** = quick sideload, **pinning OFF**, app id like `com.ahmedmonib.eshop.internal`.
  Never install over a debug with the same id (different signatures).

- **Play AAB** = **pinning ON**, cert packaged into assets, uploaded to Console. Always bump
  version, keep `mapping.txt` for deobfuscation.
