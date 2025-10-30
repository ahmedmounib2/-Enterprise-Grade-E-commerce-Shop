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
  - [After running `expo prebuild`](#after-running-expo-prebuild)
    - [Minimal, reproducible flow we use](#minimal-reproducible-flow-we-use)
    - [Verify autolinking (do these exact checks)](#verify-autolinking-do-these-exact-checks)
  - [Build or refresh the custom dev client](#build-or-refresh-the-custom-dev-client)
    - [One-time or after native changes](#one-time-or-after-native-changes)
    - [Install the **Internal Debug** dev client (the one you use daily)](#install-the-internal-debug-dev-client-the-one-you-use-daily)
    - [Full reinstall (after `node_modules` wipe or fresh clone)](#full-reinstall-after-node_modules-wipe-or-fresh-clone)
  - [Daily development loop](#daily-development-loop)
  - [Frequently used commands](#frequently-used-commands)
  - [Add or reset a device](#add-or-reset-a-device)
  - [Clean builds \& cache resets](#clean-builds--cache-resets)
  - [Examples of “native-affecting” changes in app.config.js](#examples-of-native-affecting-changes-in-appconfigjs)
  - [Troubleshooting](#troubleshooting)
  - [Automation script](#automation-script)
    - [Extra: Building the **Internal Release** APK (optional)](#extra-building-the-internal-release-apk-optional)
  - [Fix the 99% hang \& reinstall the dev client](#fix-the-99-hang--reinstall-the-dev-client)
  - [Install It](#install-it)

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
usbipd bind   --busid <BUSID>
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

2. **No axios deep imports** (use public API only).
3. **Android wrappers**: `MainApplication.kt` uses `ReactNativeHostWrapper`; `MainActivity.kt`
   returns a `ReactActivityDelegateWrapper`.
4. **Manifest** `<application android:name=".MainApplication" .../>`.
5. **Gradle uses Expo plugin**: root `mobile/android/build.gradle` has the Expo classpath;
   `app/build.gradle` applies `expo`.

---

## After running `expo prebuild`

Use this only when adding native deps (e.g. `react-native-svg`). Prebuild rewrites `mobile/android`,
then you restore your customizations.

### Minimal, reproducible flow we use

```bash
cd mobile
npx expo prebuild --platform android --clean

# Restore our Android customizations from git (important files below)
#   mobile/android/build.gradle
#   mobile/android/gradle.properties
#   mobile/android/app/build.gradle
#   mobile/android/app/src/main/AndroidManifest.xml
```

### Verify autolinking (do these exact checks)

```bash
#  Gradle runtime classpath (flavored debug)
cd mobile/android
./gradlew :app:dependencies --configuration internalDebugRuntimeClasspath \
  | grep -i -E "react-native-svg|horcrux|svg" || true
```

> Newer RN/Expo may not add explicit lines to `settings.gradle`; that’s normal.

When these checks pass, proceed to (re)install the dev client below.

---

## Build or refresh the custom dev client

> **This is the canonical way to (re)install your dev client** (flavor = `internal`, buildType =
> `debug`).

### One-time or after native changes

```bash
# 0) Set the JS env used by the dev client (tunnel is the default)
npm -w mobile run env:tunnel

# 1) Make sure the Android SDK path file exists
ls mobile/android/local.properties || printf "\nCreate mobile/android/local.properties with sdk.dir=/home/<you>/Android\n\n"

# 2) Clean Gradle (safe if you touched native deps)
cd mobile/android
./gradlew --stop || true
rm -rf ~/.gradle/kotlin .cxx app/build app/.cxx
./gradlew clean
```

### Install the **Internal Debug** dev client (the one you use daily)

```bash
 # Uninstall all variants from your device/emulator
   adb uninstall com.ahmedmonib.eshop || true
   adb uninstall com.ahmedmonib.eshop.internal || true
   adb uninstall com.ahmedmonib.eshop.internal.dev || true

 # Clean the Android build
   cd mobile/android
   ./gradlew clean

  ./gradlew :app:assembleInternalDebug
  adb install -r app/build/outputs/apk/internal/debug/app-internal-debug.apk
  cd ../..

#if its stuck run this to verify its moving in another terminal
adb logcat | rg INSTALL
```

- App label on the device: **“Eshop Debug”** (or similar).
- Package name: **`com.ahmedmonib.eshop.internal`**.
- If it hangs at 99% or fails with `INSTALL_FAILED_UPDATE_INCOMPATIBLE`, uninstall the same package
  first (see Troubleshooting).

**Need an APK instead of direct install?**

```bash

./gradlew :app:assembleInternalDebug
adb install -r app/build/outputs/apk/internal/debug/app-internal-debug.apk
```

### Full reinstall (after `node_modules` wipe or fresh clone)

```bash
# repo root
rm -rf node_modules
npm install

# quick sanity
test -d node_modules/@react-native/gradle-plugin && echo "RN gradle plugin OK"
test -d node_modules/expo-modules-autolinking/android && echo "Expo autolinking OK"
test -d node_modules/expo/packages/expo-gradle-plugin && echo "Expo gradle plugin OK"

# install the dev client
cd mobile/android
./gradlew --stop || true
rm -rf ~/.gradle/kotlin .cxx app/build app/.cxx
./gradlew clean
adb kill-server && adb start-server
adb devices -l  # must show ONE 'device'

# remove conflicting installs
adb uninstall com.ahmedmonib.eshop.internal || true

# reinstall dev client
./gradlew :app:installInternalDebug --no-build-cache --stacktrace
```

> **Note:** Debug and Internal Release share the same package `com.ahmedmonib.eshop.internal`.
> Because they’re signed with different keys, you **must uninstall** one before installing the
> other.

---

## Daily development loop

```bash
# 1) Start backend (+ optional web)
npm run dev
# backend → https://localhost:5001
# frontend → https://localhost:5173

# 2) Expose API via ngrok and sync the env
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
adb install -r mobile/android/app/build/outputs/apk/internal/debug/app-internal-debug.apk
adb pair <ip>:<pair-port> && adb connect <ip>:<debug-port>
git commit -m "msg" --no-verify            # bypass husky if needed
```

---

## Add or reset a device

1. Enable **Developer options** + **USB debugging**.
2. Install via USB or Wi-Fi:
   - **USB**: follow [USB workflow](#usb-via-usbipd-win), then
     `./gradlew :app:installInternalDebug`.
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
npx expo prebuild -p android
npx expo run:android --variant internalDebug
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
- **`./gradlew :app:installInternalDebug` hangs ~99%** → Usually a **signature/package conflict**.
  Fix:

  ```bash
  adb uninstall com.ahmedmonib.eshop.internal
  ./gradlew :app:installInternalDebug
  ```

- **`INSTALL_FAILED_UPDATE_INCOMPATIBLE`** when switching between **Internal Release** and
  **Debug**:
  - They share `com.ahmedmonib.eshop.internal` but are signed differently. Uninstall first:

    ```bash
    adb uninstall com.ahmedmonib.eshop.internal
    ```

- **Metro can’t reach API** → Open the API URL on the device, confirm `.env` profile, restart Metro
  with `-c`.
- **Deep link no-op** → Trigger with `uri-scheme` and watch Metro logs (`🔗` events). Verify email
  links still include `deepLink`.
- **ngrok restarts** → Keep the terminal open; run `npm -w mobile run env:tunnel` after each
  restart.

---

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

### Extra: Building the **Internal Release** APK (optional)

If you need a side-loadable release APK for quick QA:

```bash
# (repo root) set a non-pinned internal env if you have one
npm -w mobile run env:internal

# build & install
cd mobile/android
./gradlew clean :app:assembleInternalRelease
adb uninstall com.ahmedmonib.eshop.internal || true
adb install -r app/build/outputs/apk/internal/release/app-internal-release.apk
```

Remember: it shares the package with the debug dev client. Uninstall before switching.

## Fix the 99% hang & reinstall the dev client

```bash
# 0) Use the dev client JS env you want (tunnel is the daily-default)

npm -w mobile run env:tunnel

# 1) Make sure only ONE device is connected

adb devices -l

# 2) Remove any existing internal package (debug or release)

adb uninstall com.ahmedmonib.eshop.internal || true

# If you ever see "not installed for 0", force-uninstall for the current user

adb shell pm uninstall --user 0 com.ahmedmonib.eshop.internal || true

# 3) (Optional) clear any temp APKs to avoid partial installs

adb shell rm -f /data/local/tmp/*.apk || true

# 4) Clean just enough Gradle state (no need to nuke everything)

cd mobile/android
./gradlew --stop || true
./gradlew clean

# build APK

./gradlew :app:assembleInternalDebug
```

## Install It

```bash
adb install -r app/build/outputs/apk/internal/debug/app-internal-debug.apk
```

---
