# Android build configuration notes

This project relies on several custom Gradle settings that Expo's prebuild step generated. They are
required to build the Android application bundle that is shipped to Google Play.

## `mobile/android/app/build.gradle`

### Custom `buildTypes`

The `debug` block that adds `applicationIdSuffix ".dev"`, overrides the app name with `Eshop Dev`,
and uses the debug keystore lets you install the debug build alongside the release build and avoid
accidentally shipping a debug-signed bundle. Removing it does not break release builds, but it makes
local debugging harder because the debug build would share the same application id and label as
production.

If you already have the `Eshop Dev` build installed on a device, updating the Gradle files does not
force you to reinstall it. The next time you run `npx expo run:android --variant debug` or
`./gradlew installDebug`, the package with the `.dev` suffix is updated in place. You only need to
reinstall manually if you previously removed the `applicationIdSuffix` and installed a build that
shares the production id, because Android treats those as different apps.

### Dependency resolution strategy

The `configurations.all { resolutionStrategy { ... } }` section pins
`com.google.android.material:material:1.12.0` and `androidx.core` 1.13.1. Without the forced
versions Gradle may pull transitive versions that are too old for `compileSdkVersion 35`, causing
resource merge errors or runtime crashes. Keep the block in place for consistent release builds.

## `mobile/android/build.gradle`

The `ext { ... }` block sets `compileSdkVersion`, `targetSdkVersion`, `minSdkVersion`,
`buildToolsVersion`, and `ndkVersion`. These values are read in `app/build.gradle` via
`rootProject.ext.*`. If you remove the block the build fails because those properties become
undefined, and even if it compiles you risk targeting an SDK level that Play Console rejects.
Restore the block before creating a release bundle.

## `mobile/android/gradle.properties`

- `newArchEnabled=false` keeps the app on the stable React Native architecture. Expo SDK 54 still
  ships the new architecture behind a flag, so disabling it avoids unexpected native crashes in
  release builds.
- `reactNativeArchitectures=arm64-v8a` (optionally with `armeabi-v7a`) controls which ABIs Gradle
  packages. Play requires 64-bit binaries, so make sure this includes `arm64-v8a` before uploading
  an AAB.
- `android.ndkVersion=27.1.12297006` must match the value in the `ext` block so that the native
  dependencies compile with a known-good NDK. Leaving it undefined can cause build failures on CI
  where a different NDK is installed.

## What to restore before shipping

Before you run `./gradlew clean bundleRelease` or upload to Play, restore the `ext { ... }` block
and the dependency resolution overrides. Keeping the debug `buildTypes` block and the
`gradle.properties` entries is strongly recommended so you can reproduce the builds that have
already been tested.
