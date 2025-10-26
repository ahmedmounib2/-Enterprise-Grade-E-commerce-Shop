# Android builds: Internal APK sideload & Production AAB

Use this as your “muscle-memory” reference for building and installing **internal release APKs** (pinning **OFF**) and **Play Console AABs** (pinning **ON**). It assumes you’re working from the monorepo root.

> **TL;DR:** `cert:pull → env:<profile> → build`.
> Internal APKs must have **pinning OFF** and you **must uninstall** any debug build that shares the same `applicationId` before installing a release (signature mismatch).

---

## Prereqs

* Device connected (`adb devices -l` shows it as `device`).
* Java 17 / Android SDK set (see `mobile/docs/dev-setup.md`).
* Env scripts available in `mobile/package.json`: `env:internal`, `env:production` (details below).

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

> `use-env.sh` also runs `sync-ssl-certs.js`, which copies any `mobile/certs/*.cer` into
> `android/app/src/main/assets/` so **pinned** builds can find them.

---

## Scripts overview (maintenance)

* `scripts/fetch-ssl-cert.js` – `npm -w mobile run cert:pull -- --host <host> --name <alias>`
  Downloads server leaf cert to `mobile/certs/<alias>.cer`.

* `scripts/sync-ssl-certs.js` – invoked by `use-env.sh`
  Copies `mobile/certs/*.cer` → `mobile/android/app/src/main/assets/`.

* `scripts/use-env.sh <profile>` – switches `mobile/.env` to `.env.<profile>` and syncs certs.

* Root `scripts/patch-safe-area-yoga.mjs` – **runs on `npm install`** (root `postinstall`) and patches Safe Area **C++ Yoga** file.
  ⚠️ Only needs to run after dependency reinstall or when `react-native-safe-area-context` is updated.
  You **do not** need to run it between Internal → Prod builds.

---

## Flow A — Internal testing **APK** (pinning **OFF**)

> Build variant: `internalRelease`
> Package name: typically `com.ahmedmonib.eshop.internal`

```bash
# 1) Optional: refresh the server cert (safe even if pinning is off)
npm -w mobile run cert:pull -- --host api.ahmedmonib-eshop-demo.com --name eshop_api

# 2) Pinning OFF
npm -w mobile run env:internal

# 3) Build internal release APK
cd mobile/android
./gradlew clean :app:assembleInternalRelease
cd -

# 4) Before installing, remove any existing build with the same applicationId
adb uninstall com.ahmedmonib.eshop.internal || true

# 5) Install (absolute path so it works from repo root)
APK_ABS="$(realpath mobile/android/app/build/outputs/apk/internal/release/app-internal-release.apk)"
adb devices -l
adb push "$APK_ABS" /data/local/tmp/eshop-internal.apk
adb shell pm install -r /data/local/tmp/eshop-internal.apk
```

**Verify APK contains your cert (not required when pinning is off):**

```bash
unzip -l "$APK_ABS" | grep -i 'assets/eshop_api\.cer' || echo "No pinned cert in APK (ok for internal)"
```

---

## Flow B — Production **AAB** (pinning **ON**)

> Build variant: `prodRelease`
> Package name: typically `com.ahmedmonib.eshop`

```bash
# 1) Refresh cert (ensures latest leaf; saved to mobile/certs/eshop_api.cer)
npm -w mobile run cert:pull -- --host api.ahmedmonib-eshop-demo.com --name eshop_api

# 2) Pinning ON (env sets EXPO_PUBLIC_SSL_PINNING_CERTS and syncs cert to assets/)
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

> Remember to bump `versionCode`/`versionName` before uploading.

---

## Switching variants (Internal → Prod) **during the same session**

You **do not** need to re-run `npm ci` or the patch script when you go from Internal to Prod.

**Minimal steps:**

```bash
# You already built/installed Internal. Now switch to Prod and build AAB:
npm -w mobile run env:production
cd mobile/android
./gradlew :app:bundleProdRelease
cd -
```

**Only if you see codegen/Kotlin errors (Safe Area “Unresolved reference …Spec/Interface”) run:**

```bash
cd mobile/android
./gradlew --stop
rm -rf app/.cxx .cxx app/build ../node_modules/react-native-safe-area-context/android/build
./gradlew clean
# Try to trigger codegen explicitly (task name can vary; use the one that exists on your project)
./gradlew :app:generateCodegenSchemaFromJavaScript --info || true
./gradlew :app:generateProdReleaseCodegenArtifactsFromSchema --info || true
./gradlew :app:bundleProdRelease --info
cd -
```

---

## Common pitfalls & quick fixes

### 1) `INSTALL_FAILED_UPDATE_INCOMPATIBLE: ... signatures do not match`

You tried to install a **release** over a **debug** (or vice-versa) that shares the same `applicationId`.

```bash
adb shell pm list packages | grep eshop
adb uninstall com.ahmedmonib.eshop.internal || true   # internal flavor
adb uninstall com.ahmedmonib.eshop          || true   # prod flavor
```

> Tip: Use distinct `applicationId`/`applicationIdSuffix` per flavor so debug/dev client and release can coexist.

### 2) “Network error” on internal APK

Internal builds should have pinning **OFF**:

* Ensure you built after `npm -w mobile run env:internal`.
* Clear app data and retry:

  ```bash
  adb shell pm clear com.ahmedmonib.eshop.internal
  ```

* Watch logs while reproducing:

  ```bash
  adb logcat -c
  adb logcat -s ReactNativeJS,OkHttp,SSL,Conscrypt,ConnectivityService,AndroidRuntime
  ```

### 3) Safe Area **codegen/Kotlin** errors when switching variants

Symptoms like:

```
Unresolved reference 'NativeSafeAreaContextSpec'
Unresolved reference 'RNCSafeAreaViewManagerInterface'
```

Fix (quick reusable sequence):

```bash
cd mobile/android && \
./gradlew --stop && \
rm -rf app/.cxx .cxx app/build ../node_modules/react-native-safe-area-context/android/build && \
./gradlew clean && \
./gradlew :app:generateCodegenSchemaFromJavaScript --info || true && \
./gradlew :app:generateProdReleaseCodegenArtifactsFromSchema --info || true && \
./gradlew :app:bundleProdRelease --info
```

If the `…ProdReleaseCodegen…` task name differs, use the exact name from:

```bash
./gradlew :app:tasks --all | grep -i codegen
```

---

## Clean build checklist (when switching branches or native deps changed)

```bash
npm ci   # re-installs deps and re-applies the Safe Area Yoga patch (root postinstall)

cd mobile/android
./gradlew --stop || true
rm -rf ~/.gradle/kotlin .cxx app/build app/.cxx
./gradlew clean

adb kill-server && adb start-server
adb devices -l
```

> Re-run `npm ci` **only** if you deleted `node_modules`, changed RN/native libs, or updated `react-native-safe-area-context`. Routine Internal → Prod builds do **not** require it.

---

## Useful one-liners

**See packages on device:**

```bash
adb shell pm list packages | grep eshop
adb shell dumpsys package com.ahmedmonib.eshop.internal | sed -n '1,60p'
```

**Find Gradle variants / verify tasks:**

```bash
cd mobile/android
./gradlew :app:tasks --all | grep -i -E 'assemble|bundle|install|codegen'
```

**Confirm flavor dimensions (if unsure):**

```bash
rg "productFlavors|applicationId" mobile/android/app/build.gradle -n
```

---

## Mental model

* **Internal APK** = quick sideload, **pinning OFF**, app id like `com.ahmedmonib.eshop.internal`.
  Never install over a debug with the same id (different signatures).

* **Play AAB** = **pinning ON**, cert packaged into assets, uploaded to Console.
  Always bump version, keep `mapping.txt` for deobfuscation.

---

### FAQ

**Q: Do I need to run `npm ci` between Internal and Prod?**
A: **No.** Only if you removed `node_modules` or changed native deps. The Safe Area patch runs on install automatically.

**Q: Do I need to re-run the patch script between builds?**
A: **No.** It’s applied during install. Re-run only after dependency reinstall or if the Safe Area lib version changes.

**Q: My Internal APK built, but Prod AAB fails with Safe Area Kotlin errors. What changed?**
A: Codegen for the **Prod** variant didn’t run after a clean. Trigger the codegen tasks (see the “codegen/Kotlin errors” section) and re-build.

---
