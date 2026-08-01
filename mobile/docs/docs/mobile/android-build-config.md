# Android build configuration notes

The Android project is **generated**, not committed. `mobile/android/` and `mobile/ios/` are
git-ignored; `expo prebuild` (locally) and EAS Build (in the cloud) recreate them from
`mobile/app.config.js` on every build. Anything you edit inside `mobile/android/` is discarded on
the next prebuild.

So the rule is: **change `app.config.js` or a config plugin, never the generated Gradle files.** If
you need something Expo does not expose as a config field, write a config plugin in
`mobile/plugins/` — that is what the existing four do.

## Where the SDK levels come from

Three layers, each overriding the one above:

1. **React Native's Gradle version catalog** —
   `node_modules/react-native/gradle/libs.versions.toml`. For React Native 0.81 / Expo SDK 54 this
   declares `compileSdk = 36`, `targetSdk = 36`, `buildTools = "36.0.0"`, `minSdk = 24`,
   `ndkVersion = "27.1.12297006"`, AGP `8.11.0`, Kotlin `2.1.20`.
2. **Expo's root-project plugin** reads that catalog and publishes the values as
   `rootProject.ext.compileSdkVersion` etc., which the generated `app/build.gradle` consumes. It
   prints them at configuration time under `[ExpoRootProject] Using the following versions:` — the
   quickest way to confirm what a build actually used.
3. **`expo-build-properties`** in `app.config.js` writes `android.compileSdkVersion` /
   `android.targetSdkVersion` / `android.minSdkVersion` into `gradle.properties`, which override the
   catalog.

We pin layer 3 explicitly so that an Expo SDK upgrade shows up as a deliberate edit here rather than
a silent shift underneath the app. The pin must therefore be **kept in step with the Expo defaults**
— it once sat at 35 while SDK 54 had already moved to 36, which quietly held the app a level behind.

Current values (Android 16):

```
compileSdkVersion  36
targetSdkVersion   36   # Google Play requires this for uploads from 2026-08-31
minSdkVersion      24
buildToolsVersion  (omitted — AGP resolves 36.0.0 from the catalog)
```

Building locally against API 36 needs the platform installed:

```bash
sdkmanager "platforms;android-36" "build-tools;36.0.0"
```

EAS Build images already ship them, so this only affects manual `./gradlew` builds.

## New Architecture

`newArchEnabled=false` in the generated `gradle.properties` comes from `app.config.js`.

**The dependency that blocked it is gone.** `react-native-ssl-pinning@1.6.0` was a legacy bridge
module (`ReactContextBaseJavaModule`, no `codegenConfig`) reached through the legacy
`NativeModules.RNSslPinning` proxy, sitting on the certificate-pinned path of every authenticated
API call. It has been replaced by `react-native-ssl-public-key-pinning`, which ships a TurboModule
spec (`codegenConfig: { name: RNSslPublicKeyPinningSpec, type: modules }`) and resolves through
`TurboModuleRegistry.get` when `global.__turboModuleProxy` is present, falling back to
`NativeModules` otherwise — so it works under both architectures.

Readiness of every remaining native dependency, verified at the pinned versions:

| Dependency                                                                       | Status                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `expo` 54 and all `expo-*` modules                                               | Ready — SDK 54 defaults New Arch on for new projects.                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `@react-navigation/*` v7                                                         | JS only.                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `react-native-screens`, `safe-area-context`, `gesture-handler`, `svg`, `webview` | Ready — all ship a `codegenConfig`.                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `react-native-ssl-public-key-pinning`                                            | Ready — TurboModule spec.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `react-native-keyboard-aware-scroll-view` 0.9.5                                  | Compatible. Unmaintained since ~2020, so it was audited by source: `UIManager.viewIsDescendantOf` is implemented for bridgeless in `BridgelessUIManager.js` (via `FabricUIManager.compareDocumentPosition`), `UIManager.measureInWindow` dispatches Fabric tags in `UIManager.js`, and `scrollResponderScrollNativeHandleToKeyboard` plus `TextInput.State.currentlyFocusedInput` both still exist in RN 0.81.5. It is also currently unimported anywhere in `mobile/src`. |
| Custom native modules                                                            | None.                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |

So there is **no remaining blocker** — only cost. Enabling Fabric re-renders every screen and needs
a full-app device pass (RTL, modals, lists, keyboard handling), which is unrelated to any pinning
work. It therefore stays off until it can be done as its own change, so that any regression is
attributable to a single architectural switch.

Still a deadline, not a preference: **Expo SDK 55 removes the legacy architecture entirely** and
`newArchEnabled` disappears from the config. The follow-up task is to flip `newArchEnabled`, upgrade
to SDK 55, re-run the device matrix, and verify keyboard-aware scrolling behaviourally — replacing
it with `react-native-keyboard-controller` only if it actually misbehaves.

## Config plugins (`mobile/plugins/`)

| Plugin                  | What it does                                                                                                                                                                                           | Removable when                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------- |
| `withReleaseSigning`    | Injects `signingConfigs.release` into the generated `app/build.gradle` (reads `ESHOP_ANDROID_*`, defers to EAS's injected keystore on EAS Build, throws on a local release build with no credentials). | Never — required for manual release builds. |
| `withLargeScreenCompat` | Adds `android.window.PROPERTY_COMPAT_ALLOW_RESTRICTED_RESIZABILITY` to `<application>`. **Temporary.**                                                                                                 | See below.                                  |

Two pinning-related plugins were deleted alongside the `react-native-ssl-public-key-pinning`
migration. `withSslPinningCerts` copied `mobile/certs/*.cer` into the generated Android assets; pins
are now base64 SPKI hashes in `EXPO_PUBLIC_SSL_PINNING_HASHES`, so there is no file to copy — and no
Android-only asymmetry (the old plugin never ran for iOS, which therefore was never pinned).
`withSSLPinningFix` disabled `verifyReleaseResources` across subprojects because
`react-native-ssl-pinning@1.6.0` referenced `android:attr/lStar` in its compiled resource table;
with that dependency gone, AGP's static check passes on its own.

## Large screens and the compatibility opt-out

Apps targeting API 36 have `android:screenOrientation`, `android:resizableActivity` and the
aspect-ratio attributes **ignored** on displays whose smallest width is ≥600dp — tablets, foldables,
desktop windowing. Our `MainActivity` declares `screenOrientation="portrait"`, so without an opt-out
the app would silently become rotatable and resizable there.

`withLargeScreenCompat` restores the pre-Android-16 behaviour. Google honours the property only
until apps target API 37, so it buys roughly one release cycle.

Note it does **not** make the app phone-only: a tablet in portrait still hands us an ~800dp window,
and `src/hooks/useResponsiveColumns.js` correctly renders that as a 3-column grid. The property
blocks rotation and resizing, not large windows.

Before deleting the plugin, all of these must hold. The first two are already done:

1. ~~No module-scope `Dimensions.get()` — layout constants must come from `useWindowDimensions()`~~
   so they survive rotation, fold and split-screen resize.
2. ~~Grids derive their column count from window width~~ (`useResponsiveColumns`, applied across all
   ten product/category grids; breakpoints pinned by
   `src/tests/hooks/useResponsiveColumns.test.js`).
3. `Navbar`, `Footer` and the modal sheets verified in landscape at ≥600dp.
4. Arabic / RTL verified in landscape — RTL plus a wide layout is the least-tested combination.
5. Manual pass on a tablet emulator **and** a foldable emulator: portrait, landscape, and 50/50
   split-screen.

Then delete `plugins/withLargeScreenCompat.js`, drop it from `plugins` in `app.config.js`, and bump
`versionCode`.

## Other API 36 behaviour changes, and why they are already handled

- **Edge-to-edge is enforced with no opt-out.** Already enabled (`edgeToEdgeEnabled: true`), and
  insets come from `react-native-safe-area-context` throughout. No change.
- **Predictive back is on by default**, which stops dispatching `onBackPressed` / `KEYCODE_BACK`.
  Expo generates `android:enableOnBackInvokedCallback="false"`, so we are explicitly opted out, and
  React Native's `ReactActivity` already routes through `OnBackPressedDispatcher` regardless.
- **16 KB page sizes.** React Native 0.81 / Expo SDK 54 native libraries are already aligned, and
  `react-native-ssl-public-key-pinning` ships no `.so` (it is pure Kotlin/Java over OkHttp).

## Versioning

`version` and `android.versionCode` in `app.config.js` are the single source of truth — `eas.json`
sets `appVersionSource: "local"` and `autoIncrement: false`, so nothing bumps them for you. The
generated `app/build.gradle` merely mirrors them.

`runtimeVersion.policy` is `appVersion`, so bumping `version` fences a new native binary off from
OTA updates published for the previous one. That is deliberate — a targetSdk change is native and
cannot ship over EAS Update, so it _must_ come with a version bump and a fresh store build.

## Verifying a config change

```bash
cd mobile
npx expo config --type prebuild          # resolved config, before any native files are written
npx expo prebuild --platform android --clean --no-install

grep -E 'android\.(compile|target|min)SdkVersion' android/gradle.properties
grep -E 'versionCode|versionName' android/app/build.gradle
grep -o '<property[^/]*/>' android/app/src/main/AndroidManifest.xml
npx expo-doctor
```
