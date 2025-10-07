# Mobile Dev Setup (Expo Go Quickstart)

```md
## Table of contents

- [Who this guide is for](#who-this-guide-is-for)
- [Prerequisites](#prerequisites)
- [Clone & install the monorepo](#clone--install-the-monorepo)
- [Run the core services](#run-the-core-services)
- [Launch the mobile app with Expo Go](#launch-the-mobile-app-with-expo-go)
  - [Option A — Android emulator](#option-a--android-emulator)
  - [Option B — Physical Android device](#option-b--physical-android-device)
- [Environment profiles](#environment-profiles)
- [Frequently used npm scripts](#frequently-used-npm-scripts)
- [Troubleshooting](#troubleshooting)
- [When to use the custom dev client instead](#when-to-use-the-custom-dev-client-instead)
```

---

## Who this guide is for

You only need this quickstart if you want the fastest path to running the Expo project with **Expo
Go** on Android. It assumes you are the solo maintainer and prefer lightweight tooling over the
customised development client described in [`mobile/custom-dev-setup.md`](./custom-dev-setup.md).

---

## Prerequisites

- **Node.js 20.x** and **npm 10.x** to match the versions used by CI/hosting.
- **Git** for cloning the repository.
- **Expo CLI** (optional) — the docs below use `npx expo`, so no global install is required.
- Android tooling depends on how you run the app:
  - **Android Studio** if you plan to use an emulator.
  - **Android 10+ device** with Developer Options enabled if you will connect a phone.

> 💡 Using Windows? These instructions assume **WSL2 + Ubuntu 22.04** so you can share the same
> environment as the rest of the monorepo. macOS/Linux users can follow the same commands in a
> regular shell.

---

## Clone & install the monorepo

```bash
# 1. Grab the code
git clone https://github.com/<your-user>/E-commerce-MERN-Stack-website.git
cd E-commerce-MERN-Stack-website

# 2. Install workspace dependencies (frontend, backend, mobile, shared packages)
npm install
```

The repository uses a single `package-lock.json` with **npm workspaces**, so the command above
hydrates every app in one go.

---

## Run the core services

Open two terminals (or use a multiplexer like `tmux`). From the repo root:

```bash
# Terminal 1 — API + web app in dev mode
npm run dev

# Terminal 2 — Metro bundler for the mobile app (no tunnel)
npm -w mobile run start:local
```

The first command starts the Express API on `https://localhost:5001` and the Vite frontend on
`https://localhost:5173`. The second command boots Metro so Expo Go can load the bundle. The
`start:local` script binds Metro to `localhost` which is ideal for emulators. If you need LAN/tunnel
connectivity, switch to `npm -w mobile run start:lan` or `npm -w mobile run start:tunnel` after you
finish the initial pairing.

---

## Launch the mobile app with Expo Go

### Option A — Android emulator

1. Launch **Android Studio → Virtual Device Manager** and start an emulator (API level 33 or newer).
2. With Metro running (`npm -w mobile run start:local`), press `a` in the Metro terminal to install
   the Expo Go app inside the emulator and load the project.
3. Whenever you edit mobile code, Metro will hot-reload the changes. If the bundle gets stuck, press
   `r` in the Metro terminal to restart the bundler.

### Option B — Physical Android device

1. Enable **Developer options → USB debugging** on the phone.
2. Connect the device via USB. On Windows/WSL, run `adb devices` inside WSL to confirm it is
   detected. Accept the RSA prompt on the phone.
3. Start Metro with LAN or tunnel support:

   ```bash
   # LAN mode (phone and PC on same Wi-Fi)
   npm -w mobile run start:lan

   # or tunnel mode (Expo relay, works across networks)
   npm -w mobile run start:tunnel
   ```

4. Scan the QR code shown in the Metro terminal with the Expo Go app and open the project.

> ✅ Expo Go already understands the `eshop://` deep-link scheme declared in `app.config.js`, so you
> can test password-reset links without building the custom dev client.

---

## Environment profiles

The mobile workspace includes `.env` presets for different network setups. Copy one into place
before starting Metro:

```bash
# emulator (localhost API)
npm -w mobile run env:emu

# LAN profile (API hosted on your machine's LAN IP)
npm -w mobile run env:lan

# tunnel profile (API exposed via HTTPS tunnel)
npm -w mobile run env:tunnel
```

These scripts simply copy `mobile/.env.<profile>` to `mobile/.env`. See [`mobile/env.md`](./env.md)
for the full matrix of available variables.

---

## Frequently used npm scripts

```bash
npm -w mobile run lint        # Expo/React Native lint rules
npm -w mobile run test        # Jest tests for mobile utilities (if configured)
npm -w mobile run clean       # Reset Metro, clear caches (defined in package.json)
```

Refer to `package.json` within each workspace for additional scripts like EAS build commands or
type-checking.

---

## Troubleshooting

| Symptom                          | Fix                                                                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Metro shows `EADDRINUSE`         | Stop other Metro instances (`pkill -f "react-native"`) or reboot Metro with `npm -w mobile run start:local -- --reset-cache`.        |
| Device/emulator stuck on splash  | Reload from the developer menu (`r` double-tap) or clear the Expo Go data from Android settings.                                     |
| Deep links open the web fallback | Ensure `MOBILE_RESET_REDIRECT_URI=eshop://reset-password` is set in your backend `.env` and the device installed the latest Expo Go. |
| Cannot reach the API from device | Switch to the appropriate `.env` profile (`env:lan` or `env:tunnel`) and restart Metro.                                              |

---

## When to use the custom dev client instead

Stick with Expo Go for UI work and quick iteration. Move to the custom dev client documented in
[`mobile/custom-dev-setup.md`](./custom-dev-setup.md) when you need to:

- Test native modules that are not bundled with Expo Go.
- Ship release candidates to testers without relying on Expo Go.
- Validate changes to intent filters, splash screens, or other native Android resources.

The custom guide walks through Android SDK installation, Gradle configuration, and maintaining a
signed development build once you are ready for that workflow.
