# Android Release & Google Play Console Guide

This checklist walks through everything needed to cut a signed Android release for the Expo-based
mobile app and upload it to the Google Play Console. Follow the steps in order — the release Gradle
tasks will now fail fast if any signing secrets are missing.

---

## 1. Prerequisites

- Node 20.x and npm (matching the repo).
- Java 17 (required by the React Native Gradle plugin).
- Android SDK + Android 14 (API level 34) platform tools.
- Expo CLI installed globally (`npm install -g expo-cli`) or runnable via `npx`.
- Access to the Google Play Console project with permission to create internal testing releases.
- A secure place to store long-lived signing credentials (e.g. 1Password, Bitwarden, HashiCorp
  Vault).

> ℹ️ All commands below assume you run them from the repository root unless noted otherwise.

---

## 2. Generate the upload keystore (one-time)

Create a private signing key that Gradle will use when producing release bundles. The keystore
**must never** be committed to version control.

```bash
cd mobile/android/app
keytool -genkeypair \
  -v \
  -storetype PKCS12 \
  -keystore eshop-upload-keystore.jks \
  -alias eshopUploadKey \
  -keyalg RSA \
  -keysize 2048 \
  -validity 36500
```

During the prompt:

- **Keystore password** → choose a strong unique password. Use the same value whenever the command
  is needed (store it in your password manager as `ESHOP_ANDROID_KEYSTORE_PASSWORD`).
- **Key password** → usually matches the keystore password (store as `ESHOP_ANDROID_KEY_PASSWORD`).
- **First and last name/organization/country** → use your company/publisher details.

Move the generated file somewhere outside the repo (e.g. `~/Dev/secrets/eshop-upload-keystore.jks`)
and delete the copy from `mobile/android/app/` to avoid accidental commits. Update the docs and
password manager with the new absolute path you choose.

```bash
mkdir -p ~/Dev/secrets
mv mobile/android/app/eshop-upload-keystore.jks ~/Dev/secrets/eshop-upload-keystore.jks
```

Update your password manager with:

- `ESHOP_ANDROID_KEYSTORE_PASSWORD = <your strong password>`
- `ESHOP_ANDROID_KEY_PASSWORD = <your strong password>`
- `ESHOP_ANDROID_KEY_ALIAS = eshopUploadKey`
- `ESHOP_ANDROID_KEYSTORE = /absolute/path/to/eshop-upload-keystore.jks`

### Pick and use a password manager

If you do not already have a password manager, choose one that the team approves. Popular options:

- **1Password** (paid, supports shared vaults and CLI access)
- **Bitwarden** (open source, offers a free tier and organization sharing)
- **LastPass Teams** or a self-hosted option such as **Passbolt**

Create a new vault item named something like `Eshop Android Signing` and store the four values above
plus the keystore file path. Many password managers allow you to attach the `.jks` file directly for
safe keeping. Limit access to release engineers only and never paste the passwords into chat tools.

#### Step-by-step Bitwarden setup

1. **Create your account** — sign in at [vault.bitwarden.com](https://vault.bitwarden.com) (or the
   self-hosted instance your team uses). Complete email verification so the vault can sync.
2. **Install a client** — optionally install the desktop or mobile app and the browser extension so
   you can copy values without exposing them in plain text. The desktop app makes attaching files
   easier.
3. **Create a vault item** — choose **Add Item → Secure Note** and name it `Eshop Android Signing`.
   In the note body, add the four key/value pairs and the keystore path:

   ```
   ESHOP_ANDROID_KEYSTORE =/home/<you>/Dev/secrets/eshop-upload-keystore.jks
   ESHOP_ANDROID_KEYSTORE_PASSWORD = ****************
   ESHOP_ANDROID_KEY_ALIAS = eshopUploadKey
   ESHOP_ANDROID_KEY_PASSWORD = ****************
   ```

4. **Store passwords as custom fields** — add custom fields for each variable so Bitwarden can fill
   them individually later. Choose the **Hidden** type for passwords.
5. **Attach the keystore (optional but recommended)** — use **File Attachment → Upload** and add the
   `eshop-upload-keystore.jks` file from `~/Dev/secrets/`. This keeps a backup inside the vault
   while the original stays on disk for Gradle to read.
6. **Share with your team** — if you use a Bitwarden Organization, move the item into the
   appropriate collection so other release maintainers can access it. Do **not** share outside the
   trusted group.
7. **Retrieve values safely** — when you need the passwords, use the Bitwarden app or browser
   extension to copy them to the clipboard (Bitwarden clears the clipboard automatically after a
   timeout). Avoid pasting them into plain text documents.

---

## 3. Provide signing secrets to Gradle

The Android build now reads secrets from either environment variables or Gradle properties. Pick one
of the two options below.

### Option A — export environment variables (for CI or one-off builds)

1. Ensure the keystore exists at the path you saved (e.g.
   `~/Dev/secrets/eshop-upload-keystore.jks`).
2. Export the variables in the same shell session that will run Gradle:

   ```bash
   export ESHOP_ANDROID_KEYSTORE=/home/user/Dev/secrets/eshop-upload-keystore.jks
   export ESHOP_ANDROID_KEYSTORE_PASSWORD='<your strong password>'
   export ESHOP_ANDROID_KEY_ALIAS='eshopUploadKey'
   export ESHOP_ANDROID_KEY_PASSWORD='<your strong password>'
   ```

3. Remember that exports only last for the life of the shell process. If you open a new terminal you
   must export them again. To avoid retyping them, add the exports to a shell profile (`~/.zshrc`,
   `~/.bash_profile`) or use a tool like [`direnv`](https://direnv.net/) or `mise` to load them
   automatically when you `cd` into the repo. Never commit those profile files.
4. Verify the values are available before building:

   ```bash
   env | grep ESHOP_ANDROID
   ```

5. Double-check that `ESHOP_ANDROID_KEYSTORE` does **not** point to
   `mobile/android/app/debug.keystore`. That file is reserved for debug builds and the Play Store
   will reject bundles signed with it. Gradle now fails fast if the release config references the
   debug keystore, but it is best to confirm the path before building.
6. When finished, clear the variables with `unset ESHOP_ANDROID_KEYSTORE_PASSWORD` (and the others)
   if you are on a shared machine.

### Option B — persist in `~/.gradle/gradle.properties`

Append the following lines (with real values) to your local Gradle properties file. This keeps
secrets out of the repository while allowing Android Studio/Gradle to pick them up automatically.
Use an absolute path for `ESHOP_ANDROID_KEYSTORE` (e.g.
`/home/you/Dev/secrets/eshop-upload-keystore.jks`). Gradle does **not** expand tildes (`~`) inside
`gradle.properties`, so leaving the tilde will silently fall back to the debug keystore and Google
Play will still detect a debug signature.

```properties
# Eshop mobile release signing
ESHOP_ANDROID_KEYSTORE=/Users/you/Dev/secrets/eshop-upload-keystore.jks
ESHOP_ANDROID_KEYSTORE_PASSWORD=<your strong password>
ESHOP_ANDROID_KEY_ALIAS=eshopUploadKey
ESHOP_ANDROID_KEY_PASSWORD=<your strong password>
```

Steps:

1. Create the Gradle home if it does not already exist:

   ```bash
   mkdir -p ~/.gradle
   touch ~/.gradle/gradle.properties
   ```

2. Edit the file with your favorite editor (e.g. `code ~/.gradle/gradle.properties` or
   `nano ~/.gradle/gradle.properties`) and paste the properties above with real values. You should
   **not** copy these secrets into `mobile/android/gradle.properties`; that file stays checked in
   for documentation only.
3. Confirm the file permissions are restricted to your user (`chmod 600 ~/.gradle/gradle.properties`
   on macOS/Linux).
4. Because the file lives outside the repository, nothing extra is needed in `.gitignore`. If you
   ever create a per-project secret file (for example
   `mobile/android/app/release-signing.properties`), add it to `mobile/.gitignore` before saving
   secrets locally. Leave the commented reminder lines in `mobile/android/gradle.properties`
   untouched—they exist purely as documentation and should never be uncommented with real secrets.

> ❗ Do **not** check this file into source control. It lives under your user profile and is ignored
> by Git automatically.

💡 Run `./gradlew signingReport` from `mobile/android` to confirm Gradle can read the secrets before
starting a release build. The `Variant: release` block must show your upload keystore path and
alias, not `androiddebugkey`.

### Regenerate and validate the release bundle after fixing secrets

If Google Play previously rejected an `.aab` because it was signed with the debug keystore, delete
`mobile/android/app/build/outputs/bundle/release/app-release.aab` and rebuild from scratch:

```bash
cd mobile/android
./gradlew clean
./gradlew bundleRelease
```

Before uploading, double-check the signature inside the bundle matches your upload key alias:

```bash
jarsigner -verify -verbose -certs app/build/outputs/bundle/release/app-release.aab \
  | grep -A1 "sm      0"  # should list eshopUploadKey
```

## Only upload the freshly generated bundle after these checks pass

## 4. Bump the version before every store submission

1. Edit `mobile/app.config.js` and update both `version` (semantic string) and `android.versionCode`
   (positive integer, monotonically increasing).
2. Mirror the change in `mobile/android/app/build.gradle` under
   `defaultConfig { versionName "x.y.z"; versionCode N }`.

Example diff snippet:

```diff
-    version: '1.0.0',
+    version: '1.1.0',
...
-        versionCode 1
-        versionName "1.0.0"
+        versionCode 2
+        versionName "1.1.0"
```

Commit the version bump alongside any release notes changes.

---

## 5. Build artifacts

Always run production builds against the production backend so QA mimics what Play users will see.

### A. Production AAB (upload to Play)

```bash
# from repo root
npm install
npm run --workspace mobile expo-prebuild -- --clean   # optional: refresh native folders after config changes
npm -w mobile run env:production    # or: cp mobile/.env.production mobile/.env

cd mobile/android
./gradlew clean bundleRelease
```

Outputs:

- `mobile/android/app/build/outputs/bundle/release/app-release.aab` → upload this file to the Play
  Console.
- `mobile/android/app/build/outputs/mapping/release/mapping.txt` → archive alongside the AAB for
  crash symbolication.

### B. Optional APK (sideload for quick QA)

This artifact is not accepted for publishing but is ideal for smoke-testing on real hardware.

```bash
./gradlew assembleRelease
adb install -r app/build/outputs/apk/release/app-release.apk
```

Why it matters: sideloading lets you validate network traffic, deep links, and first-run flows
faster than waiting for Play’s review/install pipeline.

> ✅ Remember: when building the AAB/APK, ensure `mobile/.env` points to production. Do **not**
> reuse a dev/tunnel `.env`.

```bash
     npx uri-scheme open "eshop://reset-password?token=TEST123&email=user%40example.com" --android
```

     Verify the reset screen opens and the token/email parameters propagate through the UI.

## 6. Smoke-test the release build locally

Perform these checks **before** uploading anything to Play. Use the sideloaded `app-release.apk`,
not the Expo dev client.

### Prepare a test target

- **Physical Android device (USB or Wi-Fi)**
  1. Follow the **Connect a device** section in
     [`mobile/custom-dev-setup.md`](../custom-dev-setup.md) to expose your phone to WSL via
     `usbipd-win` (or pair over Wi-Fi after the first USB handshake).
  2. On the device, enable **Developer options → USB debugging** and accept the RSA prompt when you
     run `adb devices -l` from WSL. You should see a `device` status (not `unauthorized`).
  3. Install the release APK:

     ```bash
     cd mobile/android
     adb install -r app/build/outputs/apk/release/app-release.apk
     ```

     For multiple devices/emulators, scope the target with `adb -s <serial> install -r ...`.

- **Android Studio emulator**
  1. Open Android Studio → **More Actions → Virtual Device Manager** and create a Pixel/SDK 34 image
     if you do not already have one. Start the emulator.
  2. From WSL, confirm it appears with `adb devices -l` (typically `emulator-5554`).
  3. Install the APK to that emulator serial:

     ```bash
     cd mobile/android
     adb -s emulator-5554 install -r app/build/outputs/apk/release/app-release.apk
     ```

     Adjust the serial if Android Studio reports a different ID.

> 🔁 These steps reuse the same adb workflow documented for the custom dev client. The only
> difference is that you install the Gradle-generated release APK instead of `installDebug` output.

### Run the smoke tests

1. Ensure `mobile/.env` contains the production values (copy from `.env.production`). See the
   environment matrix in [`mobile/env.md`](../env.md) if you need a refresher on profiles.
2. Launch the installed release app on the device/emulator.
3. Validate production behavior end to end:
   - Sign in/out or create an account to verify the production API responds.
   - Browse products, add to cart, and attempt checkout flows.
   - Trigger the deep link via `uri-scheme`:

     ```bash
     npx uri-scheme open "eshop://reset-password?token=TEST123&email=user%40example.com" --android
     ```

     Confirm you land on the reset screen and the token/email propagate correctly.

4. If SSL pinning is enabled, double-check that the pinned certificate alias (`eshop_api`) matches
   the bundled certificate and the live backend TLS chain.
5. Fix any issues before uploading to Google Play.

---

## 7. Play Console upload & roll-out

1. Create or open the **Eshop** app in the Play Console. The package name **must** be
   `com.ahmedmonib.eshop`.
2. Complete the mandatory store configuration (only needed the first time, but review it for
   updates):
   - Store listing (icon, feature graphic, screenshots, descriptions).
   - App content (privacy policy URL, Data safety form, Ads?, Content rating questionnaire).
   - Play App Signing (accept the recommended option).
3. Navigate to **Testing → Internal testing → Create release**.
4. Upload `app-release.aab`, add release notes, resolve warnings, and roll out to the internal
   testers list.
5. Install the build from the Play Store on a tester device and repeat the smoke tests above.
6. ***

## Troubleshooting: Play reports a debug signature

If the Play Console rejects your upload with
`You uploaded an APK or Android App Bundle that was signed in debug mode`, Gradle produced a bundle
that still uses the debug keystore. Walk through the checks below before rebuilding the AAB:

1. **Inspect the signing report**

   ```bash
   cd mobile/android
   ./gradlew :app:signingReport
   ```

   In the output, the `Variant: release` section must list `Config: release` with the alias you
   configured (for example `eshopUploadKey`) and the keystore path you saved in your password
   manager. If you see `Config: debug` or the path `mobile/android/app/debug.keystore`, Gradle is
   still falling back to the debug credentials.

2. **Fix the secret source**
   - **Environment variables** — re-run the `export ESHOP_ANDROID_*` commands in the same shell
     before running Gradle, and confirm the values with `env | grep ESHOP_ANDROID`.
   - **`~/.gradle/gradle.properties`** — open the file and verify each property matches the
     generated keystore. Avoid wrapping the values in quotes and make sure the path is absolute (for
     WSL paths, prefer `/mnt/c/...` over `C:\...`).
   - When migrating secrets between machines, copy both the `.jks` file and the four property values
     at the same time.

3. **Confirm the keystore path**

   The keystore **must not** point to `mobile/android/app/debug.keystore`. That file is required for
   local debug builds—do not delete it—but it cannot be used for releases. Move your upload keystore
   outside the repo (e.g. `~/Dev/secrets/eshop-upload-keystore.jks`) and update the property/path to
   match.

4. **Clean and rebuild**

   ```bash
   cd mobile/android
   ./gradlew clean
   ./gradlew bundleRelease
   ```

   After the build finishes, re-run the signing report and verify that the release variant now shows
   the release keystore. Only upload the new `app-release.aab` to the Play Console once the check
   passes.

Following these steps ensures Play receives a bundle signed with the long-lived upload keystore.

7. When satisfied, promote the existing artifact to Closed or Production tracks—no rebuild required.

---

## 8. Dev vs. prod workflow (quick reference)

Keep your environment clean between daily development and release prep:

**Daily dev (tunnel)**

```bash
npm -w mobile run env:tunnel
cd mobile
npx expo start --tunnel --dev-client
```

**Release build (prod)**

```bash
npm -w mobile run env:production
cd mobile/android
./gradlew clean bundleRelease
```

Never mix tunnel/dev clients with release builds. Before building, ensure `mobile/.env` contains the
production values (copy from `.env.production`).

---

## 9. Final quick checklist

- [ ] Railway `MOBILE_*` variables and `PUBLIC_CLIENT_FALLBACK_URL` configured for production.
- [ ] `mobile/.env.production` targets the production API and web origin; `mobile/.env` matches it
      at build time.
- [ ] `android/app/build.gradle` release build uses `signingConfigs.release` (not debug).
- [ ] `versionCode` incremented and `versionName` updated in both `mobile/app.config.js` and
      `mobile/android/app/build.gradle`.
- [ ] `app-release.aab` built, archived with `mapping.txt`, and smoke-tested via the sideloaded
      `app-release.apk`.
- [ ] Internal testing release created first, then promoted to Production when ready.

Tag the release commit (e.g. `git tag mobile-v1.1.0 && git push --tags`), store artifacts securely,
monitor Play Console vitals, and update documentation or changelogs as needed.

---

## 10. Troubleshooting & recurring release workflow

| Symptom                                            | Fix                                                                                                         |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `Missing release keystore configuration` error     | Ensure all four `ESHOP_ANDROID_*` values are exported or defined in `~/.gradle/gradle.properties`.          |
| `Release keystore not found` error                 | Double-check the keystore path (absolute paths are safest).                                                 |
| Google Play rejects the bundle due to version code | Increment `android.versionCode` in both `app.config.js` and `build.gradle`.                                 |
| APK installs but crashes on launch                 | Check `adb logcat`, confirm the production API URL responds, and verify `npm install` ran before the build. |

### Repeat for every future store submission

1. Pull the latest `main` and merge your feature branch.
2. Bump `version`/`versionCode`.
3. Run `npm install` and refresh native code if configuration changed
   (`npm run --workspace mobile expo-prebuild --clean`).
4. Switch the environment to production (`npm -w mobile run env:production`).
5. Build the release AAB and optional APK (`./gradlew clean bundleRelease` and
   `./gradlew assembleRelease`).
6. Smoke-test using the sideloaded APK and deep-link command above.
7. Upload the fresh `app-release.aab` to Internal testing, verify, then promote to wider tracks.
8. Tag the release, archive artifacts, and monitor telemetry.

Following this loop ensures each Play Store update uses the same hardened process and reduces missed
steps during future deploys.

---
