# AhmedMonib E-Shop — Enterprise-Grade E-commerce Shop

> Production-ready, enterprise-grade e-commerce storefront (React + Vite frontend, Node.js + Express backend) with:
>
> * Secure rotated refresh tokens and sliding 30-day sessions (mobile-aware keep-alive)
> * Product variants, per-variant stock & pricing, cart reservation and automatic stock restore on cancel/expire
> * Stripe Checkout integration with robust webhook handling (PaymentIntent verification, refunds, expired sessions)
> * Order export (PDF) & fulfilment tooling, GDPR data export, Sentry observability, and 89.28% test coverage.

---

## Table of contents

* [AhmedMonib E-Shop — Enterprise-Grade E-commerce Shop](#ahmedmonib-e-shop--enterprise-grade-e-commerce-shop)
  * [Table of contents](#table-of-contents)
  * [One-line pitch](#one-line-pitch)
  * [Live demo \& hosted domains](#live-demo--hosted-domains)
  * [What this repo contains](#what-this-repo-contains)
  * [Key features (summary)](#key-features-summary)
  * [Payment \& order flow (detailed)](#payment--order-flow-detailed)
    * [Checkout (Stripe) flow — `createCheckoutSession`](#checkout-stripe-flow--createcheckoutsession)
    * [Stripe webhooks \& async handling — `stripeWebhook`](#stripe-webhooks--async-handling--stripewebhook)
    * [COD flow (Cash-on-Delivery) — `createCODOrder` \& `codCheckoutSuccess`](#cod-flow-cash-on-delivery--createcodorder--codcheckoutsuccess)
    * [Stock reservation \& restoration (variant-aware)](#stock-reservation--restoration-variant-aware)
    * [Failure \& expired session handling](#failure--expired-session-handling)
  * [Products, variants \& model behavior](#products-variants--model-behavior)
  * [Auth \& session persistence — how it works (plain language)](#auth--session-persistence--how-it-works-plain-language)
  * [Order fulfilment, PDF export \& canceled order labeling](#order-fulfilment-pdf-export--canceled-order-labeling)
  * [GDPR \& user data export](#gdpr--user-data-export)
  * [Security \& observability](#security--observability)
  * [Testing \& CI](#testing--ci)
    * [How to run tests](#how-to-run-tests)
    * [Test status](#test-status)
  * [How to run locally (no Docker)](#how-to-run-locally-no-docker)
  * [Environment variables used](#environment-variables-used)
  * [Redis debugging quick commands](#redis-debugging-quick-commands)
  * [Screenshots](#screenshots)
    * [Orders \& Fulfilment](#orders--fulfilment)
    * [Create Product \& Variants](#create-product--variants)
    * [Analytics \& Campaigns](#analytics--campaigns)
  * [Commercial license (proprietary) \& selling notes](#commercial-license-proprietary--selling-notes)
  * [Contact / commercial enquiries](#contact--commercial-enquiries)

---

## One-line pitch

Enterprise-grade e-commerce storefront with robust session security, per-variant inventory, reservation/restore logic, Stripe payments, order fulfilment tooling, and production observability.

---

## Live demo & hosted domains

* Frontend (production): `https://www.ahmedmonib-eshop-demo.com` (Vercel)
* Frontend (alternate / staging): `https://ecommerce-mern-website-seven.vercel.app`
* API (production): `https://api.ahmedmonib-eshop-demo.com` (Railway)
* Railway preview: `https://e-commerce-mern-stack-website-production.up.railway.app`

---

## What this repo contains

* `frontend/` — React (Vite) app, i18n, Zustand stores, Stripe client, Sentry instrumentation.
* `backend/` — Express API (ESM), Mongoose models, Redis for token storage/caches/locks, Stripe webhook handlers, csurf, passport strategies, email helpers.
* `tests/` — Unit/integration/E2E test suites (Jest, supertest, Cypress) and mocks (including in-memory Mongo for CI).
* CI workflows for builds and Sentry releases (GitHub Actions).
* Full production-ready features: auth, cart, checkout, admin product & order management, order export & printing.

---

## Key features (summary)

* Authentication

  * Short-lived access tokens + rotating refresh tokens stored in Redis (per-device `jti`).
  * Sliding 30-day maximum session window (`session_start:<userId>`).
  * CSRF protection for state-changing endpoints and secure cookie configuration (`SameSite=None`, `Secure`, `HttpOnly`).
  * Mobile-aware keep-alive and activity detection to improve session persistence on phones.
* Payments & Orders

  * Stripe Checkout integration with PaymentIntent verification and robust webhook handling.
  * Cart backup and order fallback stored in Redis during checkout.
  * Stock reservation at checkout, atomic DB transactions for decrements, auto-restore on cancel/expire/refund.
  * Cash-on-delivery (COD) flow with the same reservation guarantees.
* Products & Inventory

  * Variant support (attributes, per-variant price, per-variant stock, variant images).
  * Admin product flows with variant generation and image management.
* Fulfilment & Admin

  * Order listing, status changes, PDF export for fulfilment/labeling, canceled order labeling and audit trails.
* Privacy & Compliance

  * GDPR-ready: user data export endpoint and account deletion/anonymization tooling.
* Observability & Security

  * Sentry client & server integration, Winston logging with daily rotation and PII redaction.
  * Rate limiting, input sanitization, NoSQL injection protection.
* Testing

  * Extensive unit/integration/E2E tests with 89.28% backend coverage.

---

## Payment & order flow (detailed)

This section explains the exact runtime flow implemented by the backend payment controller.

### Checkout (Stripe) flow — `createCheckoutSession`

1. **Request validation**

   * Backend receives `products`, `shippingAddress`, `billingAddress`, and optional `phoneNumbers`.
   * Validates addresses and that the cart is non-empty.

2. **Cart backup & fallback**

   * Saves a `cart_backup:<userId>` key in Redis (TTL) so the server can recover cart if the client drops off.
   * Creates an `order_fallback:<orderId>` Redis key after creating the order — used as a fallback to restore reservations if Stripe session expires.

3. **Order creation (DB transaction)**

   * Loads product metadata (name, price, deal, images, variants) from MongoDB.
   * Computes `subtotal`, `taxAmount` (TAX\_RATE), `shippingCost` (SHIPPING\_COST), and `totalAmount`.
   * Creates an `Order` document that includes per-item `variantAttributes` (stored as `Map` or similar) and persists it inside a Mongo transaction.

4. **Stock reservation**

   * For each order item, calls `reserveStock(productId, quantity, orderId, variantAttributes)`.
   * Reservation logic:

     * Writes a Redis reservation key `stock_reservation:<productId>:<orderId>:<variantKey>` with TTL (e.g., 30 minutes).
     * Starts a MongoDB session + transaction and decrements the product or matched variant stock atomically.
     * Commits transaction only if decrement succeeds; otherwise aborts and errors bubble back to the checkout flow.

5. **Stripe session creation**

   * Builds `line_items` with per-item unit amounts (variant pricing applied if present), plus tax and shipping line items.
   * Creates Stripe Checkout session with `metadata` containing `orderId` and `userId`, `expires_at` equals reservation TTL, and required `success_url` & `cancel_url`.
   * Updates the `Order` with `stripeSessionId` and `paymentIntentId` from Stripe, and stores order fallback info in Redis `order_fallback:<orderId>`.

6. **Commit & response**

   * If all steps succeed, commits the DB transaction and returns the Stripe session ID to the client for redirect.

**Result:** stock is reserved server-side, order exists in DB in `order_placed` (or `cod_order_placed` for COD), and the frontend redirects the user to Stripe Checkout.

---

### Stripe webhooks & async handling — `stripeWebhook`

The webhook handler validates Stripe signatures and supports multiple events:

* **`checkout.session.expired`**

  * Processed asynchronously (background) using `handleExpiredSession(session)`.
  * Actions:

    * Acquire a Redis lock `lock:expire:<orderId>` to avoid race conditions.
    * Restore reserved stock for order by scanning `stock_reservation:*:<orderId>:*` and incrementing product/variant stock (via `restoreStockReservation`).
    * Delete the order (`Order.findByIdAndDelete(orderId)`) and clear safety keys (`cart_backup`, `order_fallback`).
  * This guarantees reserved inventory returns to stock if the checkout is abandoned.

* **`checkout.session.completed`**

  * Reads `session.metadata.orderId` and `userId`.
  * Clears backup safety keys.
  * Optionally fetches Stripe PaymentIntent to determine authoritative payment status (handles 3DS and other async flows).
  * If payment succeeded (`succeeded` / `paid`):

    * Performs optional final checks (extra stock verification).
    * Keeps paid card orders in `order_placed` (paid but awaiting fulfilment) — fulfillment status is separate.
    * Sends confirmation email if not already sent.
  * If a PaymentIntent indicates final failure (`canceled`, `failed`) then:

    * Marks order as `payment_failed`, restores stock, and sets failure reason.

* **`charge.refunded`**

  * If a full refund is detected for a payment intent related to an order:

    * Sets `order.status = 'refunded'`, persists `refundId`, updates timestamps.
    * Restores inventory via `restoreStockReservation(order._id)`.

> All webhook processing logs heavily and attempts safe recovery. Non-critical failures (e.g., email) are logged but don't crash the handler.

---

### COD flow (Cash-on-Delivery) — `createCODOrder` & `codCheckoutSuccess`

* `createCODOrder` follows almost the same path as Stripe checkout: validate payload, compute totals, create an `Order` (status `cod_order_placed`), reserve stock (variant-aware), and return order id and totals to the client.
* `codCheckoutSuccess` accepts `orderId`, verifies user ownership, clears cart and safety keys, sends confirmation email (if not sent), and returns order summary to the client.

---

### Stock reservation & restoration (variant-aware)

* Reservations are stored as Redis keys with structure:

  ```
  stock_reservation:<productId>:<orderId>:<variantKey>
  ```

  where `<variantKey>` is a stable JSON string of variant attributes (so attribute order doesn't break matching).
* `reserveStock`:

  * Saves Redis reservation (TTL \~ 30 min).
  * Opens Mongo transaction and decrements either `product.stock` or `variants.$.stock` for the matched variant.
* `restoreStockReservation`:

  * `KEYS stock_reservation:*:<orderId>:*` (production should use `SCAN` for large keyspaces) to find reservations.
  * Builds `bulkWrite` ops to increment product or variant stock back by reservation quantity, and deletes reservation keys after update.
  * Works with variant attribute matching via `arrayFilters` in Mongo bulk ops.

---

### Failure & expired session handling

* If any step fails before Stripe session creation, the transaction is aborted and the client receives a `400`/`500` error (with friendly messages).
* If Stripe session expires without payment:

  * Webhook triggers `handleExpiredSession`, which restores stock, deletes the incomplete order, and clears safety keys.
  * Because reservations use Redis keys and the DB update is atomic, this avoids double-selling.
* If a PaymentIntent partially or fully fails after session completion:

  * PaymentIntent inspection in the webhook marks `payment_failed` and restores inventory if appropriate.

---

## Products, variants & model behavior

* Products support:

  * Multiple languages for textual fields (e.g., `name.en`, `name.ar`).
  * `images` (array) and per-variant `image` override.
  * `price` and per-variant `price` override.
  * `stock` (global) and per-variant `stock`.
  * `deal` (discount percentage) applied to item price calculation.
* Variants:

  * Stored as objects with `attributes` (Map-like), `price`, `stock`, and optional `image`.
  * Matching helper `findMatchingVariant(product, variantAttributes)` is used widely to:

    * Determine per-item price on checkout and line items.
    * Decrement / increment the exact variant stock on reserve/restore.
* The frontend displays variant-specific price and stock and adds per-variant images when available.

* Product badges:

  * **Deal**, **Best Seller**, **New Release** badges.
  * **Stock badge**: shown when stock ≤ 5.
  * **Discount badge**: supports discount values from **1% to 80%**.
* Ability to **hide** items from the UI (instead of deleting) so you can retain product data and reinstate when restocked.

---

## Auth & session persistence — how it works (plain language)

* Upon login/signup the server issues:

  * `access token` cookie, short-lived (e.g., 15 minutes).
  * `refresh token` cookie, long-lived (sliding, e.g., up to 30 days).
  * Server stores the refresh token value keyed by `refresh_token:<userId>:<jti>` in Redis.
  * A `session_start:<userId>` key is set in Redis to implement a **maximum sliding lifetime** (default 30 days).
* Token rotation:

  * `/api/auth/refresh-token` validates the cookie against Redis and atomically rotates to a new `jti` to prevent reuse of stolen tokens.
* Sliding window:

  * Each successful refresh or keep-alive `EXPIRE`s the `session_start:<userId>` key for another 30 days.
  * If a user is inactive for 30 days (no refresh / keep-alive), `session_start` expires — server rejects further refreshes and requires re-login.
* Mobile friendliness:

  * Frontend implements mobile-aware keep-alive intervals and additional activity listeners (touchstart, focus) to counteract mobile background throttling.
  * The server provides both `/auth/refresh-token` and `/auth/keep-alive` paths — keep-alive refreshes cookie TTLs and can issue a fresh access token.

**UX rule implemented:** sessions persist for up to 30 days of inactivity; any visit/activity refreshes the 30-day window.

---

## Order fulfilment, PDF export & canceled order labeling

* Admin UI supports exporting order details to PDF for fulfilment and label printing. Export includes order lines, shipping address, and order metadata.
* Canceled orders are explicitly labeled in the admin (status `cancelled` / `payment_failed` / `refunded`) and included in exports as such for tracking.
* PDF export utility is implemented server-side (PDF generation libraries) and exposed to the admin for download/printing.

---

## GDPR & user data export

* Users can request their data export via the `exportUserData` endpoint. Export includes orders, profile, and related user metadata in JSON format.
* Account deletion/anonymize flow:

  * When requested, PII is removed/anonymized and orders are either retained as anonymized records or deleted per config/policy.
* The system provides audit logging and an admin review path for data exports to comply with privacy regulations.

---

## Security & observability

* Cookies: `HttpOnly`, `Secure`, `SameSite=None` (when cross-site), and domain configured to allow cross-subdomain cookies for frontend + API.
* CSRF protection enabled for state-changing endpoints (`/api/csrf-token` endpoint available).
* Rate limiting on auth endpoints and refresh endpoint.
* Sanitization and NoSQL injection protection for request bodies and query params.
* Logging: Winston with daily rotation, sensitive data redaction, and optional Sentry forwarding for uncaught exceptions and errors.

---

## Testing & CI

* Tests: Jest unit & integration tests with mocks for Redis / email / cloudinary, supertest for route-level tests, and Cypress for frontend E2E flows.
* Coverage: Backend coverage is reported at **89.28%** (unit + integration).

### How to run tests

* **Unit + integration**:

  ```bash
  # From backend/
  npm run test
  ```

* **E2E (configured Jest e2e runner)**:

  ```bash
  # From backend/
  npm run test:e2e
  ```

* **Run coverage report**:

  ```bash
  npm run test:coverage
  ```

* **Watch tests** (dev):

  ```bash
  npm run test:watch
  ```

* **Debug tests**:

  ```bash
  npm run test:debug
  ```

> The test suite uses `NODE_ENV=test` and in many places relies on an in-memory Mongo test server (MongoMemoryServer) and a mocked Redis in CI. Ensure the test environment can start the in-memory Mongo (no other DB required for tests).

### Test status

* **Coverage:** Backend test coverage: ~89% (unit + integration). Full test suite includes authorization flows, stock reservation, Stripe webhooks, and end-to-end checkout scenarios — automated in CI with deterministic mocks for external services.

---

## How to run locally (no Docker)

> Local setup does **not** require Docker. Use local MongoDB & Redis, or managed services.

1. Clone and install:

```bash
# from repo root
cd backend
npm ci

cd ../frontend
npm ci
```

2. Create `.env` files:

* `backend/.env` — list of env variables (see the list below), **do not** commit secrets to repo.
* `frontend/.env` — `VITE_API_BASE_URL` pointing at your backend (e.g., `http://localhost:5000/api`).

3. Start backend:

```bash
cd backend
npm run dev
```

4. Start frontend:

```bash
cd frontend
npm run dev
```

5. Visit the frontend URL printed by Vite (commonly `http://localhost:5173`).

---

## Environment variables used

**Authentication & cookies**

```
ACCESS_TOKEN_EXPIRE
ACCESS_TOKEN_SECRET
REFRESH_TOKEN_EXPIRE
REFRESH_TOKEN_SECRET
MAX_SESSION_LIFETIME_DAYS
COOKIE_DOMAIN
CSRF_SECRET
COOKIE_SECRET
DEBUG_REFRESH
```

**Application URLs**

```
CLIENT_URL
SERVER_URL
DEV_ORIGIN
PRODUCTION_ORIGIN
VITE_API_BASE_URL (frontend)
VITE_CLIENT_BASE_URL (frontend)
```

**Database & cache**

```
MONGO_URI
UPSTASH_REDIS_URL (or REDIS connection string)
```

**Third-party services**

```
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
SENDGRID_API_KEY
CLOUDINARY_CLOUD_NAME
CLOUDINARY_API_KEY
CLOUDINARY_API_SECRET
SENTRY_DSN
SENTRY_RELEASE
```

**Social logins & callbacks**

```
FACEBOOK_APP_ID
FACEBOOK_APP_SECRET
FACEBOOK_CALLBACK_URL
FACEBOOK_LINK_CALLBACK_URL
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
GOOGLE_CALLBACK_URL
GOOGLE_LINK_CALLBACK_URL
```

**Application settings**

```
PORT
NODE_ENV
VITE_TAX_RATE
VITE_SHIPPING_COST
UPLOADS_BASE_DIR
PRODUCT_IMAGES_DIR
COOKIE_DOMAIN
USE_HTML_REDIRECT
```

Remember to configure these per environment (Railway, Vercel, local).

## Redis debugging quick commands

Use `redis-cli` or your managed Redis console:

```bash
# connect (example)
redis-cli -h 127.0.0.1 -p 6379

# Check TTL for session_start
TTL session_start:<userId>

# Get refresh token stored server-side (debug only)
GET refresh_token:<userId>:<jti>

# Reservation keys
KEYS stock_reservation:*:<orderId>:*

# Cart backup
GET cart_backup:<userId>
```

> In production prefer `SCAN` instead of `KEYS` to avoid blocking.

---

## Screenshots

> Admin panel screenshots (admin access required)

### Orders & Fulfilment

![Orders — Paginated list](./docs/screenshots/admin_panel_orders_pagination.png)
*Orders list with pagination.*

![Orders — Dark mode list](./docs/screenshots/admin_panel_orders_dark.png)
*Orders list (dark theme).*

![Orders — Multi-order status update (light)](./docs/screenshots/admin_panel_orders_multi_order_status_update_light.png)
*Bulk update order statuses (multi-select).*

![Order — Expanded single order (light)](./docs/screenshots/admin_ordresList_expanded_order.png)
*Expanded order row showing details (light theme).*

![Order — Expanded & status change (light)](./docs/screenshots/adm_ordrs_expanded_order_statuschange_light.png)
*Expanded order with status change controls (light theme).*

![Order — Expanded & status change (dark)](./docs/screenshots/admin_expandedOrdr_statuschange_dark.png)
*Expanded order with status change controls (dark theme).*

![Order — Expanded customer view (dark)](./docs/screenshots/admin_expOrdr_custView_dark.png)
*Expanded order with customer view (dark theme).*

---

### Create Product & Variants

![Create product — Admin panel](./docs/screenshots/admin_panel_create_product.png)
*Admin UI — Create product (base fields).*

![Create product — Variants editor](./docs/screenshots/admin_panel_create_product_variants.png)
*Admin UI — Create product with variants (light theme).*

![Create product — Variants editor (dark)](./docs/screenshots/admin_panel_create_product_variants_dark.png)
*Admin UI — Create product with variants (dark theme).*

---

### Analytics & Campaigns

![Admin — Analytics dashboard (dark)](./docs/screenshots/admin_panel_analytics_dark.png)
*Admin analytics dashboard — sales, users, revenue.*

![Admin — Campaigns](./docs/screenshots/admin_campaigns.png)
*Marketing campaigns / mailing list management.*

---

## Commercial license (proprietary) & selling notes

Check LICENSE_PROPRIETARY.txt

**Selling notes**:

* Deliverables to include: source code, deployment scripts, environment setup, 30 days of post-delivery support, documentation, and optional maintenance plan.
* For buyers: the app is horizontally scalable — to increase capacity, upgrade MongoDB/Redis/hosting plans and scale server replicas behind a load balancer.

---

## Contact / commercial enquiries

For purchase, licensing, deployment assistance, or a demo with admin credentials contact:
**[ahmedmounib2@gmail.com](mailto:ahmedmounib2@gmail.com)**

---
