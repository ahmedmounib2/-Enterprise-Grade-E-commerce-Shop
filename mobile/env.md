# Mobile Environments, Monorepo & Deployment Reference

````md
## Table of contents

- [Quick map of the stack](#quick-map-of-the-stack)
- [Workspaces & npm conventions](#workspaces--npm-conventions)
- [Deploy surfaces & responsibilities](#deploy-surfaces--responsibilities)
  - [Frontend on Vercel](#frontend-on-vercel)
  - [Backend on Railway](#backend-on-railway)
  - [Mobile workspace](#mobile-workspace)
- [Environment variables & secrets](#environment-variables--secrets)
  - [Frontend (Vercel)](#frontend-vercel)
  - [Backend (Railway)](#backend-railway)
  - [Shared contracts](#shared-contracts)
  - [Mobile `.env` profiles](#mobile-env-profiles)
- [Command library](#command-library)
  - [Workspace management](#workspace-management)
  - [Deployment CLI](#deployment-cli)
  - [Deep-link validation](#deep-link-validation)
  - [Device & emulator utilities](#device--emulator-utilities)
- [Mobile run modes](#mobile-run-modes)
  - [Mode A — Android emulator (localhost)](#mode-a--android-emulator-localhost)
  - [Mode B — Tunnel Metro + LAN API](#mode-b--tunnel-metro--lan-api)
  - [Mode C — Tunnel Metro + ngrok HTTPS](#mode-c--tunnel-metro--ngrok-https)
  - [Mode picker cheat sheet](#mode-picker-cheat-sheet)
- [Operational checklists](#operational-checklists)
  - [Daily dev hygiene](#daily-dev-hygiene)
  - [Release checklist](#release-checklist)
- [Troubleshooting & FAQs](#troubleshooting--faqs)
- [Reference links](#reference-links)

---

## Quick map of the stack

| Surface | Location    | Tech                | Deploy target             | Highlights                                                                                                                                        |
| ------- | ----------- | ------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Web app | `frontend/` | Vite + React        | **Vercel**                | SPA rewrites defined in `frontend/vercel.json`; `/api/*` rewrites to Railway backend; production install forced with `npm ci --legacy-peer-deps`. |
| API     | `backend/`  | Node + Express      | **Railway**               | Emits reset emails with both deep link (`eshop://...`) and web fallback URLs.                                                                     |
| Mobile  | `mobile/`   | Expo + React Native | Local (custom dev client) | Handles custom scheme `eshop://` for password reset & OAuth flows.                                                                                |

**Deep-link flow recap**

1. Backend generates reset emails with `deepLink=eshop://reset-password?token=...&email=...` and
   `webFallback=https://<vercel-app>/reset-password`.
2. Android intent filters (see `app.config.js`) accept `eshop://` and forward to Expo.
3. Navigation prefixes: `['eshop://', 'exp+eshop-mobile://']` and config
   `{ ResetPassword: 'reset-password' }` ensure both logged-in and anonymous users land on the
   `ResetPassword` screen.

---

## Workspaces & npm conventions

This repository uses **npm workspaces** via the root `package.json`.

```bash
npm install         # installs all workspace dependencies respecting package-lock.json
npm run dev         # concurrently starts backend (https://localhost:5001) and frontend (https://localhost:5173)

# Run scripts for a specific workspace (-w <name>)
npm -w frontend run build
npm -w backend run dev
npm -w mobile run start:tunnel
```
````

Handy tips:

- Each workspace has its own `package.json`; use `npm -w <workspace> run <script>` to scope
  commands.
- Husky pre-commit hook is non-blocking (`lint-staged || true`); fix lint at your pace or run
  `npm -w <workspace> run lint` manually.
- Node 20.x is the baseline for parity with Vercel builds.

---

## Deploy surfaces & responsibilities

### Frontend on Vercel

- Project settings → **Root Directory = `frontend`**.
- `frontend/vercel.json` configures:
  - `installCommand`: `npm ci --legacy-peer-deps` (avoids ESLint peer dependency conflicts).
  - Rewrites: `/api/*` → Railway API base URL.
  - SPA fallback: rewrites everything else to `/index.html`.
  - Cache headers as needed.

- Continuous deployment from GitHub or manual deploys with the CLI.

### Backend on Railway

- Uses the `backend/` workspace.
- Configure service environment variables (see [Backend (Railway)](#backend-railway)).
- Redeploy after modifying auth flows, email templates, or env values.
- Keep the service attached to the GitHub repo for push-to-deploy, or trigger manual redeploys from
  the dashboard.

### Mobile workspace

- Expo project defined in `mobile/` with custom dev client.
- `app.config.js` declares `scheme: 'eshop'` and Android intent filters.
- Development build installed on physical device(s) via Gradle (`./gradlew installDebug`).
- `.env` management handled through scripts (see [Mobile `.env` profiles](#mobile-env-profiles)).

---

## Environment variables & secrets

### Frontend (Vercel)

| Variable                     | Description                                                                                | Where                                             |
| ---------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `PUBLIC_CLIENT_FALLBACK_URL` | HTTPS URL for the web reset page (`https://www.ahmedmonib-eshop-demo.com/reset-password`). | Vercel Project → Settings → Environment Variables |

Additional Vercel config lives in `frontend/vercel.json` (install/build commands, rewrites,
headers).

### Backend (Railway)

| Variable                     | Description                                                    | Typical value                                          |
| ---------------------------- | -------------------------------------------------------------- | ------------------------------------------------------ |
| `PUBLIC_CLIENT_FALLBACK_URL` | Matches the Vercel fallback so emails share the same SPA link. | `https://www.ahmedmonib-eshop-demo.com/reset-password` |
| `MOBILE_RESET_REDIRECT_URI`  | Custom-scheme deep link for password resets.                   | `eshop://reset-password`                               |
| `MOBILE_MAIL_CONFIRM_URI`    | Custom-scheme deep link for mailing confirmations.             | `eshop://mailing/confirm`                              |
| `MAIL_CONFIRM_WEB_URL`       | Optional HTTPS override used inside confirmation emails.       | `https://shop.example.com/mailing/confirm`             |
| `MOBILE_OAUTH_REDIRECT_URI`  | Custom-scheme deep link for OAuth return.                      | `eshop://oauth`                                        |

### Shared contracts

- Keep Vercel and Railway `PUBLIC_CLIENT_FALLBACK_URL` synchronized.
- Whenever you change any deep-link scheme or path, update:
  - Backend env vars above.
  - `mobile/app.config.js` intent filters & linking prefixes.
  - Email templates that generate reset links.

### Mobile `.env` profiles

The Expo app reads `EXPO_PUBLIC_API_BASE_URL` from `.env` in `mobile/`. Three presets are provided:

| Profile    | File          | `EXPO_PUBLIC_API_BASE_URL`                | Use for                                                             |
| ---------- | ------------- | ----------------------------------------- | ------------------------------------------------------------------- |
| Emulator   | `.env.emu`    | `http://10.0.2.2:5001/api`                | Android emulator hitting backend on host machine via special alias. |
| LAN device | `.env.lan`    | `http://<local-ip>:5001/api`              | Physical device on same Wi-Fi calling local backend.                |
| Tunnel     | `.env.tunnel` | `https://<your-ngrok>.ngrok-free.app/api` | Physical device over HTTPS via ngrok tunnel (default for sharing).  |

Switch between them with the helper scripts (run from repo root):

```bash
npm -w mobile run env:emu
npm -w mobile run env:lan
npm -w mobile run env:tunnel
```

> Whenever you edit a profile, restart Metro with `-c` (`npx expo start --tunnel -c`) to bust the
> cache.

### Transport security & pinning

- `EXPO_PUBLIC_SSL_PINNING_CERTS` — comma-separated list of certificate basenames (without file
  extension) stored under `mobile/certs/`. Example: `EXPO_PUBLIC_SSL_PINNING_CERTS=eshop_api`. The
  certificates must match the public keys served by your production API host. Pinning defends
  against malicious Wi-Fi access points and compromised certificate authorities by rejecting TLS
  handshakes unless the presented public key matches the bundled cert.
- **Why dev builds usually skip pinning** – day-to-day development relies on localhost servers, LAN
  IPs (e.g., `http://192.168.1.50:5001`), or temporary HTTPS tunnels (ngrok, Cloudflare tunnels).
  Those endpoints rotate certificates frequently or present self-signed certs, so the app
  automatically disables pin enforcement whenever the API URL resolves to `localhost`, a raw IPv4
  address, or a known tunnelling domain suffix (e.g., `*.ngrok-free.app`). Leave
  `EXPO_PUBLIC_SSL_PINNING_CERTS` empty for emulator-only work; populate it only when you need to
  exercise production-grade networking against the Railway API or another stable host.
- **How to test the pin** – run a dev client with the variable set and point
  `EXPO_PUBLIC_API_BASE_URL` at the production hostname. Any MITM attempt (Charles proxy, custom
  VPN) should trigger a network error with a "pin validation failed" message in the JS console. For
  debugging, you can temporarily comment the pin list, rebuild the dev client, and restore it once
  testing is complete.

---

## Command library

### Workspace management

```bash
npm install                  # bootstrap all workspaces
npm run dev                  # backend + frontend concurrently
npm -w frontend run build    # Vercel-ready production build
npm -w backend run dev       # backend only (nodemon)
npm -w mobile run start:tunnel  # Expo tunnel with cache clear
npm -w mobile run env:<profile> # copy .env.<profile> → .env
```

### Deployment CLI

```bash
# Vercel
npx vercel link              # link current directory to the Vercel project
npx vercel --prod --yes      # trigger production deployment

# Railway (if GitHub integration not enabled)
railway up                   # optional CLI; otherwise redeploy via dashboard
git push railway main        # if Railway is wired to the repo remote
```

### Deep-link validation

```bash
npx uri-scheme open "eshop://reset-password?token=TEST123&email=user%40example.com" --android
adb shell dumpsys package com.ahmedmonib.eshop | grep -i "scheme"
```

Look for Metro logs prefixed with `🔗` to confirm `extractResetParams` fired and navigation reached
the `ResetPassword` screen.

### Device & emulator utilities

```bash
adb devices -l                  # list connected devices
adb reverse tcp:8081 tcp:8081   # expose Metro to emulator (mode A)
adb reverse tcp:19000 tcp:19000 # expose dev menu to emulator (mode A)
adb pair <ip>:<pair-port> && adb connect <ip>:<debug-port>  # Wi-Fi debugging
adb install -r mobile/android/app/build/outputs/apk/debug/app-debug.apk
```

---

## Mobile run modes

Each mode assumes three terminals: **T1** (backend/frontend), **T2** (environment prep), **T3**
(Metro). Commands run from the repo root unless stated.

### Mode A — Android emulator (localhost)

**T1 – backend/frontend**

```bash
npm run dev
```

**T2 – prepare emulator**

```bash
npm -w mobile run env:emu
adb reverse tcp:8081 tcp:8081
adb reverse tcp:19000 tcp:19000
```

**T3 – Metro**

```bash
cd mobile
npx expo start --localhost -c
# press "a" to open the running emulator
```

Sanity checks:

- Expo shows `exp://127.0.0.1:8081`.
- `adb reverse --list` includes ports 8081 & 19000.
- `console.log(process.env.EXPO_PUBLIC_API_BASE_URL)` → `http://10.0.2.2:5001/api`.

### Mode B — Tunnel Metro + LAN API

**T1**

```bash
npm run dev
```

**T2**

```bash
npm -w mobile run env:lan  # ensure file has your current IPv4
```

**T3**

```bash
cd mobile
npx expo start --tunnel -c
# scan QR with dev client or Expo Go
```

Sanity checks:

- Tunnel URL `exp://...exp.direct` or `u.expo.dev/...`.
- Phone browser loads `http://<local-ip>:5001/api/products`.
- Expo logs show LAN API base URL.

### Mode C — Tunnel Metro + ngrok HTTPS

**T1**

```bash
npm run dev
```

**T2**

```bash
pgrep -af ngrok  # Kill any lingering ngrok agent locally

kill <PID>                 # graceful
# if it won’t die:
kill -9 <PID>
# or nuke all ngrok processes:
pkill -f ngrok

npx ngrok http https://localhost:5001
npm -w mobile run env:tunnel  # paste the ngrok HTTPS URL into .env.tunnel
```

**T3**

```bash
cd mobile
npx expo start --tunnel -c
```

Sanity checks:

- Phone browser hits `https://<subdomain>.ngrok-free.app/api/products`.
- Expo logs show HTTPS API base.
- Remember: free ngrok URLs rotate; update `.env.tunnel` each session.

### Mode picker cheat sheet

| Mode                    | Use when…                                     | Pros                                 | Cons                                                            | Core commands                                                                                      |
| ----------------------- | --------------------------------------------- | ------------------------------------ | --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **A. Emulator**         | Fast iteration on one machine                 | No Wi-Fi dependencies; quick reloads | Must keep emulator running; re-run `adb reverse` after restarts | `npm run dev` → `npm -w mobile run env:emu` → `adb reverse …` → `npx expo start --localhost -c`    |
| **B. Tunnel + LAN API** | Real device on same Wi-Fi                     | Stable Metro tunnel, fast LAN API    | Requires firewall rule & up-to-date IPv4                        | `npm run dev` → `npm -w mobile run env:lan` → `npx expo start --tunnel -c`                         |
| **C. Tunnel + ngrok**   | HTTPS required or sharing with remote testers | Works anywhere, trusted cert         | Keep ngrok session alive; URLs change                           | `npm run dev` → `npx ngrok http …` → `npm -w mobile run env:tunnel` → `npx expo start --tunnel -c` |

---

## Operational checklists

### Daily dev hygiene

- ✅ Start backend/frontend: `npm run dev`.
- ✅ Choose environment profile (`env:emu`, `env:lan`, or `env:tunnel`).
- ✅ Launch Metro with `--localhost` (emulator) or `--tunnel` (physical devices) and include `-c`
  after env changes.
- ✅ Test deep link: `npx uri-scheme open "eshop://reset-password?token=TEST..." --android`.
- ✅ Confirm Expo JS console shows expected `EXPO_PUBLIC_API_BASE_URL`.

### Release checklist

- [ ] Verify Vercel deploy succeeded (`npx vercel --prod --yes` if pushing manually).
- [ ] Ensure Railway env vars match the latest deep-link scheme/fallback URLs.
- [ ] Send yourself a reset email and test on both web and mobile.
- [ ] Update any docs if env defaults or URLs changed.
- [ ] Tag the commit and share tunnel/API URLs with the team if remote QA is needed.

---

## Troubleshooting & FAQs

- **Vercel build fails with peer dependency errors** → Confirm `frontend/vercel.json` still forces
  `npm ci --legacy-peer-deps`.
- **Expo keeps opening Expo Go** → When scanning the QR, choose the **Development Build** option so
  the custom client launches.
- **`adb devices` shows `unauthorized`** → Unlock phone, accept RSA fingerprint, then re-run
  `adb devices -l`.
- **Emulator loses connection after restart** → Re-run the two `adb reverse` commands for ports 8081
  & 19000.
- **Phone cannot hit API** → Open the API URL in mobile browser; adjust firewall (TCP 5001) or
  update `.env.lan` / `.env.tunnel`.
- **Deep link does nothing** → Use `adb logcat` filtered by `Linking` or check Metro logs for the
  `🔗` messages; ensure `extractResetParams` still matches the scheme.
- **ngrok shuts down** → Keep the terminal open or upgrade plan; update `.env.tunnel` when the
  domain changes.

---

## Reference links

- [Custom dev client setup](./dev-setup.md)
- [Expo Linking guide](https://docs.expo.dev/guides/linking/)
- [Vercel CLI docs](https://vercel.com/docs/cli)
- [Railway docs](https://docs.railway.app/)
- [Android ADB docs](https://developer.android.com/studio/command-line/adb)

---
