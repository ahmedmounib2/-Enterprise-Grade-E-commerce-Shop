# Mobile App Branding Guide

This guide documents the branding-related settings for the Expo-based Android app in `mobile/` and
explains where each value lives, what it changes in the product, and whether you must ship a new
Android App Bundle (AAB) after editing it. Use it as a quick reference before every rebrand or
release.

## Summary Table

| Thing                                                 | Where you change it                                                                                                                   | What changes in the product                                     | Rebuild needed?                                                       |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------- |
| Launcher label (name under the icon)                  | `mobile/app.config.js` → `expo.name`                                                                                                  | Name under the launcher icon, recent apps, notification headers | ✅ Yes – create a new AAB and bump the version code                   |
| Bundle identifier / package name                      | `mobile/app.config.js` → `ios.bundleIdentifier`, `android.package`                                                                    | Store identity, app links, update channel                       | ✅ Only before the first upload. After publishing, treat as permanent |
| App icon (standard + adaptive)                        | `mobile/app.config.js` → `expo.icon`, `android.adaptiveIcon.*` + PNGs in `mobile/assets/`                                             | Icon shown on the home screen and app drawer                    | ✅ Yes – rebuild with the new artwork                                 |
| Splash screen artwork                                 | `mobile/app.config.js` → `splash.*` + referenced assets                                                                               | Launch screen shown while the JS bundle loads                   | ✅ Yes – rebuild                                                      |
| Deep-link scheme / intent filters                     | `mobile/app.config.js` → `scheme`, `android.intentFilters`                                                                            | Which custom URLs open the app                                  | ✅ Yes – rebuild                                                      |
| Version name (Android)                                | `mobile/app.config.js` → `expo.version` **and** `mobile/android/app/build.gradle` → `defaultConfig.versionName`                       | Human-readable version in Android settings & Google Play        | ✅ Yes – rebuild and keep both files aligned                          |
| Version code (Android)                                | `mobile/android/app/build.gradle` → `defaultConfig.versionCode` (optionally mirror in `mobile/app.config.js` → `android.versionCode`) | Internal build number Google Play enforces                      | ✅ Yes – must increase with every AAB. Gradle value is authoritative  |
| Runtime permissions                                   | Config plugins or native overrides under `mobile/android/`                                                                            | Which Android permissions the binary requests                   | ✅ Yes – and Play may require additional declarations                 |
| API base URL and env flags                            | `mobile/app.config.js` → `extra.EXPO_PUBLIC_API_BASE_URL` (or `.env`)                                                                 | Backend endpoint bundled into the app                           | ✅ Yes – rebuild so the new value is embedded                         |
| Play Store title                                      | Google Play Console → **Grow → Store presence → Main store listing**                                                                  | Name shown in Play Store search/results                         | ❌ No – update listing only                                           |
| Store descriptions (short/full)                       | Play Console → Main store listing                                                                                                     | Marketing copy in Play Store                                    | ❌ No                                                                 |
| Store graphics (icon, feature graphic, screenshots)   | Play Console → Store listing graphics                                                                                                 | Marketing media on the Play Store page                          | ❌ No                                                                 |
| Category, tags, contact info, developer name          | Play Console → Store settings                                                                                                         | Metadata shown to customers                                     | ❌ No                                                                 |
| Privacy policy URL, Data safety form, ads declaration | Play Console → App content                                                                                                            | Compliance disclosures visible in Play                          | ❌ No (updates can trigger review)                                    |

## Detailed Guidance

### Launcher Label (App Name on Device)

- **File**: `mobile/app.config.js` → `expo.name`.
- **Impact**: Appears under the launcher icon, in recent apps, notification headers, and Android
  settings → App info.
- **Release note**: Changing it requires a new build. Always increment the version code before
  uploading to Play.

### Bundle Identifier / Package Name

- **Files**: `ios.bundleIdentifier` and `android.package` inside `mobile/app.config.js`.
- **Impact**: Controls update compatibility, deep-link routing, and store ownership.
- **Constraints**: Choose before your first upload. After any AAB hits a Play Console track, the
  package name is effectively permanent. Changing it means publishing a brand-new app listing.

### Icons

- **Files & assets**:
  - `expo.icon` for the default Expo icon.
  - `android.adaptiveIcon.foregroundImage` and `android.adaptiveIcon.backgroundColor` for adaptive
    icons.
  - PNG files inside `mobile/assets/` (`icon.png`, `adaptive-icon.png`, etc.).
- **Process**: Replace the PNGs with new artwork (same dimensions) and rebuild so the updated assets
  ship in the bundle.

### Splash Screen

- **Configuration**: `splash.image`, `splash.backgroundColor`, and `splash.resizeMode` in
  `mobile/app.config.js`.
- **Assets**: Stored alongside the other mobile assets (e.g. `mobile/assets/splash-icon.png`).
- **Note**: Any change requires a new build so the binary contains the new image.

### Deep-Link Scheme & Intent Filters

- **Config**: `scheme` and `android.intentFilters` in `mobile/app.config.js`.
- **Effect**: Determines which URLs (`eshop://...` or otherwise) launch the app. A rebuild is needed
  for Android to pick up the new manifest entries.

### Versioning (Answering “Do I bump both files?”)

- **What must change each release?** Increment both the version _name_ and _code_ before exporting a
  new AAB for Google Play.
- **Authoritative source**: Because this repository commits the native `android/` directory, the
  values in `mobile/android/app/build.gradle` (`defaultConfig.versionName` and
  `defaultConfig.versionCode`) are what end up in the uploaded binary.
- **Expo config**: Keep `mobile/app.config.js` → `expo.version` in sync for consistency across OTA
  updates, release notes, and iOS builds. Optionally set `android.versionCode` there if you rely on
  EAS Build’s managed workflow; when you run `npx expo prebuild`, Expo will copy those values into
  `android/app/build.gradle`.
- **Practical workflow**: If you edit only `app.config.js`, run `npx expo prebuild` (or let EAS
  Build handle it) so the Gradle file regenerates with the new numbers. Otherwise, simply edit both
  files manually—Gradle is the source Play inspects.

### Permissions & Other Native Config

- **Where**: Expo config plugins or manual edits under `mobile/android/` (e.g.
  `AndroidManifest.xml`).
- **Rebuild**: Any permission change requires a new binary. Some permissions also demand Play
  Console declarations (Data safety, sensitive permissions forms, etc.).

### Environment & API Settings

- **Where**: `extra.EXPO_PUBLIC_API_BASE_URL` in `mobile/app.config.js`, optionally overridden via
  `.env` variables during the build.
- **Effect**: Changes which backend the app talks to. Because the value is bundled at build time,
  generating a new AAB is required.

### Play Console–Only Branding (No Binary Change Required)

- Store name & descriptions: Update under **Grow → Store presence → Main store listing**.
- Marketing media: Upload new 512×512 store icon, 1024×500 feature graphic, and screenshots under
  **Store listing graphics**.
- Metadata: Category, tags, contact info, developer name, privacy policy URL, Data safety responses,
  and ads declarations all live in the Play Console.
- Timing: You can update these anytime. They may trigger a Play review but do **not** require a new
  build.

### Not Changeable Post-Release

Once an AAB has been uploaded to any Play track, treat the following as permanent unless you plan to
publish a separate listing:

- `android.package` (Android package name / applicationId).
- `ios.bundleIdentifier` (iOS bundle ID).

Everything else can be refreshed with a new build and incremented version code.

## Release Checklist

1. Update branding assets and configuration in `mobile/app.config.js`.
2. Bump `defaultConfig.versionCode` and `defaultConfig.versionName` in
   `mobile/android/app/build.gradle` (and mirror the version in `app.config.js`).
3. Verify icons and splash assets meet required dimensions.
4. Run `npx expo prebuild` if you changed config values that should flow into native code, then
   build a fresh AAB (e.g. `./gradlew bundleRelease` or `eas build --platform android`).
5. Upload the AAB to the Play Console and refresh store listing copy/graphics as needed.
6. Review Play Console compliance sections (privacy policy, Data safety, ads) for accuracy before
   submitting.

Keep this document handy whenever the app needs rebranding to ensure every touchpoint is updated in
the right place.
