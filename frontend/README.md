# Frontend workspace overview

## What lives here

This workspace contains the customer-facing React single-page application that powers Vexflare. It
is built with Vite for rapid iteration, Tailwind CSS + DaisyUI for styling, and a thin theming
bridge in [`frontend/theme/`](./theme/) that syncs with the shared DaisyUI catalogue defined at the
monorepo level. Runtime data flows through the shared [`@eshop/api-client`](../packages/api-client)
package, which wraps the Express backend API (`backend/`) and ships typed helpers, Axios
configuration, and authenticated request utilities. This template provides a minimal setup to get
React working in Vite with HMR and some ESLint rules.

The storefront consumes the same localization bundle as the mobile app via
[`@eshop/locales`](../shared), keeps observability parity with
[`@sentry/react`](https://docs.sentry.io/platforms/javascript/guides/react/), and integrates Stripe
Checkout using `@stripe/stripe-js`. Major UI primitives come from Tailwind/DaisyUI, Headless UI, and
Material UI (icons only), layered with local components under `src/components/`.

For more on the end-to-end theme architecture and release procedures, see the root docs:

- [DaisyUI theme source of truth](../docs/DAISYUI-THEME.md)
- [Web/mobile theme bridge design](../docs/THEME.md)
- [Deployment notes and release workflow](../docs/deployment.md)

## Backend integration

Network calls are centralised in [`src/lib/axios.js`](./src/lib/axios.js), which reads
`VITE_API_BASE_URL` and applies long-polling timeouts defined in the environment. All
feature-specific stores (e.g. `src/stores/useProductStore.js`) import the shared API client to
interact with the Express backend. Authentication state is coordinated with HTTP-only refresh tokens
issued by the backend; the frontend refreshes access tokens via the `/auth/refresh` endpoint and
keeps sessions alive according to `VITE_KEEPALIVE_INTERVAL_MS`. Stripe Checkout, webhooks, and order
fulfilment flows are aligned with the contracts documented in the
[root README](../README.md#payment--order-flow-detailed).

## Local development

Install dependencies from the monorepo root so the workspace symlinks resolve correctly:

```bash
npm install
npm -w frontend run dev
```

Common scripts exposed via `frontend/package.json`:

| Command                             | Purpose                                                                      |
| ----------------------------------- | ---------------------------------------------------------------------------- |
| `npm -w frontend run dev`           | Start the Vite dev server with HTTPS (relies on `@vitejs/plugin-basic-ssl`). |
| `npm -w frontend run build`         | Generate a production build for Vercel deployment.                           |
| `npm -w frontend run preview`       | Serve the latest build locally for smoke testing.                            |
| `npm -w frontend run lint`          | Run ESLint across the workspace.                                             |
| `npm -w frontend test`              | Execute the Jest unit test suite.                                            |
| `npm -w frontend run test:coverage` | Run Jest with coverage reporting enabled.                                    |
| `npm -w frontend run cypress:open`  | Launch the Cypress runner against the dev server.                            |
| `npm -w frontend run cypress:run`   | Execute the Electron-based Cypress suite in headless mode.                   |
| `npm -w frontend run test:e2e`      | Convenience alias for the headless Cypress run.                              |

These scripts are also orchestrated by the root helpers (`npm run dev`, `npm run dev:all`) listed in
the [monorepo README](../README.md#common-scripts).

## Environment configuration

Create a `.env` file at `frontend/.env` (or use `.env.local`) and define the Vite-prefixed variables
consumed by the app. Vite exposes variables at build time if they start with `VITE_`. Key entries:

| Variable                                                                   | Description                                                                                                  |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `VITE_API_BASE_URL`                                                        | Base URL for the Express API (defaults to `https://localhost:5001` in `src/lib/axios.js`).                   |
| `VITE_API_TIMEOUT_MS`                                                      | Overrides the long Axios timeout used for catalogue calls.                                                   |
| `VITE_STRIPE_PUBLISHABLE_KEY`                                              | Publishable key used by `@stripe/stripe-js` for checkout sessions.                                           |
| `VITE_TAX_RATE` / `VITE_SHIPPING_COST`                                     | Optional overrides for checkout calculations.                                                                |
| `VITE_KEEPALIVE_INTERVAL_MS`                                               | Interval for the session keep-alive heartbeat.                                                               |
| `VITE_SENTRY_DSN`, `VITE_SENTRY_RELEASE`, `VITE_SENTRY_TRACES_SAMPLE_RATE` | Configure Sentry error + performance monitoring.                                                             |
| `VITE_GA_ID`                                                               | GA4 Measurement ID (e.g. `G-XXXXXXXXXX`). Embedded at build time by Vite. Required for production analytics. |

Refer to the root [environment variable matrix](../README.md#environment-variables-used) for backend
and mobile counterparts.

## Google Analytics

Google Analytics 4 is gated behind cookie consent and a build-time environment variable.

**How it works:**

- `analytics.js` runs at module load time — before any React rendering — and immediately declares
  all GA4 storage as denied via
  `gtag('consent', 'default', { analytics_storage: 'denied', ..., wait_for_update: 500 })`. This
  satisfies GA4 Consent Mode v2: the consent state is set before the `gtag.js` script loads so GA4
  never fires hits in an undefined state.
- On page load, `App.jsx` reads the `site_cookie_consent` localStorage key and calls
  `initAnalytics()` for returning visitors who already granted consent.
- If the user accepts cookies, `initAnalytics()` (from `src/lib/analytics.js`) updates consent to
  `analytics_storage: granted`, injects the `gtag.js` script, and calls `gtag('config', GA_ID)`. The
  consent decision is also logged to the backend via `POST /api/users/consent-log` (no auth
  required, rate-limited 10 req/min per IP). Anonymous visitors are assigned a client UUID stored in
  localStorage as `eshop_anon_id`. On login or signup, `POST /api/users/me/merge-consent` merges
  anonymous consent records into the authenticated user account.
- Every React Router navigation fires `trackPageView(pathname)` via a `useEffect` on
  `location.pathname`.
- If the user declines or withdraws consent, `removeAnalytics()` updates consent to denied, removes
  the `gtag.js` script, and clears GA cookies. All `track*` helpers become no-ops until consent is
  granted again.

**Local development:**

Add `VITE_GA_ID=G-XXXXXXXXXX` to `frontend/.env`. Hits will appear in the GA4 Realtime report
immediately after accepting the cookie banner.

**Production (Railway):**

`VITE_GA_ID` is baked into the bundle at build time — it is **not** read at runtime. You must add it
to Railway before deploying:

```
Railway → Project → (frontend service) → Variables → Add Variable
  VITE_GA_ID = G-70S4MSK1VZ
```

A redeploy is required after adding or changing this variable.

**Verifying the integration:**

1. Open GA4 → Reports → Realtime.
2. Visit the production site and accept the cookie banner.
3. You should see at least one active user within 30 seconds.
4. In DevTools → Console, run:

   ```js
   window.gtag("get", "G-70S4MSK1VZ", "client_id", console.log);
   ```

   A printed client ID (not `undefined`) confirms GA is active.

**Custom event helpers** (`src/lib/analytics.js`):

| Function                  | GA4 event name      | When to call                              |
| ------------------------- | ------------------- | ----------------------------------------- |
| `trackPageView(path)`     | `page_view`         | Fired automatically on every route change |
| `trackContactClick()`     | `contact_click`     | Contact CTA button clicks                 |
| `trackServicesView()`     | `services_view`     | Services page mounts                      |
| `trackArchitectureView()` | `architecture_view` | Architecture page mounts                  |
| `trackCaseStudyView()`    | `case_study_view`   | Case Studies page mounts                  |
| `trackDownload(label)`    | `file_download`     | Downloadable asset links                  |
| `trackOutboundLink(url)`  | `click`             | External links (with `outbound: true`)    |

## Directory tour

The `src/` directory is structured around cohesive feature areas and reusable primitives:

| Path              | Notes                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------- |
| `src/main.jsx`    | Entry point that bootstraps React Router, i18n, Sentry, and the API base URL.               |
| `src/components/` | Shared UI components (product cards, cart widgets, modals, marketing blocks).               |
| `src/pages/`      | Route-level containers for core storefront views (home, product detail, checkout, account). |
| `src/stores/`     | Zustand stores orchestrating API calls, caching, and optimistic updates.                    |
| `src/contexts/`   | React context providers for theme selection, cart, and auth state.                          |
| `src/hooks/`      | Custom hooks for responsive behavior, form logic, and integrations.                         |
| `src/lib/`        | Infrastructure helpers (Axios instance, logger, feature flags).                             |
| `src/data/`       | Static fixtures and configuration objects used across components.                           |
| `src/utils/`      | Pure utility functions (formatters, validators).                                            |
| `src/tests/`      | Jest test utilities and MSW handlers for component isolation.                               |
| `src/i18n.js`     | Locale bootstrapper shared with `@eshop/locales`.                                           |
| `theme/`          | DaisyUI-to-native bridge used by Tailwind (pairs with docs linked above).                   |

For backend API behavior, release coordination, and infrastructure-specific instructions, defer to
the [root README](../README.md) and supporting documents in `docs/`.
