# Custom Dev Client Setup & Daily Workflow

## Table of contents

- [Custom Dev Client Setup \& Daily Workflow](#custom-dev-client-setup--daily-workflow)
  - [Table of contents](#table-of-contents)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Install the Android toolchain](#install-the-android-toolchain)
    - [Accept licenses \& set environment variables](#accept-licenses--set-environment-variables)
    - [Create `local.properties`](#create-localproperties)
  - [Connect a device](#connect-a-device)
    - [USB via `usbipd-win`](#usb-via-usbipd-win)
    - [Wi-Fi debugging](#wi-fi-debugging)
  - [Manage mobile environment profiles](#manage-mobile-environment-profiles)
  - [Pre-flight project checks](#pre-flight-project-checks)
  - [Adding native dependencies (managed workflow)](#adding-native-dependencies-managed-workflow)
  - [App variants (`APP_VARIANT`)](#app-variants-app_variant)
  - [Build or refresh the custom dev client](#build-or-refresh-the-custom-dev-client)
    - [Full reinstall (after a `node_modules` wipe or fresh clone)](#full-reinstall-after-a-node_modules-wipe-or-fresh-clone)
  - [Deep-link scheme](#deep-link-scheme)
  - [Daily development loop](#daily-development-loop)
  - [Frequently used commands](#frequently-used-commands)
  - [Add or reset a device](#add-or-reset-a-device)
  - [Clean builds \& cache resets](#clean-builds--cache-resets)
  - [Examples of “native-affecting” changes in app.config.js](#examples-of-native-affecting-changes-in-appconfigjs)
  - [Troubleshooting](#troubleshooting)
  - [Troubleshooting](#troubleshooting-1)
    - [Android Device Missing in WSL](#android-device-missing-in-wsl)
      - [Symptoms](#symptoms)
    - [Fast Diagnosis](#fast-diagnosis)
    - [Step 1 — Verify WSL Can See the Phone](#step-1--verify-wsl-can-see-the-phone)
    - [Step 2 — Reattach Device from Windows](#step-2--reattach-device-from-windows)
    - [Step 3 — Verify USB Attachment Inside WSL](#step-3--verify-usb-attachment-inside-wsl)
    - [Step 4 — Restart ADB](#step-4--restart-adb)
    - [Step 5 — Reset USB Debugging Authorization](#step-5--reset-usb-debugging-authorization)
    - [Known Recovery Sequence](#known-recovery-sequence)
    - [Expo Tunnel Failures](#expo-tunnel-failures)
      - [Symptoms](#symptoms-1)
    - [Notes](#notes)
    - [Verify Metro Is Healthy](#verify-metro-is-healthy)
    - [Verify Backend Tunnel](#verify-backend-tunnel)
    - [Distinguishing Tunnel vs ADB Problems](#distinguishing-tunnel-vs-adb-problems)
  - [Automation script](#automation-script)
    - [Extra: sideload an internal release APK](#extra-sideload-an-internal-release-apk)
  - [Fix a stuck dev-client install](#fix-a-stuck-dev-client-install)

---

## Overview

Use this guide with the [environment reference](./env.md) to manage the **Expo custom development
client** inside the monorepo.

- All commands run from **WSL (Ubuntu 22.04)** unless noted.
- The monorepo uses **npm workspaces**; scope with `npm -w <workspace> run <script>`.
- The mobile app registers the custom scheme `eshop://` (e.g. `eshop://reset-password?...`).
- Web on **Vercel**, backend on **Railway**. Emails include the deep link + a SPA fallback.

---

## Prerequisites

1. Node 20.x and npm.
2. Git, Java 17 (installed below), optional Watchman.
3. Android Studio on Windows (for AVD) or a physical device.
4. Optional: `npm i -g expo-cli` (we mostly use `npx`).
5. Clone & install:

```bash
git clone <repo>
cd ecommerce-mern-website
npm install
```

---

## Install the Android toolchain

```bash
sudo apt-get update
sudo apt-get install -y curl unzip usbutils openjdk-17-jdk cmake ninja-build

SDK="$HOME/Android"
mkdir -p "$SDK/cmdline-tools"
cd "$SDK/cmdline-tools"
curl -L -o cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip -q cmdline-tools.zip
mv cmdline-tools latest
```

### Accept licenses & set environment variables

```bash
export ANDROID_SDK_ROOT="$HOME/Android"
export ANDROID_HOME="$ANDROID_SDK_ROOT"
export PATH="$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/emulator:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"

yes | sdkmanager --licenses
yes | sdkmanager \
  "platform-tools" \
  "platforms;android-35" \
  "build-tools;35.0.0" \
  "ndk;27.1.12297006"
```

Persist in your shell profile:

```bash
sed -i '/ANDROID_SDK_ROOT/d;/ANDROID_HOME/d;/cmdline-tools/d;/platform-tools/d;/JAVA_HOME/d' ~/.zshrc
cat >> ~/.zshrc <<'ZRC'
export ANDROID_SDK_ROOT="$HOME/Android"
export ANDROID_HOME="$ANDROID_SDK_ROOT"
export PATH="$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/emulator:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
ZRC
source ~/.zshrc
```

### Create `local.properties`

```bash
cd <repo-root>/mobile/android
tee local.properties <<'EOF_LOCAL'
sdk.dir=/home/$(whoami)/Android
cmake.dir=/usr
EOF_LOCAL
```

---

## Connect a device

### USB via `usbipd-win`

**Windows (PowerShell Admin):**

```powershell
usbipd list

# first time only
usbipd bind --busid <BUSID>

# every reconnect / after reboot
usbipd attach --wsl --busid <BUSID> --auto-attach
```

**Back in WSL:**

```bash
lsusb | grep -Ei '04e8|6860' || lsusb
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="04e8", MODE="0666", GROUP="plugdev"' | sudo tee /etc/udev/rules.d/51-android.rules >/dev/null
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo usermod -aG plugdev "$USER"; newgrp plugdev

adb kill-server && adb start-server
adb devices -l   # accept RSA prompt on device
```

> Replace `04e8` if your vendor differs.

### Wi-Fi debugging

```bash
adb pair <phone-ip>:<pair-port>
adb connect <phone-ip>:<debug-port>
adb devices -l
```

---

## Manage mobile environment profiles

```bash
npm -w mobile run env:emu     # Android emulator (10.0.2.2)
npm -w mobile run env:lan     # Physical device on LAN
npm -w mobile run env:tunnel  # ngrok HTTPS (default for sharing)
```

After switching, restart Metro with `--clear`.

---

## Pre-flight project checks

1. **Entry registers the app** (`mobile/index.js`):

```js
import "expo-dev-client";
import { registerRootComponent } from "expo";
import App from "./App";
registerRootComponent(App);
```

1. **No axios deep imports** (use public API only).
2. **Android wrappers**: `MainApplication.kt` uses `ReactNativeHostWrapper`; `MainActivity.kt`
   returns a `ReactActivityDelegateWrapper`.
3. **Manifest** `<application android:name=".MainApplication" .../>`.
4. **Gradle uses Expo plugin**: root `mobile/android/build.gradle` has the Expo classpath;
   `app/build.gradle` applies `expo`.

---

## Adding native dependencies (managed workflow)

`mobile/android/` is **git-ignored** and generated by `expo prebuild` (locally) or by EAS Build at
build time — there is **no committed native project to restore**, and there are **no Gradle
flavors**. When you add or upgrade a native dependency, regenerate the native project locally:

```bash
cd mobile
APP_VARIANT=development npx expo prebuild --platform android   # regenerate android/ for the dev variant
```

Because `android/` is regenerated from `app.config.js` (plus the config plugins), there is **no
recovery procedure** after prebuild — everything custom (deep-link scheme, SSL-pinning cert copy,
SDK versions) is reapplied automatically by `app.config.js`,
`mobile/plugins/withSslPinningCerts.js`, and `expo-build-properties`. There is nothing to restore
from git.

---

## App variants (`APP_VARIANT`)

There are no Gradle flavors. `app.config.js` reads `process.env.APP_VARIANT` and sets the
`applicationId`, deep-link scheme, and launcher label per build:

| `APP_VARIANT` | applicationId                   | Scheme              | Use                     |
| ------------- | ------------------------------- | ------------------- | ----------------------- |
| `production`  | `com.ahmedmonib.eshop`          | `vexflare`          | Play Store (default)    |
| `internal`    | `com.ahmedmonib.eshop.internal` | `vexflare-internal` | EAS internal / sideload |
| `development` | `com.ahmedmonib.eshop.dev`      | `vexflare-dev`      | local dev / dev-client  |

- **Locally**, pick the variant with the env var: `APP_VARIANT=development npx expo run:android`.
- **On EAS**, each `eas.json` profile sets `APP_VARIANT` in its `env` (production / internal /
  development), so the cloud build bakes in the matching id + scheme.

The three application IDs differ, so all three apps install side-by-side on one device.

---

## Build or refresh the custom dev client

The dev client is the `development` variant (`com.ahmedmonib.eshop.dev`, scheme `vexflare-dev`).
Build and install it with Expo — no Gradle commands:

```bash
# Pick the JS env (tunnel is the daily default), then build + install the dev client in one step.
npm -w mobile run env:tunnel
cd mobile
APP_VARIANT=development npx expo run:android
```

`expo run:android` regenerates `android/` if needed, compiles the debug dev client, and installs it.
Once it is installed, the daily loop is just Metro (see "Daily development loop"):

```bash
APP_VARIANT=development npx expo start --dev-client
```

### Full reinstall (after a `node_modules` wipe or fresh clone)

```bash
# repo root
rm -rf node_modules && npm install

cd mobile
adb kill-server && adb start-server
adb devices -l                                   # must show ONE 'device'
adb uninstall com.ahmedmonib.eshop.dev || true   # remove a conflicting prior dev client
APP_VARIANT=development npx expo run:android
```

> If install fails with `INSTALL_FAILED_UPDATE_INCOMPATIBLE`, uninstall the same package first — a
> build signed with a different key is already on the device (see Troubleshooting).

---

## Deep-link scheme

The deep-link scheme is set by `app.config.js` from `APP_VARIANT` (`vexflare` / `vexflare-internal`
/ `vexflare-dev`) — there is no committed `AndroidManifest.xml` to hand-edit. To change a scheme,
edit the `VARIANTS` map in `mobile/app.config.js` and rebuild (`expo run:android` locally, or
`eas build`); prebuild regenerates the manifest from the config.

## Daily development loop

```bash
# 1) Start backend (+ optional web)
npm run dev
# backend → https://localhost:5001
# frontend → https://localhost:5173

# 2) Expose API via ngrok and sync the env
#  The backend listens on HTTPS locally. Ngrok must forward to
#  https://localhost:5001, otherwise requests will fail with
#  ERR_NGROK_3004 and the mobile app will receive HTTP 503 errors.

npx ngrok http https://localhost:5001
npm -w mobile run env:tunnel   # writes .env for the tunnel URL

# 3) Start Metro (tunnel + dev client)
cd mobile
npx expo start --tunnel --dev-client --clear

# 4) Open the dev client (scan QR, choose “Development Build”)

# 5) Smoke test deep link
npx uri-scheme open "eshop://reset-password?token=TEST123&email=user%40example.com" --android
```

---

## Frequently used commands

```bash
npm run dev                                # backend + frontend
npm -w mobile run start:tunnel             # Expo tunnel with cache bust
adb devices -l                             # list devices
adb reverse tcp:8081 tcp:8081              # Metro to emulator
adb reverse tcp:19000 tcp:19000            # dev menu to emulator
APP_VARIANT=development npx expo run:android   # build + install the dev client (from mobile/)
adb pair <ip>:<pair-port> && adb connect <ip>:<debug-port>
git commit -m "msg" --no-verify            # bypass husky if needed
```

---

## Add or reset a device

1. Enable **Developer options** + **USB debugging**.
2. Install via USB or Wi-Fi:
   - **USB**: follow [USB workflow](#usb-via-usbipd-win), then
     `APP_VARIANT=development npx expo run:android`.
   - **Wi-Fi**: `adb pair`, `adb connect`, then `adb install -r ...`.

3. Start Metro using the [daily loop](#daily-development-loop).
4. If `unauthorized`, unlock device and accept RSA; re-run `adb devices -l`.

Remove a stale build:

```bash
adb uninstall com.ahmedmonib.eshop.internal || true
adb shell pm clear com.ahmedmonib.eshop.internal || true
```

---

## Clean builds & cache resets

```bash
cd mobile/android
./gradlew --stop || true
rm -rf ~/.gradle/caches .cxx
./gradlew clean
```

Reset Metro cache:

```bash
cd mobile
npx expo start --tunnel -c   # or --localhost -c
```

Verify `local.properties` if Gradle references Windows paths:

```
cmake.dir=/usr
sdk.dir=/home/<you>/Android
ndk.dir=/home/<you>/Android/ndk/27.1.12297006
```

## Examples of “native-affecting” changes in app.config.js

These do require:

```bash
cd mobile
APP_VARIANT=development npx expo run:android   # regenerates android/ and rebuilds the dev client
```

- android.softwareKeyboardLayoutMode (e.g. 'pan' vs 'resize')

- android.intentFilters, scheme (deeplinks)

- android.permissions (adding/removing)

- android.edgeToEdgeEnabled, icon, adaptiveIcon images

- android.package (applicationId), versionCode, minSdkVersion/targetSdkVersion via plugins

- Any plugin/config that rewrites AndroidManifest.xml, Gradle, or resources

- These do NOT require prebuild:

- Only extra: { EXPO*PUBLIC*\* } values used by JS

- Theming, translations, UI logic, axios calls, navigation configuration (unless you also changed
  deeplink intent filters)

---

## Troubleshooting

- **`adb devices` empty** → Reattach with `usbipd attach --wsl`, check `lsusb`, restart ADB, accept
  prompts.
- **Expo opens Expo Go** → Use `--dev-client` and choose **Development Build**.
- **CMake/NDK errors** → Wipe caches (`rm -rf ~/.gradle/caches .cxx`), check `local.properties`,
  `./gradlew clean`.
- **`expo run:android` hangs ~99% / install fails** → Usually a **signature/package conflict**. Fix:

  ```bash
  adb uninstall com.ahmedmonib.eshop.dev
  APP_VARIANT=development npx expo run:android
  ```

- **`INSTALL_FAILED_UPDATE_INCOMPATIBLE`** when reinstalling the same variant signed by a different
  key (e.g. a local dev build over an EAS-signed one):
  - Uninstall the matching package first:

    ```bash
    adb uninstall com.ahmedmonib.eshop.dev       # dev client
    adb uninstall com.ahmedmonib.eshop.internal  # EAS internal APK
    ```

- **Metro can’t reach API** → Open the API URL on the device, confirm `.env` profile, restart Metro
  with `-c`.
- **Deep link no-op** → Trigger with `uri-scheme` and watch Metro logs (`🔗` events). Verify email
  links still include `deepLink`.
- **ngrok restarts** → Keep the terminal open; run `npm -w mobile run env:tunnel` after each
  restart.

- **ngrok returns ERR_NGROK_3004**

  Symptoms:

  ```text
  ERR_NGROK_3004
  The server returned an invalid or incomplete HTTP response
  ```

  Mobile app may also show:

  ```text
  Login failed
  Status code 503
  ```

  Verify the backend protocol:

  ```bash
  curl -vk https://localhost:5001
  ```

  If the backend is serving HTTPS, start ngrok with:

  ```bash
  ngrok http https://localhost:5001
  ```

  Do NOT use:

  ```bash
  ngrok http http://localhost:5001
  ```

  when the backend is configured for HTTPS.

  when the backend is configured for HTTPS.

---

## Troubleshooting

### Android Device Missing in WSL

#### Symptoms

```bash
adb devices -l
```

Returns:

```text
List of devices attached
```

with no devices listed.

Common symptoms:

- `npx expo start --dev-client` launches Metro but cannot open the app.
- Pressing `a` in Expo takes a long time or does nothing.
- Android Studio does not detect the phone from WSL.
- `adb reverse` commands fail.
- Expo development build never opens on the device.

---

### Fast Diagnosis

Run:

```bash
lsusb && adb devices -l
```

Interpretation:

| Result                                  | Meaning                 |
| --------------------------------------- | ----------------------- |
| Samsung visible + device visible in adb | Everything is working   |
| Samsung visible + adb empty             | ADB issue               |
| Samsung not visible                     | USB/IP attachment issue |

---

### Step 1 — Verify WSL Can See the Phone

```bash
lsusb
```

Expected:

```text
04e8:6860 Samsung Electronics
```

If Samsung is not listed:

```text
WSL lost the USB attachment.
```

Continue to Step 2.

---

### Step 2 — Reattach Device from Windows

Open PowerShell as Administrator:

```powershell
wsl --shutdown

usbipd list

usbipd attach --wsl --busid 1-4
```

Replace:

```text
1-4
```

with the current Samsung BUSID shown by:

```powershell
usbipd list
```

Expected output:

```text
usbipd: info: Using WSL distribution 'Ubuntu-22.04'
usbipd: info: Device attached
```

---

### Step 3 — Verify USB Attachment Inside WSL

```bash
lsusb
```

Expected:

```text
Bus XXX Device XXX: ID 04e8:6860 Samsung Electronics
```

If Samsung appears, continue.

---

### Step 4 — Restart ADB

```bash
adb kill-server
adb start-server
adb devices -l
```

Expected:

```text
R58T60ETLJZ device
```

or another connected device serial.

---

### Step 5 — Reset USB Debugging Authorization

If the device still does not appear:

On the phone:

```text
Developer Options
  → Revoke USB debugging authorizations
```

Then:

```text
Disable USB debugging
Enable USB debugging
```

Unplug and reconnect the cable.

Accept the RSA authorization prompt.

---

### Known Recovery Sequence

This sequence resolved the issue on 2026-06-07:

1. Phone disappeared from WSL.
2. `adb devices -l` returned no devices.
3. `lsusb` did not show Samsung.
4. Ran:

```powershell
wsl --shutdown
usbipd attach --wsl --busid 1-4
```

1. Verified:

```bash
lsusb
```

showed:

```text
Samsung Electronics
```

1. Restarted ADB:

```bash
adb kill-server
adb start-server
adb devices -l
```

1. Device appeared again:

```text
R58T60ETLJZ device
```

Root cause: WSL lost the USB/IP attachment even though Windows still detected the phone.

### Expo Tunnel Failures

#### Symptoms

```text
CommandError: failed to start tunnel

remote gone away
```

---

### Notes

This issue is usually unrelated to ADB.

Possible causes:

- Expo tunnel service issue
- ngrok issue
- Expo/ngrok integration issue
- Temporary network issue

---

### Verify Metro Is Healthy

Run:

```bash
npx expo start --dev-client --clear
```

If Metro starts and displays:

```text
Metro waiting on exp+...
```

then Metro is healthy.

---

### Verify Backend Tunnel

Manual backend tunnel:

```bash
ngrok http https://localhost:5001
```

Verify:

```bash
curl https://your-ngrok-url.ngrok-free.app
```

returns a valid response.

---

### Distinguishing Tunnel vs ADB Problems

| Symptom                              | Likely Cause            |
| ------------------------------------ | ----------------------- |
| Tunnel fails with "remote gone away" | Expo/ngrok issue        |
| `adb devices -l` empty               | ADB / USB issue         |
| Samsung missing from `lsusb`         | USB/IP issue            |
| Metro starts successfully            | Expo bundler is healthy |

Always check:

```bash
lsusb
adb devices -l
```

before spending time debugging Expo.

## Automation script

Optional one-shot bootstrap (`scripts/setup-wsl-android.sh`):

```bash
#!/usr/bin/env bash
set -euo pipefail

sudo apt-get update
sudo apt-get install -y curl unzip usbutils openjdk-17-jdk cmake ninja-build

SDK="$HOME/Android"
mkdir -p "$SDK/cmdline-tools"
cd "$SDK/cmdline-tools"
curl -L -o cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip -q -o cmdline-tools.zip
mv -f cmdline-tools latest || true

export ANDROID_SDK_ROOT="$SDK"
export ANDROID_HOME="$SDK"
export PATH="$SDK/platform-tools:$SDK/emulator:$SDK/cmdline-tools/latest/bin:$PATH"
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"

yes | sdkmanager --licenses
yes | sdkmanager "platform-tools" "platforms;android-35" "build-tools;35.0.0" "ndk;27.1.12297006"

sed -i '/ANDROID_SDK_ROOT/d;/ANDROID_HOME/d;/cmdline-tools/d;/platform-tools/d;/JAVA_HOME/d' ~/.zshrc
cat >> ~/.zshrc <<'ZRC'
export ANDROID_SDK_ROOT="$HOME/Android"
export ANDROID_HOME="$ANDROID_SDK_ROOT"
export PATH="$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/emulator:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"
ZRC

cat <<'RULE' | sudo tee /etc/udev/rules.d/51-android.rules >/dev/null
SUBSYSTEM=="usb", ATTR{idVendor}=="04e8", MODE="0666", GROUP="plugdev"
RULE

sudo udevadm control --reload-rules && sudo udevadm trigger
sudo usermod -aG plugdev "$USER" || true

echo "✅ Android SDK installed. Restart your shell or run: source ~/.zshrc"
```

Run:

```bash
chmod +x scripts/setup-wsl-android.sh
bash scripts/setup-wsl-android.sh
```

Then reconnect your device (`usbipd attach --wsl` or `adb connect ...`) and follow the
[daily loop](#daily-development-loop).

---

### Extra: sideload an internal release APK

For a side-loadable release APK for quick QA, build the `internal` profile on EAS and install the
downloaded APK (full details in `mobile/docs/build-and-install.md`):

```bash
eas build --platform android --profile internal   # com.ahmedmonib.eshop.internal, scheme vexflare-internal
# then download the APK from the EAS dashboard and:
adb uninstall com.ahmedmonib.eshop.internal || true
adb install -r ~/Downloads/<downloaded>.apk
```

## Fix a stuck dev-client install

If `expo run:android` hangs at ~99% or fails to install, it's almost always a signature/package
conflict — a prior build of the same id signed with a different key is on the device:

```bash
# 1) One device only
adb devices -l

# 2) Remove any existing dev-client package
adb uninstall com.ahmedmonib.eshop.dev || true
# If you see "not installed for 0", force-uninstall for the current user:
adb shell pm uninstall --user 0 com.ahmedmonib.eshop.dev || true

# 3) (Optional) clear temp APKs to avoid partial installs
adb shell rm -f /data/local/tmp/*.apk || true

# 4) Rebuild + reinstall the dev client (from mobile/)
cd mobile
APP_VARIANT=development npx expo run:android
```

---
