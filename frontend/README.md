# Frontend workspace overview

## What lives here

This workspace contains the customer-facing React single-page application that powers AhmedMonib E-Shop. It is built with Vite for rapid iteration, Tailwind CSS + DaisyUI for styling, and a thin theming bridge in [`frontend/theme/`](./theme/) that syncs with the shared DaisyUI catalogue defined at the monorepo level. Runtime data flows through the shared [`@eshop/api-client`](../packages/api-client) package, which wraps the Express backend API (`backend/`) and ships typed helpers, Axios configuration, and authenticated request utilities.
This template provides a minimal setup to get React working in Vite with HMR and some ESLint rules.

The storefront consumes the same localization bundle as the mobile app via [`@eshop/locales`](../shared), keeps observability parity with [`@sentry/react`](https://docs.sentry.io/platforms/javascript/guides/react/), and integrates Stripe Checkout using `@stripe/stripe-js`. Major UI primitives come from Tailwind/DaisyUI, Headless UI, and Material UI (icons only), layered with local components under `src/components/`.

For more on the end-to-end theme architecture and release procedures, see the root docs:

- [DaisyUI theme source of truth](../docs/DAISYUI-THEME.md)
- [Web/mobile theme bridge design](../docs/THEME.md)
- [Deployment notes and release workflow](../docs/deployment.md)

## Backend integration

Network calls are centralised in [`src/lib/axios.js`](./src/lib/axios.js), which reads `VITE_API_BASE_URL` and applies long-polling timeouts defined in the environment. All feature-specific stores (e.g. `src/stores/useProductStore.js`) import the shared API client to interact with the Express backend. Authentication state is coordinated with HTTP-only refresh tokens issued by the backend; the frontend refreshes access tokens via the `/auth/refresh` endpoint and keeps sessions alive according to `VITE_KEEPALIVE_INTERVAL_MS`. Stripe Checkout, webhooks, and order fulfilment flows are aligned with the contracts documented in the [root README](../README.md#payment--order-flow-detailed).

## Local development

Install dependencies from the monorepo root so the workspace symlinks resolve correctly:

```bash
npm install
npm -w frontend run dev
```

Common scripts exposed via `frontend/package.json`:

| Command | Purpose |
| --- | --- |
| `npm -w frontend run dev` | Start the Vite dev server with HTTPS (relies on `@vitejs/plugin-basic-ssl`). |
| `npm -w frontend run build` | Generate a production build for Vercel deployment. |
| `npm -w frontend run preview` | Serve the latest build locally for smoke testing. |
| `npm -w frontend run lint` | Run ESLint across the workspace. |
| `npm -w frontend test` | Execute the Jest unit test suite. |
| `npm -w frontend run test:coverage` | Run Jest with coverage reporting enabled. |
| `npm -w frontend run cypress:open` | Launch the Cypress runner against the dev server. |
| `npm -w frontend run cypress:run` | Execute the Electron-based Cypress suite in headless mode. |
| `npm -w frontend run test:e2e` | Convenience alias for the headless Cypress run. |

These scripts are also orchestrated by the root helpers (`npm run dev`, `npm run dev:all`) listed in the [monorepo README](../README.md#common-scripts).

## Environment configuration

Create a `.env` file at `frontend/.env` (or use `.env.local`) and define the Vite-prefixed variables consumed by the app. Vite exposes variables at build time if they start with `VITE_`. Key entries:

| Variable | Description |
| --- | --- |
| `VITE_API_BASE_URL` | Base URL for the Express API (defaults to `https://localhost:5001` in `src/lib/axios.js`). |
| `VITE_API_TIMEOUT_MS` | Overrides the long Axios timeout used for catalogue calls. |
| `VITE_STRIPE_PUBLISHABLE_KEY` | Publishable key used by `@stripe/stripe-js` for checkout sessions. |
| `VITE_TAX_RATE` / `VITE_SHIPPING_COST` | Optional overrides for checkout calculations. |
| `VITE_KEEPALIVE_INTERVAL_MS` | Interval for the session keep-alive heartbeat. |
| `VITE_SENTRY_DSN`, `VITE_SENTRY_RELEASE`, `VITE_SENTRY_TRACES_SAMPLE_RATE` | Configure Sentry error + performance monitoring. |
| `VITE_GA_ID` | Optional Google Analytics measurement ID. |

Refer to the root [environment variable matrix](../README.md#environment-variables-used) for backend and mobile counterparts.

## Directory tour

The `src/` directory is structured around cohesive feature areas and reusable primitives:

| Path | Notes |
| --- | --- |
| `src/main.jsx` | Entry point that bootstraps React Router, i18n, Sentry, and the API base URL. |
| `src/components/` | Shared UI components (product cards, cart widgets, modals, marketing blocks). |
| `src/pages/` | Route-level containers for core storefront views (home, product detail, checkout, account). |
| `src/stores/` | Zustand stores orchestrating API calls, caching, and optimistic updates. |
| `src/contexts/` | React context providers for theme selection, cart, and auth state. |
| `src/hooks/` | Custom hooks for responsive behavior, form logic, and integrations. |
| `src/lib/` | Infrastructure helpers (Axios instance, logger, feature flags). |
| `src/data/` | Static fixtures and configuration objects used across components. |
| `src/utils/` | Pure utility functions (formatters, validators). |
| `src/tests/` | Jest test utilities and MSW handlers for component isolation. |
| `src/i18n.js` | Locale bootstrapper shared with `@eshop/locales`. |
| `theme/` | DaisyUI-to-native bridge used by Tailwind (pairs with docs linked above). |

For backend API behavior, release coordination, and infrastructure-specific instructions, defer to the [root README](../README.md) and supporting documents in `docs/`.
