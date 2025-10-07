# Custom Dev Client Setup & Daily Workflow

````md
## Table of contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Install the Android toolchain](#install-the-android-toolchain)
  - [Accept licenses & set environment variables](#accept-licenses--set-environment-variables)
  - [Create `local.properties`](#create-localproperties)
- [Connect a device](#connect-a-device)
  - [USB via `usbipd-win`](#usb-via-usbipd-win)
  - [Wi-Fi debugging](#wi-fi-debugging)
- [Manage mobile environment profiles](#manage-mobile-environment-profiles)
- [Pre-flight project checks](#pre-flight-project-checks)
- [Build or refresh the custom dev client](#build-or-refresh-the-custom-dev-client)
- [Daily development loop](#daily-development-loop)
- [Frequently used commands](#frequently-used-commands)
- [Add or reset a device](#add-or-reset-a-device)
- [Clean builds & cache resets](#clean-builds--cache-resets)
- [Troubleshooting](#troubleshooting)
- [Automation script](#automation-script)

---

## Overview

Use this guide alongside the [environment reference](./env.md) to manage the Expo custom development
client inside the monorepo. Key facts:

- All commands run from **WSL (Ubuntu 22.04)** unless otherwise noted.
- The monorepo uses **npm workspaces**; scope commands with `npm -w <workspace> run <script>`.
- The mobile app registers the custom scheme `eshop://` and links `reset-password` to the
  `ResetPassword` screen for both authenticated and anonymous users.
- Production web deploys live on **Vercel**, backend on **Railway**. Emails include both a deep link
  (mobile) and a web fallback (SPA).

---

## Prerequisites

1. Install Node 20.x and npm (already provided in dev container / WSL image).
2. Install Git, Watchman (optional on Linux), and Java 17 (installed during toolchain step).
3. Ensure Windows has **Android Studio** (for AVD manager) or physical device available.
4. Optional but recommended: [Expo CLI](https://docs.expo.dev/workflow/expo-cli/) globally
   (`npm install -g expo-cli`)—not required because we use `npx`.
5. Clone the repo and install dependencies:

   ```bash
   git clone <repo>
   cd ecommerce-mern-website
   npm install
   ```
````

---

## Install the Android toolchain

Run these commands once inside WSL to install SDK tools, CMake/NDK, and Gradle prerequisites.

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

Persist variables in your shell profile (zsh shown):

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

Gradle must know where the Linux SDK/NDK lives.

```bash
cd <repo-root>/mobile/android  # adjust path if different
tee local.properties <<'EOF_LOCAL'
sdk.dir=/home/$(whoami)/Android
cmake.dir=/usr
EOF_LOCAL
```

If you store the repo elsewhere, update the path accordingly. This prevents Gradle from pointing at
`/mnt/c/...` and resolves CMake JSON errors.

---

## Connect a device

Choose **one** connection workflow. USB is recommended for the first install.

### USB via `usbipd-win`

**On Windows (PowerShell as Administrator):**

```powershell
usbipd list
# Note the BusId for your phone (e.g., 1-6)
usbipd bind   --busid <BUSID>
usbipd attach --wsl --busid <BUSID> --auto-attach
```

**Back in WSL:**

```bash
# See that a new USB device exists
lsusb | grep -Ei '04e8|6860' || lsusb

echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="04e8", MODE="0666", GROUP="plugdev"' | sudo tee /etc/udev/rules.d/51-android.rules >/dev/null
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo usermod -aG plugdev "$USER"; newgrp plugdev

adb kill-server && adb start-server
adb devices -l  # accept the RSA prompt on the phone
```

> Replace the vendor ID if you are not using a Samsung device (`04e8`).

### Wi-Fi debugging

After the first USB pairing you can go wireless:

1. On the phone, enable **Developer options → Wireless debugging**.

2. In WSL, pair and connect:

   ```bash
   adb pair <phone-ip>:<pair-port>
   adb connect <phone-ip>:<debug-port>
   adb devices -l
   ```

3. Keep the device awake while pairing; accept prompts on-screen.

---

## Manage mobile environment profiles

Environment presets live in `mobile/.env.*`. Switch between them using workspace scripts (run from
repo root):

```bash
npm -w mobile run env:emu     # Android emulator profile (10.0.2.2)
npm -w mobile run env:lan     # Physical device on same Wi-Fi
npm -w mobile run env:tunnel  # HTTPS via ngrok (default for sharing)
```

Review what each profile contains in [Mobile `.env` profiles](./env.md#mobile-env-profiles). After
changing profiles, restart Metro with the `-c` flag to clear cache.

---

## Pre-flight project checks

Before building or launching the dev client, confirm these baseline assumptions so Expo native
modules register correctly.

1. **Entry file registers the app** – `mobile/index.js` must import `expo-dev-client` and call
   `registerRootComponent(App)`. The current file already does this:

   ```js
   import "expo-dev-client";
   import { registerRootComponent } from "expo";
   import App from "./App";

   registerRootComponent(App);
   ```

2. **Axios deep imports removed** – replace any `axios/lib/...` imports with the public API. For the
   SSL pinning adapter we now rely on:

   ```js
   import axios, { AxiosError } from "axios";
   const requestUrl = axios.getUri(config);
   ```

   Run `rg "axios/lib" -g"*.js"` to ensure no other deep imports remain.

3. **Android wrappers are present** – verify `MainApplication.kt` uses `ReactNativeHostWrapper` and
   `MainActivity.kt` returns a `ReactActivityDelegateWrapper`. These are required for Expo modules
   to auto-register.

4. **Manifest points at the wrapper** – inside `android/app/src/main/AndroidManifest.xml` the
   `<application>` tag must include `android:name=".MainApplication"`.

5. **Gradle wires in the Expo plugin** – the root `mobile/android/build.gradle` must add
   `classpath('expo:expo-gradle-plugin')` inside `buildscript.dependencies`, and
   `android/app/build.gradle` must include `apply plugin: 'expo'` with the other Android/Kotlin
   plugins. If either line is missing you will hit the “Plugin with id 'expo' not found” failure
   when configuring `:app` and Expo native modules will never load.

Fix any deviation before proceeding; otherwise the dev client APK will install without the Expo
native modules and JS will crash on boot with “Cannot find native module …” errors.

---

## Build or refresh the custom dev client

Run when you change native modules, `app.config.js`, or the Android project. Pure JS/TS edits do not
require a rebuild.

1. **Remove every old build from the device/emulator** – delete both the dev and prod IDs (plus Expo
   Go if it is still present) so you cannot accidentally open an outdated binary:

   ```bash
   adb uninstall com.ahmedmonib.eshop.dev || true
   adb uninstall com.ahmedmonib.eshop      || true
   adb uninstall host.exp.exponent         || true
   ```

2. **Clean Gradle caches** – only necessary when native dependencies change, but it is the safest
   way to recover from “missing native module” crashes:

   ```bash
   cd mobile/android
   ./gradlew --stop || true
   rm -rf ~/.gradle/kotlin .cxx app/build app/.cxx
   ./gradlew clean
   ```

3. **Install the debug dev client** – this variant bundles Expo Dev Launcher and every Expo module
   required by the JS bundle. Keep it installed for day-to-day development. The installed Android
   app is labelled **“Eshop Debug”**, so it can live alongside the Play Store production build on
   the same device without package conflicts:

   ```bash
   ./gradlew :app:installDebug --no-build-cache --stacktrace
   ```

   Need an APK instead of direct install?

   ```bash
   ./gradlew :app:assembleDebug
   adb install -t -r -d app/build/outputs/apk/debug/app-debug.apk
   ```

   4. **Full reinstall checklist (after wiping `node_modules` or cloning fresh)** – run this when
      you have reinstalled dependencies or when Gradle/expo-native modules behave as if they are
      missing. The commands rebuild the workspace, confirm that critical Expo plugins resolved
      correctly, and reinstall the dev client from a clean slate:

   ```bash
   npm ci

   # Sanity: verify Gradle/autolinking plugins materialised under node_modules
   test -d node_modules/@react-native/gradle-plugin && echo "RN gradle plugin OK"
   test -d node_modules/expo-modules-autolinking/android && echo "Expo autolinking OK"
   test -d node_modules/expo/packages/expo-gradle-plugin && echo "Expo gradle plugin OK"

   cd mobile/android

   ./gradlew --stop || true
   rm -rf ~/.gradle/kotlin .cxx app/build app/.cxx
   ./gradlew clean

   adb kill-server && adb start-server
   adb devices -l    # must show ONE device as 'device'

   adb shell am force-stop com.ahmedmonib.eshop.dev || true
   adb shell pm clear     com.ahmedmonib.eshop.dev || true
   adb shell pm uninstall --user 0 com.ahmedmonib.eshop.dev || true
   adb shell pm uninstall --user 0 com.ahmedmonib.eshop     || true

   # Install the dev client
   ./gradlew :app:installDebug --no-build-cache --stacktrace
   ```

   Use this flow when you switch Git branches that touch native code, after major Expo SDK bumps, or
   whenever a device reinstall fails with mysterious "native module not found" crashes.

---

## Daily development loop

Default workflow: **Tunnel Metro + ngrok HTTPS API** (works anywhere). Swap steps using other modes
from [Mobile run modes](./env.md#mobile-run-modes).

1. **Backend & optional frontend**

   ```bash
   npm run dev
   # backend → https://localhost:5001
   # frontend → https://localhost:5173
   ```

2. **Expose API via HTTPS**

   ```bash
   npx ngrok http https://localhost:5001
   npm -w mobile run env:tunnel  # update .env with new ngrok URL
   ```

3. **Start Metro (tunnel)** – always use the dev-client flag so the CLI opens the custom dev client
   rather than Expo Go, and clear Metro's cache after changing env profiles:

   ```bash
   cd mobile
   npx expo start --tunnel --dev-client --clear
   ```

4. **Open the dev client** by scanning the QR code (choose “Development Build” if prompted).

5. **Smoke test deep link**:

   ```bash
   npx uri-scheme open "eshop://reset-password?token=TEST123&email=user%40example.com" --android
   ```

Sanity checks: tunnel URL appears in Expo CLI (`u.expo.dev/...`), mobile browser reaches the ngrok
API URL, and Metro logs show `🔗` messages when the deep link fires.

---

## Frequently used commands

```bash
npm run dev                                # backend + frontend together
npm -w mobile run start:tunnel             # Expo tunnel with cache bust
npm -w mobile run tunnel:api               # ngrok helper (if defined)
adb devices -l                             # list devices
adb reverse tcp:8081 tcp:8081              # expose Metro to emulator
adb reverse tcp:19000 tcp:19000            # expose dev menu to emulator
adb install -r mobile/android/app/build/outputs/apk/debug/app-debug.apk
adb pair <ip>:<pair-port> && adb connect <ip>:<debug-port>
npx vercel --prod --yes                    # manual production deploy (frontend)
git commit -m "msg" --no-verify            # bypass husky if ever needed
```

---

## Add or reset a device

1. Enable **Developer options** and **USB debugging** on the device.
2. Install via USB or Wi-Fi:
   - USB: follow the [USB workflow](#usb-via-usbipd-win), then run `./gradlew installDebug` or
     `adb install -r ...`.
   - Wi-Fi: `adb pair`, `adb connect`, then install using `adb install -r ...`.

3. Launch a dev session using the [daily loop](#daily-development-loop).
4. If the device shows `unauthorized`, unlock it and accept the RSA prompt, then rerun
   `adb devices -l`.

To remove a stale build:

```bash
adb uninstall com.ahmedmonib.eshop
adb shell pm clear com.ahmedmonib.eshop
```

---

## Clean builds & cache resets

Use when Gradle caches become corrupted or when Metro serves stale bundles.

```bash
cd mobile/android
./gradlew --stop || true
rm -rf ~/.gradle/caches .cxx
./gradlew clean
```

Reset Metro cache:

```bash
cd mobile
npx expo start --tunnel -c   # or --localhost -c for emulator
```

Verify `local.properties` if Gradle references Windows paths:

```
cmake.dir=/usr
sdk.dir=/home/<you>/Android
ndk.dir=/home/<you>/Android/ndk/27.1.12297006
```

---

## Troubleshooting

- **`adb devices` empty** → Reattach with `usbipd attach --wsl`, ensure `lsusb` shows the device,
  restart ADB, and accept prompts.
- **Expo launches Expo Go** → Choose “Development Build” after scanning the QR; ensure custom dev
  client is installed.
- **Build fails with CMake/NDK errors** → Wipe caches (`rm -rf ~/.gradle/caches .cxx`), confirm
  `local.properties` paths, rerun `./gradlew clean`.
  - **`./gradlew :app:installDebug` hangs at 99%** → A release build from the Play Store uses the
    same `applicationId` and cannot be replaced by the debug keystore. Either uninstall the store
    build from the device or use the new debug-only suffix (`com.ahmedmonib.eshop.debug`). Re-run
    the install once the device shows `adb devices` and the installer will finish within a few
    seconds.
- **Metro cannot reach API** → Open the API URL in mobile browser (LAN/ngrok), check `.env` profile,
  restart Metro with `-c`.
- **Deep link does nothing** → Run the URI command above and inspect Metro logs for `🔗` prefixes;
  confirm backend emails still send the `deepLink` param.
- **ngrok closes unexpectedly** → Keep the terminal open or use a paid plan; update `.env.tunnel`
  after every restart.
  - **“Cannot find native module …” after reinstalling** → Confirm the installed package matches the
    debug dev client: `adb shell pm list packages | grep eshop` then
    `adb shell dumpsys package com.ahmedmonib.eshop.dev | sed -n '1,40p'`. If the package name or
    version is stale, uninstall again and reinstall with `./gradlew :app:installDebug`.

---

## Automation script

Optional bootstrap script to install the Android toolchain. Save as `scripts/setup-wsl-android.sh`,
mark executable, then run once.

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

After the script completes, reconnect your device (`usbipd attach --wsl` or `adb connect ...`) and
follow the [daily development loop](#daily-development-loop).

---
