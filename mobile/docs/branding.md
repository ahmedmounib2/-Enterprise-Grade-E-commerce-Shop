# Mobile App Branding Guide

This guide documents the branding-related settings for the Expo-based Android app in `mobile/` and
explains where each value lives, what it changes in the product, and whether you must ship a new
Android App Bundle (AAB) after editing it. Use it as a quick reference before every rebrand or
release.

## Summary Table

| Thing                                                 | Where you change it                                                                       | What changes in the product                                     | Rebuild needed?                                                       |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------- |
| Launcher label (name under the icon)                  | `mobile/app.config.js` → `expo.name`                                                      | Name under the launcher icon, recent apps, notification headers | ✅ Yes – create a new AAB and bump the version code                   |
| Bundle identifier / package name                      | `mobile/app.config.js` → `ios.bundleIdentifier`, `android.package`                        | Store identity, app links, update channel                       | ✅ Only before the first upload. After publishing, treat as permanent |
| App icon (standard + adaptive)                        | `mobile/app.config.js` → `expo.icon`, `android.adaptiveIcon.*` + PNGs in `mobile/assets/` | Icon shown on the home screen and app drawer                    | ✅ Yes – rebuild with the new artwork                                 |
| Splash screen artwork                                 | `mobile/app.config.js` → `splash.*` + referenced assets                                   | Launch screen shown while the JS bundle loads                   | ✅ Yes – rebuild                                                      |
| Deep-link scheme / intent filters                     | `mobile/app.config.js` → `scheme`, `android.intentFilters`                                | Which custom URLs open the app                                  | ✅ Yes – rebuild                                                      |
| Version name (Android)                                | `mobile/app.config.js` → `expo.version`                                                   | Human-readable version in Android settings & Google Play        | ✅ Yes – rebuild                                                      |
| Version code (Android)                                | `mobile/app.config.js` → `android.versionCode`                                            | Internal build number Google Play enforces                      | ✅ Yes – must increase with every AAB                                 |
| Runtime permissions                                   | Config plugins in `mobile/plugins/` or `app.config.js` (e.g. `android.permissions`)       | Which Android permissions the binary requests                   | ✅ Yes – and Play may require additional declarations                 |
| API base URL and env flags                            | `mobile/app.config.js` → `extra.EXPO_PUBLIC_API_BASE_URL` (or `.env`)                     | Backend endpoint bundled into the app                           | ✅ Yes – rebuild so the new value is embedded                         |
| Play Store title                                      | Google Play Console → **Grow → Store presence → Main store listing**                      | Name shown in Play Store search/results                         | ❌ No – update listing only                                           |
| Store descriptions (short/full)                       | Play Console → Main store listing                                                         | Marketing copy in Play Store                                    | ❌ No                                                                 |
| Store graphics (icon, feature graphic, screenshots)   | Play Console → Store listing graphics                                                     | Marketing media on the Play Store page                          | ❌ No                                                                 |
| Category, tags, contact info, developer name          | Play Console → Store settings                                                             | Metadata shown to customers                                     | ❌ No                                                                 |
| Privacy policy URL, Data safety form, ads declaration | Play Console → App content                                                                | Compliance disclosures visible in Play                          | ❌ No (updates can trigger review)                                    |

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

- **Config**: the per-variant `scheme` (set from the `VARIANTS` map, keyed by `APP_VARIANT`) and
  `android.intentFilters` in `mobile/app.config.js`.
- **Effect**: Determines which custom-scheme URLs (`vexflare://`, `vexflareinternal://`,
  `vexflaredev://`) launch the app. A rebuild is needed for Android to pick up the new manifest
  entries.

### Versioning

- **What must change each release?** Increment both `version` (name) and `android.versionCode` in
  `mobile/app.config.js` before building a new AAB for Google Play.
- **Authoritative source**: `mobile/android/` is git-ignored and fully regenerated by
  `expo prebuild` (locally) or EAS Build (in the cloud) from `mobile/app.config.js` on every build —
  there is no committed `build.gradle` to edit or keep in sync. `mobile/app.config.js` is the single
  source of truth for both `version` and `android.versionCode`; the generated `build.gradle` merely
  mirrors them. `eas.json` sets `appVersionSource: "local"` and `autoIncrement: false`, so nothing
  bumps these for you.
- **Practical workflow**: Edit `version` / `android.versionCode` in `app.config.js`, then build (EAS
  regenerates `android/` automatically; a local build needs
  `npx expo prebuild --platform android --clean` first). Never hand-edit a value inside
  `mobile/android/` — it is discarded on the next prebuild.

### Permissions & Other Native Config

- **Where**: Expo config plugins under `mobile/plugins/`, or declarative keys in
  `mobile/app.config.js` (e.g. `android.permissions`). `mobile/android/` is regenerated on every
  build, so a manual edit to `AndroidManifest.xml` there is discarded on the next prebuild — it must
  go through a config plugin or an `app.config.js` key to persist.
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
2. Bump `version` and `android.versionCode` in `mobile/app.config.js` — the single source of truth.
3. Verify icons and splash assets meet required dimensions.
4. Build a fresh AAB (`eas build --platform android --profile production`, or locally:
   `npx expo prebuild --platform android --clean` then `./gradlew bundleRelease`).
5. Upload the AAB to the Play Console and refresh store listing copy/graphics as needed.
6. Review Play Console compliance sections (privacy policy, Data safety, ads) for accuracy before
   submitting.

Keep this document handy whenever the app needs rebranding to ensure every touchpoint is updated in
the right place.
