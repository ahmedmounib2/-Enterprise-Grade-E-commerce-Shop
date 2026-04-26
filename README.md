# AhmedMonib E-Shop — Enterprise-Grade E-commerce Shop

> Production-ready, enterprise-grade e-commerce storefront (React + Vite frontend, Node.js + Express
> backend) with:
>
> - Secure rotated refresh tokens and sliding 30-day sessions (mobile-aware keep-alive)
> - Product variants, per-variant stock & pricing, cart reservation and automatic stock restore on
>   cancel/expire
> - Stripe Checkout integration with robust webhook handling (PaymentIntent verification, refunds,
>   expired sessions)
> - Order export (PDF) & fulfilment tooling, GDPR data export, Sentry observability, and 89.28% test
>   coverage.

---

## Table of contents

- [AhmedMonib E-Shop — Enterprise-Grade E-commerce Shop](#ahmedmonib-e-shop--enterprise-grade-e-commerce-shop)
  - [Table of contents](#table-of-contents)
  - [Maintainer context](#maintainer-context)
  - [One-line pitch](#one-line-pitch)
  - [Live demo \& hosted domains](#live-demo--hosted-domains)
  - [Mobile app download links](#mobile-app-download-links)
  - [Monorepo architecture \& workspaces](#monorepo-architecture--workspaces)
    - [Workspaces](#workspaces)
    - [Common scripts](#common-scripts)
    - [Environment references](#environment-references)
    - [Multi-store / multi-tenant foundations](#multi-store--multi-tenant-foundations)
    - [Platform admin store \& seller-dashboard context](#platform-admin-store--seller-dashboard-context)
    - [Seller onboarding flow (design \& implementation)](#seller-onboarding-flow-design--implementation)
      - [Entry points \& UX design](#entry-points--ux-design)
      - [Data model design (Seller + Store)](#data-model-design-seller--store)
      - [API flow (step-by-step)](#api-flow-step-by-step)
      - [State machine (clear, enforced transitions)](#state-machine-clear-enforced-transitions)
      - [Security \& ownership controls](#security--ownership-controls)
      - [Post-approval seller experience](#post-approval-seller-experience)
      - [Operational notes](#operational-notes)
      - [Seller bank-update review lifecycle (pending/approved/rejected + seller cancel)](#seller-bank-update-review-lifecycle-pendingapprovedrejected--seller-cancel)
      - [Seller dashboard behavior for bank-update reviews](#seller-dashboard-behavior-for-bank-update-reviews)
      - [Backend contract for bank-update records](#backend-contract-for-bank-update-records)
      - [Bank-update API routes (seller + admin)](#bank-update-api-routes-seller--admin)
    - [Seller store settings, storefront policies, and admin review flow](#seller-store-settings-storefront-policies-and-admin-review-flow)
      - [Seller settings: editable fields \& review gates](#seller-settings-editable-fields--review-gates)
      - [What happens when a seller updates policies or branding](#what-happens-when-a-seller-updates-policies-or-branding)
      - [Admin review workflow (store profile + policy updates)](#admin-review-workflow-store-profile--policy-updates)
      - [Storefront policy rendering (public store page + checkout)](#storefront-policy-rendering-public-store-page--checkout)
      - [Pause store (Hide All) behavior](#pause-store-hide-all-behavior)
    - [Seller product creation \& admin moderation (end-to-end)](#seller-product-creation--admin-moderation-end-to-end)
      - [Flow overview (from seller draft to storefront)](#flow-overview-from-seller-draft-to-storefront)
      - [Data model \& status fields (seller-aware products)](#data-model--status-fields-seller-aware-products)
      - [Seller experience (create, save draft, submit)](#seller-experience-create-save-draft-submit)
      - [Backend pipeline (validation, images, ownership)](#backend-pipeline-validation-images-ownership)
      - [Admin moderation workflow (review, approve, reject)](#admin-moderation-workflow-review-approve-reject)
      - [State transitions \& audit trail](#state-transitions--audit-trail)
      - [Visibility rules \& storefront behavior](#visibility-rules--storefront-behavior)
      - [Operational safeguards \& failure handling](#operational-safeguards--failure-handling)
    - [Fee Engine \& Seller Subscription Architecture](#fee-engine--seller-subscription-architecture)
      - [FeeConfig defaults (plan commission + Stripe price)](#feeconfig-defaults-plan-commission--stripe-price)
      - [Commission rules hierarchy (global/seller/category)](#commission-rules-hierarchy-globalsellercategory)
      - [Commission calculation \& order locking](#commission-calculation--order-locking)
      - [Pro Plan subscription purchase flow (Stripe Checkout)](#pro-plan-subscription-purchase-flow-stripe-checkout)
      - [Subscription renewal \& expiration handling](#subscription-renewal--expiration-handling)
      - [Admin/operations notes](#adminoperations-notes)
  - [Subscription Billing Ledger](#subscription-billing-ledger)
    - [Architecture \& Design](#architecture--design)
      - [1) Separation from settlement payouts](#1-separation-from-settlement-payouts)
      - [2) End-to-end flow](#2-end-to-end-flow)
      - [3) Idempotency design (`stripeInvoiceId`)](#3-idempotency-design-stripeinvoiceid)
    - [Data Model Specification (`sellerBilling.model.js`)](#data-model-specification-sellerbillingmodeljs)
    - [API Specification](#api-specification)
      - [Seller billing history](#seller-billing-history)
      - [Admin billing history](#admin-billing-history)
    - [Webhook Handling Behavior](#webhook-handling-behavior)
      - [Event filtering and processing](#event-filtering-and-processing)
      - [Duplicate replay behavior and idempotent outcomes](#duplicate-replay-behavior-and-idempotent-outcomes)
      - [Failure handling and logging](#failure-handling-and-logging)
    - [Frontend Design](#frontend-design)
      - [Seller dashboard IA change](#seller-dashboard-ia-change)
      - [Billing History tab behavior](#billing-history-tab-behavior)
      - [UI state handling](#ui-state-handling)
      - [Explicit separation note](#explicit-separation-note)
    - [Settlement Isolation Guarantees](#settlement-isolation-guarantees)
    - [Test Coverage Matrix](#test-coverage-matrix)
    - [Backward Compatibility / Rollout Notes](#backward-compatibility--rollout-notes)
      - [Existing sellers with no billing records](#existing-sellers-with-no-billing-records)
      - [Index rollout considerations](#index-rollout-considerations)
      - [Post-deploy operational verification checklist](#post-deploy-operational-verification-checklist)
    - [Mobile Category Tree v2 (dynamic categories)](#mobile-category-tree-v2-dynamic-categories)
    - [DaisyUI theming (web + mobile)](#daisyui-theming-web--mobile)
    - [Wishlist DTO \& API contract](#wishlist-dto--api-contract)
    - [Public Store APIs](#public-store-apis)
      - [`GET /api/public/store-details/:slug`](#get-apipublicstore-detailsslug)
      - [`GET /api/public/stores/:id`](#get-apipublicstoresid)
      - [`GET /api/public/stores/search`](#get-apipublicstoressearch)
      - [Product payload note (`sellerSummary`)](#product-payload-note-sellersummary)
      - [Frontend integration notes](#frontend-integration-notes)
  - [Deployment surfaces \& release workflow](#deployment-surfaces--release-workflow)
    - [Frontend (Vercel)](#frontend-vercel)
    - [Backend (Railway Docker image)](#backend-railway-docker-image)
    - [Mobile app (Expo custom dev client)](#mobile-app-expo-custom-dev-client)
  - [Mobile deep-link flow overview](#mobile-deep-link-flow-overview)
  - [Mobile app architecture \& features](#mobile-app-architecture--features)
    - [Key capabilities](#key-capabilities)
    - [Native integrations](#native-integrations)
    - [Mobile documentation](#mobile-documentation)
    - [Mobile screenshots](#mobile-screenshots)
  - [What this repo contains](#what-this-repo-contains)
  - [Key features (summary)](#key-features-summary)
  - [Payment \& order flow (detailed)](#payment--order-flow-detailed)
    - [Checkout (Stripe) flow — `createCheckoutSession`](#checkout-stripe-flow--createcheckoutsession)
    - [Stripe webhooks \& async handling — `stripeWebhook`](#stripe-webhooks--async-handling--stripewebhook)
    - [COD flow (Cash-on-Delivery) — `createCODOrder` \& `codCheckoutSuccess`](#cod-flow-cash-on-delivery--createcodorder--codcheckoutsuccess)
    - [COD eligibility policy, seller financial summary, and recovery path](#cod-eligibility-policy-seller-financial-summary-and-recovery-path)
    - [Return request + COD refund workflow](#return-request--cod-refund-workflow)
    - [Provider architecture \& adapter contract (COD payouts)](#provider-architecture--adapter-contract-cod-payouts)
      - [Adapter input contract](#adapter-input-contract)
      - [Adapter output contract](#adapter-output-contract)
      - [Error handling notes (adapter layer)](#error-handling-notes-adapter-layer)
      - [Design goals](#design-goals)
      - [Runtime architecture (modules and responsibilities)](#runtime-architecture-modules-and-responsibilities)
      - [End-to-end lifecycle (happy path)](#end-to-end-lifecycle-happy-path)
      - [COD refund lifecycle (state-focused)](#cod-refund-lifecycle-state-focused)
      - [State model and transitions](#state-model-and-transitions)
      - [Data model mapping (what lives where)](#data-model-mapping-what-lives-where)
      - [Failure handling strategy](#failure-handling-strategy)
      - [Transaction boundaries and consistency model](#transaction-boundaries-and-consistency-model)
      - [Observability model](#observability-model)
      - [Runtime tuning envs and design impact](#runtime-tuning-envs-and-design-impact)
      - [Security model](#security-model)
    - [Extending to a new provider (implementation checklist)](#extending-to-a-new-provider-implementation-checklist)
      - [Idempotency behavior (safe retries)](#idempotency-behavior-safe-retries)
      - [How to detect stuck refunds / payouts](#how-to-detect-stuck-refunds--payouts)
      - [Safe stuck-refund retry procedure (runbook condensed)](#safe-stuck-refund-retry-procedure-runbook-condensed)
      - [Chimoney sandbox/config expectations](#chimoney-sandboxconfig-expectations)
      - [Chimoney webhook endpoint setup (sandbox dashboard)](#chimoney-webhook-endpoint-setup-sandbox-dashboard)
    - [Cron jobs added for COD payout resiliency](#cron-jobs-added-for-cod-payout-resiliency)
      - [1) COD payout reconciliation cron](#1-cod-payout-reconciliation-cron)
      - [2) COD refund details purge cron](#2-cod-refund-details-purge-cron)
    - [Key rotation \& webhook replay procedures](#key-rotation--webhook-replay-procedures)
      - [Rotate Chimoney API key safely](#rotate-chimoney-api-key-safely)
      - [Rotate Chimoney webhook secret safely](#rotate-chimoney-webhook-secret-safely)
      - [Replay webhook (admin/on-call detailed)](#replay-webhook-adminon-call-detailed)
      - [Troubleshooting: stuck pending refunds](#troubleshooting-stuck-pending-refunds)
      - [COD refund record API example (sanitized fields)](#cod-refund-record-api-example-sanitized-fields)
    - [Stock reservation \& restoration (variant-aware)](#stock-reservation--restoration-variant-aware)
    - [Failure \& expired session handling](#failure--expired-session-handling)
    - [Seller settlements, ledger lifecycle, and payout execution](#seller-settlements-ledger-lifecycle-and-payout-execution)
    - [Admin marketplace financials API (`/api/admin/platform-financials/*`)](#admin-marketplace-financials-api-apiadminplatform-financials)
      - [`GET /api/admin/platform-financials/summary`](#get-apiadminplatform-financialssummary)
      - [`GET /api/admin/platform-financials/ledger`](#get-apiadminplatform-financialsledger)
      - [`GET /api/admin/platform-financials/liabilities`](#get-apiadminplatform-financialsliabilities)
    - [Seller financial summary behavior updates](#seller-financial-summary-behavior-updates)
    - [Frontend financial documentation updates](#frontend-financial-documentation-updates)
      - [Seller dashboard financials scoping](#seller-dashboard-financials-scoping)
      - [Operator rows in seller ledger](#operator-rows-in-seller-ledger)
      - [Admin marketplace financials scope switcher](#admin-marketplace-financials-scope-switcher)
      - [UI diagram (scope flow)](#ui-diagram-scope-flow)
    - [Release notes — accounting and financial UI refactor](#release-notes--accounting-and-financial-ui-refactor)
      - [Consolidated changes in this release](#consolidated-changes-in-this-release)
      - [Frontend action required (short checklist)](#frontend-action-required-short-checklist)
    - [Reserve / Holdback Policy (implemented architecture)](#reserve--holdback-policy-implemented-architecture)
      - [1) Policy model implemented in this repo](#1-policy-model-implemented-in-this-repo)
      - [2) Ledger contract for reserves](#2-ledger-contract-for-reserves)
      - [3) Risk tiers, percentage resolution, and env defaults](#3-risk-tiers-percentage-resolution-and-env-defaults)
      - [Per-seller special agreements](#per-seller-special-agreements)
      - [4) Reserve release cron job (design + operations)](#4-reserve-release-cron-job-design--operations)
      - [5) Fixed reserve vs rolling reserve (when to choose each)](#5-fixed-reserve-vs-rolling-reserve-when-to-choose-each)
      - [6) Admin reserve APIs and UI](#6-admin-reserve-apis-and-ui)
        - [`GET /api/admin/reserves/summary`](#get-apiadminreservessummary)
        - [`GET /api/admin/reserves/sellers/:sellerId`](#get-apiadminreservessellerssellerid)
        - [`POST /api/admin/reserves/release`](#post-apiadminreservesrelease)
        - [`POST /api/admin/reserves/increase`](#post-apiadminreservesincrease)
        - [`POST /api/admin/reserves/update-config`](#post-apiadminreservesupdate-config)
      - [7) Manual test plan (reserve flows)](#7-manual-test-plan-reserve-flows)
    - [Ledger Adjustment Service](#ledger-adjustment-service)
      - [What it does](#what-it-does)
      - [API surface](#api-surface)
      - [Frontend admin panel](#frontend-admin-panel)
    - [Settlement Reconciliation \& Payout Recovery](#settlement-reconciliation--payout-recovery)
      - [1) Reconciliation Architecture (Backend)](#1-reconciliation-architecture-backend)
      - [2) Cron Job for Reconciliation](#2-cron-job-for-reconciliation)
      - [3) Environment Variables](#3-environment-variables)
      - [4) Admin Reconciliation Endpoints and UI Usage](#4-admin-reconciliation-endpoints-and-ui-usage)
      - [5) Payout Reversals](#5-payout-reversals)
      - [6) Runbook Linkage](#6-runbook-linkage)
    - [Settlement Payout Retry \& Cron Operations](#settlement-payout-retry--cron-operations)
      - [1) What changed](#1-what-changed)
        - [`executeStripePayout` retry behavior](#executestripepayout-retry-behavior)
        - [`reverseStripePayout` idempotency and retry behavior](#reversestripepayout-idempotency-and-retry-behavior)
        - [`SettlementPayout` schema/index hardening](#settlementpayout-schemaindex-hardening)
        - [New payout retry cron job](#new-payout-retry-cron-job)
        - [Job-health monitoring](#job-health-monitoring)
      - [2) Environment variables reference](#2-environment-variables-reference)
      - [3) Recommended value profiles](#3-recommended-value-profiles)
      - [4) Runbook / troubleshooting](#4-runbook--troubleshooting)
      - [5) Validation checklist (post-deploy)](#5-validation-checklist-post-deploy)
    - [Stripe Connect prerequisites (seller payouts)](#stripe-connect-prerequisites-seller-payouts)
  - [Products, variants \& model behavior](#products-variants--model-behavior)
  - [Dynamic categories system (tree + drag-and-drop)](#dynamic-categories-system-tree--drag-and-drop)
    - [Data model \& flags](#data-model--flags)
    - [Admin workflow (drag-and-drop + image upload)](#admin-workflow-drag-and-drop--image-upload)
    - [Customer browsing (categories-first, then products)](#customer-browsing-categories-first-then-products)
    - [Product creation \& image upload pipeline (frontend → backend → Cloudinary)](#product-creation--image-upload-pipeline-frontend--backend--cloudinary)
      - [1) Frontend (Create / Edit Product forms)](#1-frontend-create--edit-product-forms)
      - [2) Upload middleware (backend)](#2-upload-middleware-backend)
      - [3) Image processing (Sharp)](#3-image-processing-sharp)
      - [4) Cloudinary upload \& cleanup](#4-cloudinary-upload--cleanup)
      - [5) Variants \& image mapping](#5-variants--image-mapping)
      - [6) Storage of record](#6-storage-of-record)
      - [7) Environment (what to set)](#7-environment-what-to-set)
      - [8) Operational notes](#8-operational-notes)
  - [Auth \& session persistence — how it works (plain language)](#auth--session-persistence--how-it-works-plain-language)
  - [Order fulfilment, PDF export \& canceled order labeling](#order-fulfilment-pdf-export--canceled-order-labeling)
  - [GDPR \& user data export](#gdpr--user-data-export)
  - [Security \& observability](#security--observability)
  - [Testing \& CI](#testing--ci)
    - [How to run tests](#how-to-run-tests)
    - [How to test the COD refund purge cron job](#how-to-test-the-cod-refund-purge-cron-job)
    - [Test status](#test-status)
  - [How to run locally (no Docker)](#how-to-run-locally-no-docker)
    - [Local HTTPS for development (optional but recommended)](#local-https-for-development-optional-but-recommended)
  - [Environment variables used](#environment-variables-used)
    - [Example `.env` for local development (copy \& fill)](#example-env-for-local-development-copy--fill)
    - [Deployment platform variables (Railway / Vercel)](#deployment-platform-variables-railway--vercel)
  - [Redis (Optional)](#redis-optional)
    - [Architecture overview](#architecture-overview)
    - [Environment variables](#environment-variables)
    - [Redis configuration examples](#redis-configuration-examples)
    - [Redis debugging quick commands (WSL/Ubuntu)](#redis-debugging-quick-commands-wslubuntu)
      - [Install Redis locally (dev sessions)](#install-redis-locally-dev-sessions)
      - [Start Redis](#start-redis)
      - [Verify it’s running](#verify-its-running)
      - [Useful inspection commands](#useful-inspection-commands)
  - [Screenshots](#screenshots)
    - [Orders \& Fulfilment](#orders--fulfilment)
    - [Create Product \& Variants](#create-product--variants)
    - [Analytics \& Campaigns](#analytics--campaigns)
    - [Emails](#emails)
    - [Mobile](#mobile)
  - [Commercial license (proprietary) \& selling notes](#commercial-license-proprietary--selling-notes)
  - [Demo credentials (web \& mobile)](#demo-credentials-web--mobile)
  - [Additional documentation](#additional-documentation)
  - [Contact / commercial enquiries](#contact--commercial-enquiries)

---

## Maintainer context

This project is built and maintained by a single freelance developer. Any "playbook" style
documentation in [`docs/`](docs/) is optional scaffolding kept around for potential future growth of
the project into a team setting; it is **not** required for day-to-day solo development. Feel free
to ignore those extras unless you want structured guidance when collaborating with additional
contributors later on.

---

## One-line pitch

Enterprise-grade e-commerce web storefront and Expo mobile app sharing a single Node/Express API,
complete with robust session security, per-variant inventory, reservation/restore logic, Stripe
payments, order fulfilment tooling, and production observability.

---

## Live demo & hosted domains

- Frontend (production): `https://www.ahmedmonib-eshop-demo.com` (Vercel)
- Frontend (alternate / staging): `https://ecommerce-mern-website-seven.vercel.app`
- API (production): `https://api.ahmedmonib-eshop-demo.com` (Railway)
- Railway preview: `https://e-commerce-mern-stack-website-production.up.railway.app`

---

## Mobile app download links

- **Google Play (production)** —
  <https://play.google.com/store/apps/details?id=com.ahmedmonib.eshop>
- **QR code** —

- ![Mobile — QR Code](./docs/screenshots/play-eshop-qr.png)

---

## Monorepo architecture & workspaces

This repository is an npm **workspace monorepo**. Every app or shared package lives in its own
workspace so tooling, builds, and deployments stay isolated but share a single lockfile.

### Workspaces

| Workspace    | Path        | Purpose                                                                                        |
| ------------ | ----------- | ---------------------------------------------------------------------------------------------- |
| `frontend`   | `frontend/` | React (Vite) SPA deployed to Vercel.                                                           |
| `backend`    | `backend/`  | Express API deployed to Railway using the root Dockerfile.                                     |
| `mobile`     | `mobile/`   | Expo + React Native custom development client (Android) with deep-link support.                |
| `shared`     | `shared/`   | Localization bundle consumed by web and mobile clients.                                        |
| `packages/*` | `packages/` | Shared Axios API client (`@eshop/api-client`) with optional SSL pinning for Expo/React Native. |

|

### Common scripts

Run scripts from the repo root unless noted otherwise:

```bash
npm install                     # bootstrap every workspace with a single lockfile
npm run dev                     # concurrently start backend (https://localhost:5001) + frontend (https://localhost:5173)
npm run dev:all                 # start backend, frontend, and Expo dev server together
npm -w backend run dev          # backend only (nodemon + HTTPS)
npm -w frontend run dev         # frontend only (Vite dev server)
npm -w mobile run start:tunnel  # Expo dev client over tunnel with cache bust (-c)
npm -w mobile run env:<profile> # copy mobile/.env.<profile> -> mobile/.env (emu|lan|tunnel)
```

Additional workspace scripts:

```bash
npm -w backend test             # backend Jest suite
npm -w backend run migrate:settlement-payout-indexes # migrate settlement payout externalTransferId unique partial index
npm -w frontend run build       # production build for Vercel
npm -w mobile run android       # rebuild/install the custom dev client (native changes)
npm -w mobile run start:local   # Metro bound to localhost (Android emulator)
npm -w mobile run start:lan     # Metro on LAN (update HOSTNAME env in package.json first)
```

### Environment references

Solo development rarely needs heavy process docs, but these optional references are available if you
decide to collaborate with others or formalise operations:

- [`mobile/env.md`](mobile/env.md) — environments, workspace responsibilities, run modes, deployment
  checklists.
- [`mobile/dev-setup.md`](mobile/dev-setup.md) — Android toolchain, device connectivity, Expo
  workflows.
- [`docs/deployment.md`](docs/deployment.md) — optional release runbook covering frontend, backend,
  and mobile rollouts when multiple people coordinate deployments.
- [`docs/security/access-controls.md`](docs/security/access-controls.md) — optional template for
  documenting account provisioning and least-privilege roles once more contributors join.
- [`docs/INCIDENT_RESPONSE.md`](docs/INCIDENT_RESPONSE.md) — optional production incident playbook
  if you later need structured response expectations.

Mobile-only feature flags live in `mobile/.env.*`. To exercise the Category Tree v2 UI, set
`EXPO_PUBLIC_FEATURE_MOBILE_CATEGORY_TREE_V2=true` in your dev/tunnel profile; leave it `false`
until you are ready to launch in production, then flip it to `true` in `.env.production` as well
(`.env` files on Vercel/Railway do not need this flag).

### Multi-store / multi-tenant foundations

- Multi-store support is an **optional capability**: the foundations are in place and can be
  enabled, disabled, or paused safely through environment flags.
- Dedicated **Seller** and **Store** models include ownership metadata and scoped indexes on
  `sellerId` and `slug` to support tenant isolation.
- Added dedicated **Seller** and **Store** models with ownership metadata and scoped indexes on
  `sellerId` and `slug` to support isolation.
- Products and orders carry seller/store references plus approval, visibility, and payout status
  fields; required indexes are synced during migrations for sellers, stores, products, and orders.
- Run the idempotent migration to seed the platform admin seller/store and backfill existing data:
  `npm -w backend exec node src/migrations/0001_multi_tenant_foundation.js` (requires the usual DB
  env vars such as `MONGO_URI`).
- Public store lookup is available at `GET /api/stores/{slug}` and only surfaces the `StorePublic`
  fields when the multi-seller flag is enabled.
- Multi-store tenant flags:
  - `FEATURE_MULTI_SELLER=true`
  - `FEATURE_SELLER_KYC=true`
  - `FEATURE_SELLER_ORDERS=true`
- Keep backend/frontend flags aligned per environment so APIs and UI stay in sync (for example,
  mirror KYC flags with `VITE_FEATURE_SELLER_KYC=true` in `frontend/.env` and your Vercel
  environment settings when needed).

### Platform admin store & seller-dashboard context

The platform (main website) storefront is provisioned as a normal `Seller` + `Store` pair so
first-party products and payouts use the same architecture as marketplace sellers.

- **Idempotent ensure at server startup**
  - On backend startup, `ensureAdminStore` ensures platform records exist.
  - If platform seller/store already exist, they are updated safely; if missing, they are created.
- **Provisioning defaults**
  - Seller slug defaults to `admin` (override-aware by env conventions in code paths).
  - Seller is ensured as `status: active` and `kyc.status: verified`.
  - Store slug defaults to `admin-store`.
  - Store name is configurable with `ADMIN_STORE_NAME` (fallback: `Admin Store`).
- **Admin/staff seller-dashboard behavior**
  - `requireSeller` maps `admin`/`staff` users to platform seller context (`req.seller` and
    `req.context.sellerId`) so `/api/seller/*` routes are seller-scoped to the platform store.
  - Seller Orders routes allow `seller`, `admin`, and `staff` roles while still enforcing seller
    scope through `req.context.sellerId`.
- **Platform product behavior**
  - Products created in platform seller context are created live (`approvalStatus=approved`,
    `visibility=visible`, `hidden=false`).
  - Platform seller context bypasses Free/Pro product count and featured-item caps.
  - Admin product edits no longer overwrite ownership (`sellerId` / `storeId`) on existing products.
- **Store access behavior**
  - Protected `GET /api/stores/:id` supports authenticated admin/staff (or owner) retrieval.
  - Public `GET /api/stores/:slug` eligibility rules remain unchanged for regular sellers
    (`status=active` + `kyc.status=verified`).
- **Financials scope for platform seller dashboard**
  - Platform seller financial summary/ledger on seller dashboard is scoped to platform-owned orders
    only (entries tied to orders containing platform-seller items), avoiding marketplace-wide
    liability noise.
- **Related env vars**
  - `PLATFORM_SELLER_ID`
  - `PLATFORM_SELLER_SLUG` (optional fallback behavior in code)
  - `ADMIN_STORE_ID`
  - `ADMIN_STORE_NAME`

### Seller onboarding flow (design & implementation)

The seller onboarding flow is designed as a **KYC-first onboarding pipeline** with explicit
ownership scoping, document capture, and staged activation. It ensures that any seller-visible
feature requires an approved, verified seller record plus a matching store profile. The flow spans
frontend entry points, backend application APIs, KYC administration, and post-approval store
provisioning. The implementation focuses on three priorities: **secure ownership**, **clear state
transitions**, and **audit-friendly data snapshots**.

#### Entry points & UX design

- **Pre-application gate:** `/seller/pre-apply` explains requirements and prevents accidental
  submissions before the seller is ready with documents. This provides a soft landing zone before
  sharing sensitive data.
- **Application form:** `/seller/apply` collects business identity, store name, contact info, and
  banking/settlement details. It builds a unified `application` payload that is stored on the Seller
  model rather than scattering fields across multiple models.
- **Access gating:** Seller routes are blocked for non-sellers or non-verified sellers. The UI
  checks the authenticated user’s seller status and KYC status before allowing access to the seller
  dashboard, giving a consistent “access denied” path for admins or standard users.

#### Data model design (Seller + Store)

- **Seller** is the primary identity for onboarding. Key fields include:
  - `ownerUserId` (ownership linkage to the user)
  - `businessName` / `slug`
  - `status` (`active` / `inactive`)
  - `kyc.status` (`draft`, `pending`, `verified`, `rejected`, `action_required`)
  - `application` (normalized KYC + business data payload)
  - `application.banking` (encrypted with `SELLER_DATA_SECRET` to minimize exposure at rest)
- **Store** represents the customer-facing storefront and is **created only after approval**. Store
  data is generated from the seller’s approved profile and defaults to a safe baseline (branding,
  policies, visibility) until the seller completes store customization.
- **Admin store (platform-owned storefront):** During server setup, an idempotent migration creates
  a **platform seller + store** that powers the main website experience (e.g., “Sold by
  E-commerce/Main Store”). This store acts as the default merchant for legacy products, platform-run
  inventory, and “first-party” listings while keeping the seller/onboarding system cleanly
  separated. It shares the same Seller/Store schema but is owned by the platform admin and is
  provisioned before any third-party seller onboarding begins.

#### API flow (step-by-step)

1. **Create/Update application** — `POST /api/sellers/apply`
   - Authenticated users submit their seller application.
   - The backend either creates a new Seller or updates the existing Seller tied to `ownerUserId`.
   - The application can be saved as **draft** or **pending** (submit for review), which lets
     sellers progressively complete details without triggering review too early.
   - Banking payloads are encrypted and stored in `application.banking`.
   - Payout-country validation is based on the submitted bank country (`bankCountry`), not
     `citizenshipCountry`. This keeps KYC identity fields decoupled from payout rail eligibility.
2. **Upload documents** — `POST /api/sellers/:id/docs`
   - Sellers upload KYC documents (IDs, registrations, etc.).
   - Each document is stored in `SellerDocument`, linked by `sellerId`. This keeps document metadata
     isolated from the seller profile and supports audit trails.
   - For `bank_statement` uploads during onboarding, the backend now increments `version` using the
     same version resolver used by the bank-update flow, so repeated uploads are explicitly ordered
     by version (with `createdAt` as a tie-breaker).
3. **Review status** — `GET /api/sellers/me`
   - Sellers retrieve their onboarding status (application, KYC status, documents, and store
     provisioning state) in a single response.
4. **Admin decision** — `PATCH /api/sellers/admin/:id/verify`
   - Admins mark KYC as `verified`, `rejected`, or `action_required`.
   - Rejections store both `kyc.rejectionReason` and a frozen `kyc.rejectedApplication` snapshot for
     auditability.
   - For `action_required`, sellers can re-submit without losing their previous submission context.
5. **Store provisioning** (auto after approval)
   - On approval, the backend ensures a Store exists for the seller, using defaults derived from the
     seller profile (store name, slug, branding defaults, and basic policies).
   - Sellers can then configure their store via the seller dashboard settings page.

#### State machine (clear, enforced transitions)

| Seller `status` | KYC `status`      | Meaning / Access                                                         |
| --------------- | ----------------- | ------------------------------------------------------------------------ |
| `inactive`      | `draft`           | Application saved but not submitted. Seller dashboard access is blocked. |
| `inactive`      | `pending`         | Submitted for review; admin action required.                             |
| `inactive`      | `action_required` | Seller must re-submit updates; previous data is preserved.               |
| `inactive`      | `rejected`        | Rejected with reason; seller must restart or re-apply.                   |
| `active`        | `verified`        | Approved; full seller access unlocked and store provisioning enabled.    |

#### Security & ownership controls

- **Ownership enforcement:** All seller endpoints use `protectRoute` + role restriction plus
  `requireSeller`, ensuring only the seller who owns a seller profile can access seller APIs.
- **Data isolation:** Seller-scoped queries use `sellerId` consistently (orders, products, store
  settings) to prevent cross-tenant leakage.
- **Encrypted banking data:** Sensitive settlement details are stored with encryption using
  `SELLER_DATA_SECRET`, limiting exposure if raw DB access occurs.
- **Email notifications:** Seller KYC status changes trigger transactional email updates (approved,
  rejected, or action required) so sellers receive a clear next step.

#### Post-approval seller experience

Once `status=active` and `kyc.status=verified`, seller access unlocks in phases so compliance, store
readiness, and payout setup stay aligned:

1. **Immediate dashboard access**
   - Seller can open `/seller/dashboard` and manage store profile data, policies, and approved
     catalog workflows.
   - Order tools are seller-scoped (only their own `sellerId` records) and can be exposed behind
     `FEATURE_SELLER_ORDERS=true`.

2. **Required legal consents in dashboard**
   - Before full product workflow readiness, seller must accept both Privacy Policy and Terms from
     the seller dashboard consent gate.
   - Consent is persisted on the store record (`privacyAccepted`, `privacyAcceptedAt`,
     `termsAccepted`, `termsAcceptedAt`) and is designed to be idempotent (once accepted, the seller
     is not repeatedly blocked for the same consent).

3. **Store readiness checks before listing**
   - Seller profile completion enforces core storefront requirements (store identity fields +
     shipping/return policy content + accepted legal consents) before allowing product creation
     flows.

4. **Payout setup via Stripe external onboarding link**
   - From the seller payment tab, the system creates a Stripe Connect onboarding link
     (`POST /seller/connect/onboarding-link`) and redirects the seller to Stripe-hosted onboarding.
   - Stripe returns the seller back to `/seller/dashboard?tab=payment&stripeConnectReturn=1`, where
     the app refreshes `GET /seller/connect/status` and shows whether payouts are fully enabled or
     still require action.
   - If onboarding is incomplete, the payout-readiness banner remains visible with requirement hints
     until Stripe account requirements are satisfied.

5. **After onboarding is complete**
   - Sellers continue managing products, orders, policies, and settings within their tenant scope.
   - If a store is paused or status changes later, storefront visibility can be updated without
     revoking historical KYC verification data.

#### Operational notes

- The onboarding flow is intentionally **idempotent**: repeated submissions update the same seller
  record instead of creating duplicates.
- Drafts allow multi-step completion, while pending submissions lock the application for review.
- Rejections preserve a snapshot of the submitted application to support support reviews or audits.

#### Seller bank-update review lifecycle (pending/approved/rejected + seller cancel)

Bank updates for already-verified sellers now run on a dedicated review lifecycle that is separate
from seller-application KYC statuses. The current lifecycle is:

`pending` → (`approved` | `rejected`) with a seller-side cancel action available from `pending` and
`rejected`.

There is **no `action_required` state for bank updates**. Any legacy reference to bank-update
`action_required` should be treated as obsolete.

| Bank update review status | Who sets it           | Meaning                                                                |
| ------------------------- | --------------------- | ---------------------------------------------------------------------- |
| `pending`                 | Seller submits update | Admin review is required; storefront/product creation remains paused.  |
| `approved`                | Admin approve action  | New bank details are promoted to approved payout bank fields.          |
| `rejected`                | Admin reject action   | Submitted bank update is denied; admin notes explain why.              |
| `cancelled` (UI outcome)  | Seller cancel action  | Seller reverts to last approved bank profile and clears active review. |

State transition summary:

1. Seller submits a bank update (`pending`).
2. Admin approves (`approved`) or rejects (`rejected`).
3. Seller can cancel while review is `pending` or after `rejected`; cancel restores the last
   approved bank profile and clears the in-flight review metadata.

#### Seller dashboard behavior for bank-update reviews

The seller dashboard payment-method flow now has explicit UX rules:

- **Pending header copy:** pending requests show a clear “under review” header and messaging that
  product creation/storefront visibility are paused until review resolves.
- **Denied header + reason:** rejected requests show a denial header and render admin review notes
  as the reason.
- **Always-visible cancel button:** a **Cancel and restore last approved bank** action is always
  visible on bank-update review cards (enabled only when status is cancellable).
- **Application Details tab:** review UIs include an **Application Details** tab showing submitted
  bank identity and statement metadata for traceability.

#### Backend contract for bank-update records

The backend model contract is intentionally strict so UI and admin tooling can rely on stable
semantics:

- Seller KYC bank documents use a single canonical type: `bank_statement`. New records must not use
  the legacy `bank` type.
- Bank statement documents now include explicit phase metadata: `isOnboarding=true` for onboarding
  uploads before seller approval and `isOnboarding=false` for post-approval bank-update uploads.
- bank statement `version` is incremented in both onboarding and bank-update upload paths, so the
  latest statement is deterministic without relying on legacy type aliases.
- `seller.updateReview` is the active review envelope for bank updates (`type/requestType`
  `bank_update`).
- `updateReview.status` for bank updates is constrained to `pending`, `approved`, or `rejected`.
- On deny, the submitted payload is copied into an archived record (`ArchivedBankUpdate`) and the
  mutable in-review payload is cleared from the active review envelope.
- `seller.payout.bank` is the approved source of truth for live settlement data. UIs should treat
  this as canonical approved bank information.
- Seller cancel restores `application.banking` from `payout.bank` and clears `updateReview`, so the
  dashboard returns to the last approved bank baseline.

#### Bank-update API routes (seller + admin)

Use the following API routes for the full bank-update lifecycle:

- **Submit bank update (seller):** `POST /api/seller/payment-method`
  - Submit one-time bank token + statement metadata to start/replace a bank-update review.
  - Backend immediately exchanges the token with Stripe (`createExternalAccount`) and updates the
    seller's connected external bank account on their Stripe Connect account.
  - Creates/updates `updateReview` with `status=pending` for verified active sellers.
- **Admin approve/deny review:** `PATCH /api/sellers/admin/:id/update-review?action=approve|reject`
  - `approve`: promotes previously persisted non-sensitive bank metadata to `payout.bank` and marks
    review `approved`.
  - Approval does **not** reuse or persist bank token identifiers; token use is submit-time only.
  - `reject`: marks review `rejected`, requires notes, archives denied update snapshot.
- **Seller cancel update review:**
  - `POST /api/seller/payment-method/cancel-bank-update`
  - `POST /api/seller/bank-update/cancel` (alias)
  - Allowed when current bank-update review is `pending` or `rejected`; restores last approved bank
    fields and clears active review state.
  - The primary cancel endpoint is intentionally excluded from the global API rate limiter to avoid
    blocking seller recovery actions with `429` during repeated cancel/retry attempts.
    `action_required` remains valid for seller-application KYC and store-profile review workflows,
    but it is **not** part of the bank-update review state machine. The KYC document access layer
    now enforces a **view-first, policy-driven** model:

- **Inline viewer is the default** for all roles. Document access uses short-lived view tokens and
  renders with `Content-Disposition: inline`.
- **Tiered controls are enforced per document**:
  - **Tier 1** (`identity`, `authorization`) is **view-only**.
  - **Tier 2** documents can be viewed and are eligible for tightly controlled admin download.
- **Audit logging is built in** for view token creation, viewer access attempts, frontend viewer
  events, and download attempts/outcomes.
- **Tier 2 downloads are admin-only** and pass through a watermarking step before file delivery.

Current endpoint behavior:

- `GET /api/sellers/docs/:docId/view-token` — seller/admin authorized token issue for viewer use.
- `GET /api/sellers/admin/docs/:docId/view-token` — admin-scoped token issuance.
- `GET /api/sellers/docs/view/:token` — secure inline render endpoint.
- `POST /api/sellers/docs/:docId/audit-events` — client-side viewer audit event ingestion.
- `GET /api/sellers/admin/docs/:docId/download` — admin-only Tier 2 download with watermarking.
- `GET /api/sellers/docs/:docId/download` — deprecated and intentionally returns `410` (do not use).
- `GET /api/sellers/admin/docs/audit` — admin audit query endpoint for operational/compliance
  review.

Configuration and operational notes for maintainers:

- `FEATURE_SELLER_KYC=true` must remain enabled where seller onboarding/KYC is expected.
- `SELLER_DOC_VIEW_TOKEN_SECRET` should be set explicitly in each environment; if omitted, the
  service falls back to `ACCESS_TOKEN_SECRET`.
- Treat document downloads as an exception path: operational SOP should direct teams to the inline
  viewer first and reserve Tier 2 download for approved admin workflows.
- Any older examples, scripts, or runbooks that imply unrestricted seller/admin KYC downloads should
  be updated to the tiered policy above.

### Seller store settings, storefront policies, and admin review flow

This section documents how **seller settings and store policies** are edited, when changes require
admin review, and how the public storefront stays consistent while reviews are pending. The flow is
split into seller UI behavior, backend review gating, and storefront policy rendering.

#### Seller settings: editable fields & review gates

Seller settings live in the seller dashboard’s **Store profile** page and are backed by the store
profile API. The editable fields are grouped into **reviewed** vs **instant** updates:

**Fields that require admin review (store review pipeline):**

- **Store name** (only before KYC approval; after verification it is locked for sellers).
- **Store description** (long-form description shown on the store page).
- **Logo + banner images** (branding assets).
- **SEO tagline + short description** (SEO metadata for the store page).
- **Shipping policy + return/refund policy** (seller policies shown to buyers).

When any of the above change, the backend marks the store profile as `reviewStatus=pending` and
captures a **review snapshot** of the last approved profile so the storefront can keep showing
approved content. Sellers see these fields **locked** while review is pending.

**Fields that can be updated without review:**

- **Support email + support phone** (customer contact info).
- **Profile categories** (up to 5 category tags + optional category descriptions).
- **Branding object overrides** (theme/branding metadata stored under `branding`).

These updates save immediately and do **not** require admin approval, unless a store review is
already pending, in which case the same fields are temporarily locked to avoid conflicting edits.

#### What happens when a seller updates policies or branding

When a seller hits **Save** on the store settings page:

1. The backend validates required fields (store name, logo, description, shipping policy, return
   policy).
2. If a **reviewed** field changed, the backend:
   - Stores a **review snapshot** of the last approved profile.
   - Marks the store as `reviewStatus=pending`.
   - Locks store branding + policy fields until a decision is made.
3. The store profile record **still stores the new edits** immediately, but the public storefront
   continues to serve the **previous snapshot** until approval.
4. Sellers see a review banner in the settings UI when changes are **rejected** or **action
   required**, including the admin’s notes.

Policy edits are **forward-looking only**: the seller UI explicitly warns that **new orders** use
the latest approved policies, while prior orders keep their original policy snapshot at the time of
purchase.

#### Admin review workflow (store profile + policy updates)

Store profile reviews appear in the admin **Seller Requests → Policy update reviews** tab:

1. Admins can list store review requests by `reviewStatus` using the admin store review API.
2. The review screen displays:
   - **Live approved** store description + policies.
   - **Submitted changes** with highlighted diffs.
   - Branding previews (logo + banner).
3. Admin decisions:
   - **Approve** → `reviewStatus=approved`, clear `reviewSnapshot`, the new profile goes live.
   - **Reject** → `reviewStatus=rejected`, keep `reviewNotes`, storefront stays on last approved
     snapshot.
   - **Action required** → `reviewStatus=action_required` with required notes.
4. Email notifications are sent automatically to the seller on approve/reject/action required.

This workflow ensures **brand/policy edits are moderated** while the storefront remains stable for
buyers.

#### Storefront policy rendering (public store page + checkout)

**Public store pages** always read policy and branding data from the **last approved snapshot** if a
store review is pending. This means that while a seller is editing or awaiting approval, customers
see the **previously approved** shipping/return policies and branding details on:

- Storefront tabs (Shipping policy + Return & refund policy).
- Store profile branding (logo, banner, description).

**Checkout policy snapshots** are captured at order creation time from the **approved store
profile** so that order history can always display the exact policy the buyer agreed to.
Specifically:

- During checkout, the backend resolves the **approved** policy source (current store profile if
  `reviewStatus=approved`, otherwise the `reviewSnapshot`) and derives the **shipping** and
  **return/refund** policy text.
- These values are persisted on the **Order** as immutable fields (e.g.,
  `orderShippingPolicy`/`orderReturnPolicy` or `order_shipping_policy`/`order_return_policy`
  depending on API shape).
- Because the order record stores the policy text at purchase time, **future policy edits do not
  retroactively alter** the buyer’s stored terms. Order detail views always read from the order
  snapshot rather than the seller’s live settings.

#### Pause store (Hide All) behavior

The seller settings page includes a **Pause store** (Hide All) action that lets sellers temporarily
hide all of their products **without changing policies**. When enabled:

- The store status becomes `hidden`.
- All seller products are set to **hidden** with a `pausedByStore` flag.
- Customers no longer see the store’s listings in search/browse.

When the seller resumes the store, products paused by the store are reactivated. This is designed as
a **temporary safety switch** for sellers who want to pause orders while policies are being reviewed
or updated.reviewed or updated.

### Seller product creation & admin moderation (end-to-end)

This section documents how **seller-owned products** move from draft creation to admin moderation
and finally into storefront visibility. The flow is designed to keep strict ownership boundaries
between sellers, enforce content quality through moderation, and guarantee the storefront never
shows unreviewed inventory. Think of it as a staged pipeline with explicit status fields,
human-in-the-loop review, and an auditable trail.

#### Flow overview (from seller draft to storefront)

1. **Seller creates product draft** in the seller dashboard.
2. **Images + variants are uploaded and validated** through the same pipeline used by admin products
   (Sharp + Cloudinary), but the product is flagged as seller-owned.
3. **Seller submits for review** (state moves from `draft` → `pending`).
4. **Admin reviews the submission** inside the moderation queue.
5. **Admin approves or rejects**:
   - **Approved** → product becomes `approved` and can be visible in storefront listings.
   - **Rejected** → seller receives reason and can edit & resubmit.
6. **Seller updates product** (if needed) and repeats the submit process.

This keeps the storefront clean (no unreviewed listings) while still letting sellers iterate quickly
in draft mode.

#### Data model & status fields (seller-aware products)

Seller products extend the base Product model with seller-scoped metadata and moderation status
fields. The primary fields that drive this flow include:

- `sellerId` + `storeId`: ownership scope, used to filter product access and listing results.
- `createdBy`: the authenticated user who created the product (for audit traces).
- `visibility`: determines if the product can be surfaced (`public`, `hidden`, `archived`).
- `approvalStatus`: moderation state (`draft`, `pending`, `approved`, `rejected`).
- `moderation`: optional admin notes (decision reason, reviewer ID, timestamps).

These fields allow the backend to enforce seller isolation and keep admin decisions transparent and
traceable.

#### Seller experience (create, save draft, submit)

**Seller dashboard (create product):**

- The seller fills in base fields (title, description, category, pricing).
- Variants and images are created just like the platform admin UI, but scoped to their store.
- The UI treats the product as a **draft** until the seller explicitly clicks **Submit for review**.

**Save flows:**

- **Save Draft** → persists the product with `approvalStatus=draft`.
- **Submit for review** → sends a status update to `pending` and locks fields that require review
  (admin-side approval required before public visibility).

If a seller edits an already-approved product, the system can re-open moderation (e.g., move back to
`pending`) depending on the fields changed. This preserves quality without blocking minor metadata
changes unnecessarily.

#### Backend pipeline (validation, images, ownership)

The backend applies the same robust product creation pipeline used by admins, plus seller-specific
guards:

1. **Authentication + seller validation**
   - Requires an authenticated user with `seller` role.
   - Verifies the seller is `active` with `kyc.status=verified`.
2. **Ownership scoping**
   - `sellerId` and `storeId` are always bound server-side; the client cannot spoof them.
   - This prevents sellers from attaching products to another store.
3. **Payload validation**
   - Enforces required fields (name, price, category, inventory).
   - Validates variants and SKU uniqueness within the seller scope.
4. **Image pipeline**
   - Uses Sharp to normalize images and Cloudinary to store final assets.
   - Temporary files are cleaned up after upload.
5. **Default moderation fields**
   - New seller products default to `approvalStatus=draft` (or `pending` if submitted).
   - `visibility` defaults to `hidden` until approval.

#### Admin moderation workflow (review, approve, reject)

Admins have a moderation queue scoped to seller submissions:

1. **Review list** shows all `approvalStatus=pending` products.
2. **Detail view** includes:
   - Seller identity (business name, store name).
   - Product metadata, variants, images, category mapping.
   - Historical submissions / prior rejection notes.
3. **Decision**
   - **Approve** → sets `approvalStatus=approved`, `visibility=public` (or scheduled visibility).
   - **Reject** → sets `approvalStatus=rejected`, `visibility=hidden`, and writes a rejection
     reason.
4. **Notification**
   - Seller receives a status update in dashboard (and optionally email).

This workflow ensures every seller product has a clear admin decision before being shown in public
lists or categories.

#### State transitions & audit trail

The flow is intentionally explicit to avoid ambiguous states:

| Current status | Action                                 | Next status | Notes                                                                    |
| -------------- | -------------------------------------- | ----------- | ------------------------------------------------------------------------ |
| `draft`        | Seller saves without submitting        | `draft`     | Editable by seller; never visible publicly.                              |
| `draft`        | Seller submits for review              | `pending`   | Locks fields that require moderation.                                    |
| `pending`      | Admin approves                         | `approved`  | Eligible for storefront when visibility rules also allow it.             |
| `pending`      | Admin rejects                          | `rejected`  | Seller receives reason and must edit before re-submit.                   |
| `rejected`     | Seller edits and re-submits            | `pending`   | Starts a new review cycle; prior rejection context is retained.          |
| `approved`     | Seller edits moderation-sensitive data | `pending`   | Re-review gate for core listing changes.                                 |
| `approved`     | Admin unpublishes/archives             | `hidden`    | Listing remains in system for audit/reporting but is removed storefront. |
| `hidden`       | Admin republishes (if policy permits)  | `approved`  | Returns to approved lifecycle without creating a new product record.     |

Audit trail expectations:

- Every moderation decision stores actor (`adminId`/seller), timestamp, and reason metadata.
- Rejections preserve reviewer notes for seller feedback and operational follow-up.
- Status history is retained to support disputes, compliance review, and rollback investigations.

#### Visibility rules & storefront behavior

Storefront queries enforce the visibility rules:

- Only `approvalStatus=approved` products are allowed in public listings.
- `visibility=public` is required for storefront exposure.
- Admin can soft-hide products without deleting them.
- Seller drafts and rejected items remain accessible only inside seller dashboards.

This separation guarantees that the public catalog only contains approved listings.

#### Operational safeguards & failure handling

- **Idempotency:** repeated submissions update the same product record; duplicates are not created.
- **Validation errors:** the API returns structured validation responses (field-level errors), and
  images are rolled back if the product fails validation.
- **Image failures:** failed Cloudinary uploads trigger cleanup of temporary files and return a
  retryable error to the seller UI.
- **Moderation consistency:** any attempted public visibility change without approval is rejected
  server-side.
- **Soft delete:** archived products are retained for reporting and audit purposes.

### Fee Engine & Seller Subscription Architecture

The platform implements a **fee engine** that combines plan-based defaults (Free vs. Pro) with
time-bound commission rules and a Stripe-backed subscription system. The end goal is to let admins
set platform economics in the database while keeping commission calculations deterministic and
auditable for every order.

#### FeeConfig defaults (plan commission + Stripe price)

The **FeeConfig** collection is the **source of truth** for baseline commission rates, plan limits,
and the Pro plan price metadata. Each configuration record includes:

- `freeCommissionRate` — percentage for Free plan sellers.
- `proCommissionRate` — percentage for Pro plan sellers.
- `minimumFee` — fixed minimum fee per item (applied per unit).
- `freeMaxProducts` / `freeMaxFeatured` — Free plan product + featured product limits.
- `proMaxProducts` / `proMaxFeatured` — Pro plan product + featured product limits.
- `proPlanPriceAmount` / `proPlanPriceCurrency` — stored for **read-only display** of plan pricing.
- `stripePriceId` — Stripe Price ID for the Pro subscription checkout + renewals.
- `effectiveStartDate` / `effectiveEndDate` — date range for when the config is active.

At runtime the API loads the **active FeeConfig** (effective date window containing “now” and
highest `effectiveStartDate`) and applies it across order calculations and subscription pricing.
Commission rates, plan limits, and Stripe Price IDs are **not** controlled by environment variables;
they live in FeeConfig so admins can adjust them without redeploying. The only exception is the
optional `STRIPE_PRO_PRICE_ID` fallback (used when no active FeeConfig is found), but the DB value
always takes precedence. The Pro plan price amount/currency are stored to show admins and sellers
the current display price; Stripe remains the billing source of truth.

#### Commission rules hierarchy (global/seller/category)

Beyond the FeeConfig defaults, **CommissionRule** records allow time-bound overrides:

- `scope`: `global`, `seller`, or `category`.
- `percentageRate` and `minimumFee` for the override.
- `effectiveStartDate` / `effectiveEndDate` to activate the rule.
- Optional `sellerId` or `categoryId` targets (required for their scope).

When calculating fees, the system **filters to only active rules** (date range includes now) and
chooses the most specific rule in this order:

1. **Category rule** (if the product category has an active rule).
2. **Seller rule** (if the seller has an active rule).
3. **Global rule** (if defined).

If no rule matches, the fee engine falls back to the plan-based defaults in FeeConfig. This gives
admins precise control over promotions (e.g., category discounts) or negotiated seller rates while
keeping a single global baseline.

#### Commission calculation & order locking

Commission is computed **per item** during checkout and stored on the order so it never changes
retroactively. The flow is:

1. Resolve the active rule (category → seller → global) or fall back to the plan commission rate
   (Free vs. Pro) from FeeConfig.
2. Calculate net unit price after any deal/discount.
3. Compute the per-unit fee using `max(netUnit * commissionPct, minimumFee)`.
4. Multiply by quantity to get the item’s `platformFee`.
5. Persist `commissionPct`, `commissionRuleId`, `commissionRuleScope`, `commissionMinFee`, and the
   order-level `commissionTotal`.

This locks the fee details on each order and supports reporting/analytics without recalculating fees
later. It also ensures minimum fee enforcement on **every unit**, not just per line item.

#### Pro Plan subscription purchase flow (Stripe Checkout)

Sellers can upgrade to the Pro plan via Stripe Checkout. The flow is:

1. Seller clicks **Upgrade to Pro** in the dashboard.
2. Backend calls `POST /api/seller/subscription/checkout`, resolves the **active FeeConfig
   `stripePriceId`** (or `STRIPE_PRO_PRICE_ID` fallback), and creates a **Stripe Checkout Session**
   in `subscription` mode.
3. The session stores `sellerId` in metadata and returns `session.id` + `session.url`.
4. After payment, the frontend calls `POST /api/seller/subscription/confirm` (or `/upgrade`) with
   the `sessionId`. The backend:
   - Verifies the session is complete and matches the seller.
   - Loads the Stripe subscription and payment method details.
   - Sets `subscriptionPlan = "pro"` and `planExpiresAt` using Stripe’s `current_period_end` (or a
     fallback based on the price interval).
   - Saves Stripe customer/subscription/payment method metadata on the seller.
   - Resets the subscription renewal tracking and applies Pro plan limits.

The result is a clean separation: **Stripe handles billing**, while the platform tracks plan state,
expiry, and limits in the Seller record for fast authorization checks across the product, order, and
analytics workflows.

#### Subscription renewal & expiration handling

Pro subscriptions auto-renew via a scheduled job:

- **Scheduler:** enabled by default, runs on a cron schedule (hourly by default), and performs an
  immediate run at startup. It can be disabled with `SUBSCRIPTION_RENEWAL_ENABLED=false`.
- **Lookahead window:** the job scans for Pro sellers with `planExpiresAt` within the configured
  lookahead days to ensure renewals happen before expiry.
  - **Cancel-at-period-end handling:** if a seller’s Stripe record has
    `payout.stripe.cancelAtPeriodEnd=true`, the renewal sweep will **not** attempt to renew. If the
  - **Expiration sweep:** in every cycle, the job first downgrades any Pro sellers whose
    `planExpiresAt` is in the past **and** who are marked cancel-at-period-end, so cancellations are
    enforced even if the seller never hits the subscription API.
- **Billing attempt:** uses the seller’s stored Stripe `customerId`, `paymentMethodId`, and
  `subscriptionId`. If no subscription exists, a new one is created using the active FeeConfig
  price.
- **Success path:** plan stays Pro, `planExpiresAt` is extended, and retry counters are reset.
- **Failure path:** retry attempts are scheduled with a configurable delay. After the final retry,
  the system schedules a downgrade (grace period) and emails the seller about billing failures.
- **Downgrade:** once the grace period passes, the seller is auto-downgraded to Free, expiry is
  cleared, and subscription limits are re-applied.

**Scheduled cron job settings (recommended defaults)**

```env
# Enable the auto-renewal system by default so Pro sellers auto-renew.
SUBSCRIPTION_RENEWAL_ENABLED=true
# Run hourly – frequent sweeps ensure renewals happen promptly.
SUBSCRIPTION_RENEWAL_CRON="0 * * * *"
# Use a neutral timezone like UTC to avoid DST issues.
SUBSCRIPTION_RENEWAL_TZ="UTC"
# Start attempting renewals 3 days before plan expiry (gives retry window).
SUBSCRIPTION_RENEWAL_LOOKAHEAD_DAYS=3
# Try up to 3 payment attempts before giving up.
SUBSCRIPTION_RENEWAL_MAX_RETRIES=3
# Wait 12 hours between retries (two retries per day).
SUBSCRIPTION_RENEWAL_RETRY_DELAY_HOURS=12
# If all retries fail, give a 1-day grace period before auto-downgrading.
SUBSCRIPTION_RENEWAL_DOWNGRADE_GRACE_HOURS=24
```

**Testing settings (temporary overrides)**

Use these for local testing when you want fast cycles and immediate retry windows:

```env
# Enable the auto-renewal system.
SUBSCRIPTION_RENEWAL_ENABLED=true
# Run every minute so changes are visible quickly.
SUBSCRIPTION_RENEWAL_CRON="* * * * *"
# Use a neutral timezone like UTC to avoid DST issues.
SUBSCRIPTION_RENEWAL_TZ="UTC"
# Only attempt renewals at/after expiry.
SUBSCRIPTION_RENEWAL_LOOKAHEAD_DAYS=0
# Keep retries low for faster downgrade validation.
SUBSCRIPTION_RENEWAL_MAX_RETRIES=2
# Wait 1 minute between retries (leave as-is for realism).
SUBSCRIPTION_RENEWAL_RETRY_DELAY_HOURS=0.016
# Short grace period for testing downgrade enforcement ~2minutes.
SUBSCRIPTION_RENEWAL_DOWNGRADE_GRACE_HOURS=0.032
```

Separately, the subscription read endpoint enforces expiration on demand: if a Pro seller’s
`planExpiresAt` is in the past, the API automatically moves them to the Free plan and clears any
pending cancellation flags. This ensures the UI and backend authorization stay consistent even if a
seller’s plan expires between renewal sweeps.

#### Admin/operations notes

- **Update fee defaults:** In the admin dashboard, open **Commission settings** and create/update a
  FeeConfig record with the Free/Pro commission %, minimum fee, and active date range.
- **Update Pro plan price:** Create a new **Stripe Price** in Stripe, then paste the new Price ID
  into FeeConfig. The next checkout/renewal will automatically use the updated price.
- **Rule overrides:** Use the same admin screen to create seller- or category-specific commission
  rules when you need targeted promotions or negotiated fee rates.

These changes are stored in the database and take effect immediately without requiring a deploy or
environment variable updates (except the optional Stripe price fallback).

## Subscription Billing Ledger

### Architecture & Design

#### 1) Separation from settlement payouts

Subscription billing ledger records are stored in a dedicated `SellerBilling` collection, not in
settlement payout entities (`SettlementBatch`, `SettlementPayout`) and not as payout ledger rows.
This keeps subscription invoice history conceptually and operationally separate from settlement
payout execution.

#### 2) End-to-end flow

1. Stripe sends `invoice.payment_succeeded` webhook event.
2. Webhook handler validates invoice identity and linkage (`customer` and/or `subscription`).
3. Handler resolves seller by Stripe linkage IDs.
4. Handler maps Stripe invoice into normalized billing row fields (`amountCents`, `currency`,
   `date`, `status`, `description`, `stripeInvoiceId`).
5. Service persists into `SellerBilling` via idempotent insert semantics.
6. Seller query API (`/api/seller/billing-history`) returns seller-only rows with pagination
   metadata.
7. Admin/staff query API (`/api/admin/platform-financials/billing-history`) returns global or
   seller-filtered rows.
8. Frontend seller dashboard `Billing History` tab fetches and renders rows with
   loading/empty/error/pagination states.

#### 3) Idempotency design (`stripeInvoiceId`)

- `stripeInvoiceId` is a required field and has a unique index.
- First delivery inserts the row.
- Replay delivery of the same invoice triggers duplicate key (`code 11000`) and is translated into a
  non-fatal duplicate outcome.
- Webhook still returns `200 { received: true }` for duplicates to avoid Stripe retry storms.

### Data Model Specification (`sellerBilling.model.js`)

Collection: `SellerBilling`

Fields:

- `sellerId` (`ObjectId`, required, indexed)
  - Reference: `Seller`
  - Must be present for every row.
- `amountCents` (`Number`, required)
  - Stored in minor units (cents).
  - Service normalizes values by numeric cast + rounding.
- `currency` (`String`, required, trimmed, uppercased)
  - Required currency code, normalized by schema/service.
- `date` (`Date`, required, default `Date.now`)
  - Invoice event date (or current time fallback).
- `stripeInvoiceId` (`String`, required, trimmed)
  - Canonical Stripe invoice identifier.
  - Global unique index enforces idempotency.
- `status` (`String`, required, enum)
  - Allowed: `draft`, `open`, `paid`, `void`, `uncollectible`, `payment_failed`.
- `description` (`String`, optional, trimmed, default `null`)
  - Human-readable invoice context.

Indexes:

1. `sellerBillingSchema.index({ sellerId: 1, date: -1, _id: -1 })`
   - Supports seller/admin history pagination with stable reverse-chronological sort.
2. `sellerBillingSchema.index({ stripeInvoiceId: 1 }, { unique: true })`
   - Enforces webhook idempotency by invoice ID.

Pagination and sorting approach:

- Query params `page` and `limit` are parsed as positive integers.
- `limit` is capped at `100`.
- Server computes `skip = (page - 1) * limit`.
- Sort order is deterministic: `{ date: -1, _id: -1 }` (stable even when timestamps tie).
- Response metadata includes `page`, `limit`, `total`, `totalPages`.

### API Specification

#### Seller billing history

- **Method/Path:** `GET /api/seller/billing-history`
- **AuthN/AuthZ:** authenticated seller route (`protectRoute` + `requireSeller`).
- **Query params:**
  - `page` (optional, positive int, default `1`)
  - `limit` (optional, positive int, default `20`, max `100`)
- **Success response (200):**

```json
{
  "items": [
    {
      "id": "<billing_row_id>",
      "sellerId": "<seller_id>",
      "recordType": "subscription_invoice",
      "amountCents": 2500,
      "currency": "USD",
      "date": "2026-01-01T10:00:00.000Z",
      "stripeInvoiceId": "in_001",
      "status": "paid",
      "billingStatus": "paid",
      "settlementPayoutStatus": null,
      "description": "Pro monthly subscription"
    }
  ],
  "page": 1,
  "limit": 20,
  "total": 17,
  "totalPages": 1
}
```

- **Error responses:**
  - `403 { "message": "Seller account required" }` when seller context is missing.
  - `500` delegated to error middleware for unexpected failures.

#### Admin billing history

- **Method/Path:** `GET /api/admin/platform-financials/billing-history`
- **AuthN/AuthZ:** authenticated `admin` or `staff` (`protectRoute` +
  `restrictTo('admin', 'staff')`).
- **Query params:**
  - `sellerId` (optional, ObjectId string)
  - `page` (optional, positive int, default `1`)
  - `limit` (optional, positive int, default `20`, max `100`)
- **Success response (200):** same schema as seller endpoint.
- **Error responses:**
  - `400 { "message": "Invalid sellerId" }` when provided sellerId is not a valid ObjectId.
  - `500` delegated to error middleware for unexpected failures.

### Webhook Handling Behavior

#### Event filtering and processing

- Only the Stripe `invoice.payment_succeeded` event path creates subscription billing rows in
  Task 15.
- Processing prerequisites:
  - valid invoice ID (`invoice.id`)
  - at least one seller-linkage identifier (`invoice.customer` or `invoice.subscription`)
  - seller lookup success
- If any prerequisite is missing, handler logs a warning and acknowledges with success
  (`{ received: true }`).

#### Duplicate replay behavior and idempotent outcomes

- Duplicate webhook deliveries for the same invoice reuse the same `stripeInvoiceId`.
- Unique index rejects second insert; service returns `{ recorded: false, duplicate: true }`.
- Handler logs informational duplicate message and still returns `200`, achieving idempotent side
  effects.

#### Failure handling and logging

- Non-fatal validation misses are logged as `warn` and acknowledged.
- Persistence/processing exceptions are logged as `error` with event and Stripe linkage context.
- Even processing failures currently acknowledge webhook receipt (`200`) to avoid uncontrolled retry
  loops; errors remain observable through logs.

### Frontend Design

#### Seller dashboard IA change

A dedicated seller-dashboard tab was added:

- Tab label: **Billing History**
- Panel: `SellerBillingHistoryPanel`
- Data source: `/api/seller/billing-history`

#### Billing History tab behavior

- Shows invoice-focused table columns: date, amount, currency, Stripe invoice ID, status badge,
  description.
- The subtitle explicitly communicates separation from payouts: invoices are “separately from
  settlement payouts.”

#### UI state handling

- **Loading:** initial skeleton/notice while first page is fetched.
- **Empty:** “No invoices found yet.”
- **Error:** visible inline error banner using API/store error message.
- **Pagination:** previous/next controls + `Page X of Y` summary using API metadata.

#### Explicit separation note

This tab is invoice history only; settlement payouts remain under the existing settlement/financial
views and are not merged into this table.

### Settlement Isolation Guarantees

- Settlement payout scheduling/execution calculations remain based on settlement ledger and payout
  entities, not `SellerBilling` rows.
- Subscription invoice rows contribute to liabilities visibility (`subscription_fees`) but are not
  transformed into settlement payout rows.
- Therefore, subscription billing records do **not** mutate settlement batch payable computation or
  payout transfer execution paths.

### Test Coverage Matrix

| Requirement                      | Concrete tests                                                                                                                                                                                                                                                                                                  |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Idempotency by `stripeInvoiceId` | `backend/src/tests/services/sellerBilling.service.spec.js` verifies duplicate-key handling (`code=11000`) returns `{ recorded:false, duplicate:true }`; `backend/src/tests/controllers/payment.webhook.controller.spec.js` verifies replay of same Stripe invoice is acknowledged and reprocessed idempotently. |
| Pagination correctness           | `backend/src/tests/controllers/billing.history.controller.spec.js` verifies deterministic pagination metadata and stable sort contract; `backend/src/tests/routes/billing-history.pagination.route.spec.js` verifies route-level `page/limit/total/totalPages` response behavior for seller/admin endpoints.    |
| Settlement balances unaffected   | `backend/src/tests/services/sellerBilling.service.spec.js` and liabilities integration in `backend/src/services/ledger/liabilities.service.js` keep subscription dues scoped to outstanding billing statuses only, leaving settlement payout status accounting untouched.                                       |
| Webhook integration behavior     | `backend/src/tests/controllers/payment.webhook.controller.spec.js` validates `invoice.payment_succeeded` ingestion, seller linkage checks, duplicate replay behavior, and missing-linkage acknowledgement semantics.                                                                                            |

### Backward Compatibility / Rollout Notes

#### Existing sellers with no billing records

- Seller billing endpoint returns empty `items` with deterministic metadata (`total=0`,
  `totalPages=1`) and frontend displays empty state.
- No data migration is required for historical sellers to continue operating.

#### Index rollout considerations

- `stripeInvoiceId` unique index must be created before relying on hard idempotency guarantees.
- If legacy duplicates somehow exist, index build will fail; deduplicate by invoice ID before
  enforcing uniqueness.
- Build index during low-traffic window if collection is large.

#### Post-deploy operational verification checklist

1. Confirm unique index exists on `stripeInvoiceId`.
2. Send/observe a real or staged `invoice.payment_succeeded` event and verify one `SellerBilling`
   row is created.
3. Replay same event and verify no second row is created; logs should indicate duplicate replay
   ignored.
4. Verify seller endpoint pagination and UI tab rendering for both non-empty and empty sellers.
5. Verify admin endpoint with/without `sellerId` filter.
6. Verify settlement payout scheduling/execution outputs are unchanged for same ledger period
   inputs.

### Mobile Category Tree v2 (dynamic categories)

- The mobile home screen now mirrors the dynamic category tree (same source of truth as web) when
  `EXPO_PUBLIC_FEATURE_MOBILE_CATEGORY_TREE_V2=true`.
- The flag is baked into native builds via `app.config.js` extras. After changing the flag or
  updating `app.config.js`, rebuild the **internal release APK** so the embedded config is
  refreshed.
- The Expo debug/dev client will pick up JS-only changes over Metro, but you only need to rebuild it
  if you want a new native binary with the updated `app.config.js` extras (e.g., for offline testing
  without Metro).

### DaisyUI theming (web + mobile)

The Tailwind/DaisyUI palettes live outside the native theme tokens so designers can iterate on
colours without disturbing existing light/dark logic. The shared catalogue is generated once and the
bridges in each client translate it into platform-friendly tokens, which is how we can flip the
entire colour system in minutes:

- [`docs/DAISYUI-THEME.md`](docs/DAISYUI-THEME.md) — explains the source-of-truth files and how to
  propagate updates across platforms.
- [`docs/THEME.md`](docs/THEME.md) — native token architecture (light/dark palettes, typography) for
  both web and mobile clients.
- `shared/theme/daisyThemes.js` — authoritative DaisyUI catalogue that the build scripts consume.
- `frontend/theme/styleThemes.js` — bridges the DaisyUI catalogue with the native tokens and exports
  helpers for Tailwind + the React context.
- `frontend/tailwind.config.js` — loads the DaisyUI palette list and passes it to the plugin at
  build time.
- `mobile/scripts/build-mobile-theme.mjs` — converts the shared catalogue into Expo-friendly
  JavaScript before Metro starts.
- `mobile/src/theme/generated/daisyThemes.js` — the generated mirror that NativeWind consumes.
- `mobile/src/theme/styleThemes.js` — mobile bridge that keeps DaisyUI logic separate from the
  native palette definitions.

Because the shared generators hydrate both clients, rebranding requires editing **one** DaisyUI
entry or the native `themeTokens.js` file, running `npm run gen:mobile-theme`, and restarting the
relevant dev servers. The storefront, dashboard, and Expo app inherit the new palette instantly
without manual CSS or component tweaks.

Each catalogue ships with the full set of official DaisyUI themes (35 as of v5.3) plus the bespoke
native palette. Every entry exposes both a light and dark variant so the navbar theme picker can
switch between **36** options without fighting the existing appearance toggle.

**Adding a new DaisyUI theme**

1. Duplicate an entry in `frontend/theme/daisyThemes.js` (or the mobile counterpart) within the
   `DAISY_STYLE_THEMES` array.
2. Pick lighter colours for the `light` object and darker tones for the `dark` object so each mode
   matches the palette intent.
3. Provide a unique `id`, `label`, and `description` to surface it inside the theme picker.
4. Copy the updated `daisyThemes.js` file between web and mobile if both apps should share the
   palette.
5. Restart `npm -w frontend run dev` (and/or `npm -w mobile run start`) so Tailwind/Metro reloads
   the config.

Run `npm install` once, then restart the relevant dev server (`npm -w frontend run dev` or
`npm -w mobile run start`) after editing a palette so Tailwind picks up the new configuration.

---

### Wishlist DTO & API contract

The wishlist flow is intentionally thin so the web and mobile clients can share the exact same DTO
and optimistic update logic:

- **Endpoint:** `POST /api/users/me/wishlist`
- **Body DTO:** `{ "productId": string }`
- **Success response:** `{ success: true, data: { ids: string[], products: WishlistProduct[] } }`

Key implementation files:

- Backend controller and DTO normalisation (`normalizeWishlistIds`, `normalizeWishlistProduct`):
  `backend/src/controllers/user.controller.js`
- Route registration: `backend/src/routes/user.routes.js`
- Frontend Zustand store that consumes the DTO and handles optimistic updates:
  `frontend/src/stores/useUserStore.js`
- Mobile Expo store mirroring the same contract: `mobile/src/stores/useUserStore.js`

All three surfaces share the same shape, so adding wishlist toggles or badges in any client only
requires UI work—the network contract and cached data stay in sync automatically.

### Public Store APIs

Use these endpoints for customer-facing storefront discovery and public seller identity surfaces.

#### `GET /api/public/store-details/:slug`

Returns full public storefront details for a specific store slug, including approved branding and
profile fields plus storefront policy content (for example banner assets, About/profile metadata,
and policy blocks used on public store pages and checkout policy display).

#### `GET /api/public/stores/:id`

Returns a minimal store summary payload intended for lightweight cards, references, and seller
identity snippets:

- `id`
- `slug`
- `name`
- `logoUrl`
- `description`

#### `GET /api/public/stores/search`

Primary endpoint for storefront and store discovery search.

**Query params**

- `q` — search keyword text
- `limit` — page size
- `page` — 1-based page index

**Response shape**

```json
{
  "data": ["stores"],
  "meta": {
    "page": 1,
    "limit": 12,
    "total": 120,
    "totalPages": 10
  }
}
```

#### Product payload note (`sellerSummary`)

Public product payloads now include `sellerSummary` so storefront, listing, and PDP experiences can
render public seller identity consistently without an additional lookup in common UI paths.

#### Frontend integration notes

- For store discovery and storefront search, use `GET /api/public/stores/search` (not legacy search
  routes/response conventions).
- Search response parsing should read:
  - `response.data.data` for stores
  - `response.data.meta.total` for total count
- This replaces legacy key paths such as `stores`, `results`, or `count`.

---

## Deployment surfaces & release workflow

### Frontend (Vercel)

- Project **Root Directory** must be `frontend`.
- `frontend/vercel.json` enforces `npm ci --legacy-peer-deps`, configures rewrites for `/api/*`, and
  applies SPA fallbacks + cache headers.

- Trigger deployments from CI or manually via CLI:

  ```bash
  npm -w frontend run build      # optional smoke build locally
  npx vercel link                # once per machine
  npx vercel --prod --yes        # production deploy (uses Vercel auth)
  ```

- Required environment variable:
  `PUBLIC_CLIENT_FALLBACK_URL=https://www.ahmedmonib-eshop-demo.com/reset-password`.

### Backend (Railway Docker image)

- Railway deploys the API using the root [`Dockerfile`](Dockerfile). It installs only the backend
  workspace and runs `backend/src/server.js` in production mode.
- Useful commands:

  ```bash
  docker build -t eshop-backend:latest .
  docker run --rm -p 5000:5000 -e NODE_ENV=production eshop-backend:latest
  ```

- Required Railway variables:

  | Key                          | Example                                                    | Notes                                   |
  | ---------------------------- | ---------------------------------------------------------- | --------------------------------------- |
  | `SESSION_SECRET`             | `base64:Vh1YhGq7...`                                       | Generate a 32+ byte random string.      |
  | `SESSION_REDIS_PREFIX`       | `session:`                                                 | Optional; keep consistent across envs.  |
  | `PUBLIC_CLIENT_FALLBACK_URL` | `https://www.ahmedmonib-eshop-demo.com/reset-password`     | Must match the frontend fallback.       |
  | `MOBILE_RESET_REDIRECT_URI`  | `eshop://reset-password`                                   | Deep link opened by reset emails.       |
  | `MOBILE_MAIL_CONFIRM_URI`    | `eshop://mailing/confirm`                                  | Deep link opened by mailing emails.     |
  | `MAIL_CONFIRM_WEB_URL`       | `https://shop.example.com/mailing/confirm?token={{token}}` | Overrides the browser confirmation URL. |
  | `MOBILE_OAUTH_REDIRECT_URI`  | `eshop://oauth`                                            | Deep link used by OAuth providers.      |

- Redeploy whenever you change env vars, email templates, or native deep-link paths.

### Mobile app (Expo custom dev client)

- Expo project lives in `mobile/` with a custom dev client that registers the `eshop://` scheme and
  Android intent filters.
- Install/update the dev client after native or deep-link changes:

  ```bash
  npm -w mobile run android     # builds & installs via Gradle
  npm -w mobile run env:tunnel  # copy HTTPS API profile for remote testing
  npm -w mobile run start:tunnel
  ```

- Deep-link smoke test:

  ```bash
  npx uri-scheme open "eshop://reset-password?token=TEST123&email=user%40example.com" --android
  ```

Refer to [`mobile/env.md`](mobile/env.md) for the full run-mode matrix and env profiles.

---

## Mobile deep-link flow overview

- Password reset emails include both a web fallback (`PUBLIC_CLIENT_FALLBACK_URL`) and a deep link
  (`MOBILE_RESET_REDIRECT_URI`).
- Android intent filters in `mobile/app.config.js` accept the `eshop://` scheme and route to the
  Expo app.
- Navigation prefixes: `['eshop://', 'exp+eshop-mobile://']` with screen config
  `{ ResetPassword: 'reset-password' }`.
- `extractResetParams` normalizes `token`/`code` query params and surfaces them as React Navigation
  route params.
- `ResetPassword` screen lives in both auth and authed stacks so the user always lands on the
  correct screen whether the app is cold-started or already in memory.
- Metro logs beginning with `🔗` confirm the linking handlers fired; re-run
  `npx uri-scheme open ...` to verify changes.

---

## Mobile app architecture & features

### Key capabilities

- **Shared domain logic:** Mobile consumes the same REST API as the web frontend through the
  `packages/api-client` workspace, reusing authentication flows, cart logic, and localization from
  the shared bundle.
- **Offline-friendly UX:** Secure tokens are cached in Expo SecureStore with background refresh
  logic so shoppers remain signed in across cold starts.
- **Full shopping journey:** The app surfaces the entire catalogue, promotional sliders, cart,
  checkout, order history, profile management (including GDPR data export and account deletion), and
  transactional policy screens.
- **Deep-link aware navigation:** Any email CTA (reset password, mailing confirmation, OAuth
  completion) can launch straight into the appropriate native screen thanks to the linking config
  described above.

### Native integrations

- **Expo custom dev client** with an Android `eas build` profile for shipping a signed release
  build.
- **Secure credential storage** through Expo SecureStore and Android keystore-backed encryption.
- **Splash screen & icons** managed in `mobile/app.config.js` with adaptive icon layers.
- **Over-the-air updates** remain available through Expo runtime while still allowing a Play Store
  binary release.

### Mobile documentation

All mobile-specific runbooks live alongside the app code:

- [`mobile/env.md`](mobile/env.md) — Environment matrix, API profiles (emulator/LAN/tunnel), and
  deployment checklist.
- [`mobile/custom-dev-setup.md`](mobile/custom-dev-setup.md) — Android Studio, Java, Gradle, and
  Expo prerequisites for a fresh machine.
- [`mobile/docs/mobile-checkout-flow.md`](mobile/docs/mobile-checkout-flow.md) — Annotated checkout
  sequence from product selection to confirmation screens.
- [`mobile/docs/android-release-play-store.md`](mobile/docs/android-release-play-store.md) — Release
  readiness checklist for generating an Android App Bundle and submitting to Play Console.
- [`mobile/docs/docs/mobile/android-build-config.md`](mobile/docs/docs/mobile/android-build-config.md)
  — Native build configuration (gradle.properties, signing files, `eas.json` notes).
- [`mobile/certs/README.md`](mobile/certs/README.md) — How signing certificates are stored and
  rotated during release.
- Expo CLI metadata (`mobile/.expo/`) is ephemeral and Git-ignored; regenerate locally via
  `expo start` when needed.

### Mobile screenshots

Representative UI captures (see [`docs/screenshots`](./docs/screenshots)):

- ![Mobile — Home light](./docs/screenshots/mobile-app-homepage-light.jpg) _Native home screen with
  featured sliders._

- ![Mobile — Deals & best sellers](./docs/screenshots/mobile-app-Deals,bestSellers-sliders-dark.jpg)
  _Dark mode promotional rails._
  ![Mobile — Category grid](./docs/screenshots/mobile-app-category-page-dark.jpg) _Category browsing
  with image tiles._
- ![Mobile — Cart (light)](./docs/screenshots/mobile-app-cart-page-light.jpg) _Cart summary with
  quantity controls._
- ![Mobile — Order history](./docs/screenshots/mobile-app-order-history-page-dark.jpg) _Past orders
  with status badges._
- ![Mobile — Profile & account tools](./docs/screenshots/mobile-app-profile-page-light.jpg) _Profile
  settings including export/delete controls._

---

## What this repo contains

- `frontend/` — React (Vite) app, i18n, Zustand stores, Stripe client, Sentry instrumentation.
- `backend/` — Express API (ESM), Mongoose models, Redis for token storage/caches/locks, Stripe
  webhook handlers, csurf, passport strategies, email helpers. Deployed with the root Dockerfile to
  Railway.
- `mobile/` — Expo + React Native custom dev client with deep-link integration for password reset
  and OAuth flows. See [`mobile/custom-dev-setup.md`](mobile/custom-dev-setup.md).
- `backend/src/tests/` + `backend/test/` — Jest unit/integration/E2E suites (supertest, fixtures,
  and in-memory Mongo harness for CI).
- `frontend/src/tests/` + `frontend/cypress/` — React Testing Library coverage and Cypress
  storefront flows.
- Root Dockerfile + Railway config for container-based API deployments.
- CI workflows for builds and Sentry releases (GitHub Actions).
- Full production-ready features: auth, cart, checkout, admin product & order management, order
  export & printing.

---

## Key features (summary)

- Authentication
  - Short-lived access tokens + rotating refresh tokens stored in Redis (per-device `jti`).
  - Sliding 30-day maximum session window (`session_start:<userId>`).
  - CSRF protection for state-changing endpoints and secure cookie configuration (`SameSite=None`,
    `Secure`, `HttpOnly`).
  - Mobile-aware keep-alive and activity detection to improve session persistence on phones.

- Payments & Orders
  - Stripe Checkout integration with PaymentIntent verification and robust webhook handling.
  - Cart backup and order fallback stored in Redis during checkout.
  - Stock reservation at checkout, atomic DB transactions for decrements, auto-restore on
    cancel/expire/refund.
  - Cash-on-delivery (COD) flow with the same reservation guarantees.

- Products & Inventory
  - Variant support (attributes, per-variant price, per-variant stock, variant images).
  - Admin product flows with variant generation and image management.

- Fulfilment & Admin

- Order listing, status changes, PDF export for fulfilment/labeling, canceled order labeling and
  audit trails.
- **Fully responsive Admin Dashboard** — manage orders, add products, edit variants, and run the
  store from a **phone or tablet** (mobile-friendly layouts and inputs).
- Privacy & Compliance

- GDPR-ready: user data export endpoint and account deletion/anonymization tooling.
- Observability & Security
  - Sentry client & server integration, Winston logging with daily rotation and PII redaction.
  - Rate limiting, input sanitization, NoSQL injection protection.

- Testing
  - Extensive unit/integration/E2E tests with 89.28% backend coverage.

---

## Payment & order flow (detailed)

This section explains the exact runtime flow implemented by the backend payment controller.

### Checkout (Stripe) flow — `createCheckoutSession`

1. **Request validation**
   - Backend receives `products`, `shippingAddress`, `billingAddress`, and optional `phoneNumbers`.
   - Validates addresses and that the cart is non-empty.

2. **Cart backup & fallback**
   - Saves a `cart_backup:<userId>` key in Redis (TTL) so the server can recover cart if the client
     drops off.
   - Creates an `order_fallback:<orderId>` Redis key after creating the order — used as a fallback
     to restore reservations if Stripe session expires.

3. **Order creation (DB transaction)**
   - Loads product metadata (name, price, deal, images, variants) from MongoDB.
   - Computes `subtotal`, `taxAmount` (TAX_RATE), `shippingCost` (SHIPPING_COST), and `totalAmount`.
   - Creates an `Order` document that includes per-item `variantAttributes` (stored as `Map` or
     similar) and persists it inside a Mongo transaction.

4. **Stock reservation**
   - For each order item, calls `reserveStock(productId, quantity, orderId, variantAttributes)`.
   - Reservation logic:
     - Writes a Redis reservation key `stock_reservation:<productId>:<orderId>:<variantKey>` with
       TTL (e.g., 30 minutes).
     - Starts a MongoDB session + transaction and decrements the product or matched variant stock
       atomically.
     - Commits transaction only if decrement succeeds; otherwise aborts and errors bubble back to
       the checkout flow.

5. **Stripe session creation**
   - Builds `line_items` with per-item unit amounts (variant pricing applied if present), plus tax
     and shipping line items.
   - Creates Stripe Checkout session with `metadata` containing `orderId` and `userId`, `expires_at`
     equals reservation TTL, and required `success_url` & `cancel_url`.
   - Updates the `Order` with `stripeSessionId` and `paymentIntentId` from Stripe, and stores order
     fallback info in Redis `order_fallback:<orderId>`.

6. **Commit & response**
   - If all steps succeed, commits the DB transaction and returns the Stripe session ID to the
     client for redirect.

**Result:** stock is reserved server-side, order exists in DB in `order_placed` (or
`cod_order_placed` for COD), and the frontend redirects the user to Stripe Checkout.

---

### Stripe webhooks & async handling — `stripeWebhook`

The webhook handler validates Stripe signatures and supports multiple events:

- **`checkout.session.expired`**
  - Processed asynchronously (background) using `handleExpiredSession(session)`.
  - Actions:
    - Acquire a Redis lock `lock:expire:<orderId>` to avoid race conditions.
    - Restore reserved stock for order by scanning `stock_reservation:*:<orderId>:*` and
      incrementing product/variant stock (via `restoreStockReservation`).
    - Delete the order (`Order.findByIdAndDelete(orderId)`) and clear safety keys (`cart_backup`,
      `order_fallback`).

  - This guarantees reserved inventory returns to stock if the checkout is abandoned.

- **`checkout.session.completed`**
  - Reads `session.metadata.orderId` and `userId`.
  - Clears backup safety keys.
  - Optionally fetches Stripe PaymentIntent to determine authoritative payment status (handles 3DS
    and other async flows).
  - If payment succeeded (`succeeded` / `paid`):
    - Performs optional final checks (extra stock verification).
    - Keeps paid card orders in `order_placed` (paid but awaiting fulfilment) — fulfillment status
      is separate.
    - Sends confirmation email if not already sent.

  - If a PaymentIntent indicates final failure (`canceled`, `failed`) then:
    - Marks order as `payment_failed`, restores stock, and sets failure reason.

- **`charge.refunded`**
  - If a full refund is detected for a payment intent related to an order:
    - Sets `order.status = 'refunded'`, persists `refundId`, updates timestamps.
    - Restores inventory via `restoreStockReservation(order._id)`.

> All webhook processing logs heavily and attempts safe recovery. Non-critical failures (e.g.,
> email) are logged but don't crash the handler.

---

### COD flow (Cash-on-Delivery) — `createCODOrder` & `codCheckoutSuccess`

- `createCODOrder` follows almost the same path as Stripe checkout: validate payload, compute
  totals, create an `Order` (status `cod_order_placed`), reserve stock (variant-aware), and return
  order id and totals to the client.
- `codCheckoutSuccess` accepts `orderId`, verifies user ownership, clears cart and safety keys,
  sends confirmation email (if not sent), and returns order summary to the client.

### COD eligibility policy, seller financial summary, and recovery path

COD checkout now enforces a seller-level **negative outstanding ledger balance policy** before a COD

- For each seller represented in the cart, backend computes:
  - current `outstandingBalanceCents` from pending ledger entries in policy currency.
  - projected commission impact for the attempted order.
  - projected post-order balance (`projectedBalanceCents`).
- COD is blocked when `projectedBalanceCents < COD_NEGATIVE_BALANCE_THRESHOLD_CENTS`.
- When blocked, checkout returns `409` and forces a **prepaid-only recovery path** (e.g., Stripe)
  until the seller balance recovers.
- Automatic re-enable is implicit: once a seller's outstanding balance improves above the threshold,
  subsequent COD checkout attempts pass again (no manual toggle required).

**New COD risk env vars**

- `COD_NEGATIVE_BALANCE_THRESHOLD_CENTS=-10000`
  - Default is `-10000` (−100.00 in minor units).
  - Parsing clamps to `<= 0` so positive values are treated as `0`.
- `COD_BALANCE_CURRENCY`
  - Policy currency for balance checks.
  - Fallback chain: `COD_BALANCE_CURRENCY` → `SETTLEMENT_CURRENCY` → `USD`.
  - Invalid/non-ISO-3 values also fall back to `USD`.
- `COD_BLOCK_NOTIFY_COOLDOWN_HOURS=24`
  - Cooldown window for repeat seller block-notification emails.
  - `0` disables cooldown (notification can send on every blocked attempt).

**API additions**

- COD checkout rejection payload:

  ```json
  {
    "error": "Cash on delivery is currently blocked for this seller.",
    "code": "COD_BLOCKED_NEGATIVE_BALANCE",
    "blockedSellers": [
      {
        "sellerId": "<sellerId>",
        "balanceCents": -15000,
        "projectedCommissionCents": 1200,
        "projectedBalanceCents": -16200
      }
    ]
  }
  ```

- Seller financial summary endpoint: `GET /api/seller/financials/summary`
- Returns current policy inputs/decision, reserve status, and seller-scoped liabilities summary:

```json
{
  "data": {
    "currency": "USD",
    "outstandingBalanceCents": -11000,
    "liabilitiesSummary": {
      "taxCents": 1300,
      "shippingCents": 450,
      "subscriptionFeesCents": 999,
      "totalLiabilitiesCents": 2749
    },
    "reserve": {
      "byCurrency": [
        {
          "currency": "USD",
          "balanceCents": 2500,
          "nextReleaseDate": "2026-04-01T00:00:00.000Z"
        }
      ],
      "percentage": 15,
      "tier": "new"
    },
    "codPolicy": {
      "negativeThresholdCents": -10000,
      "codAllowed": false,
      "blockedReason": "negative_balance_threshold"
    }
  }
}
```

### Return request + COD refund workflow

endpoints.

**Customer return request endpoint**

- `POST /api/orders/:orderId/return-request`
- Requires authenticated order owner (or admin).
- Validates:
  - order is currently `delivered`
  - order is within return window (`ORDER_RETURN_WINDOW_DAYS`, default 14)
  - prepaid orders must have `deliveryLedgerPostedAt`
  - `reasonKey` must map to an active `ReturnReason`
  - evidence upload is required when `requiresEvidence=true` on the reason
  - COD orders must include `codRefund.method` + `codRefund.details` matching required fields.
    Primary source: dynamic beneficiary rules endpoint
    (`GET /api/cod-refund-config/beneficiary-rules/:countryCode`). Fallback only: env-based
    `COD_BANK_FIELDS` / `COD_WALLET_FIELDS` when provider rules are unavailable.
- Side effects:
  - snapshots reason metadata into `order.returnRequest` (`label`, `refundShipping`,
    `requiresEvidence`, `evidenceTypes`)
  - stores audit actor metadata (`requestedAt`, `requestedByUserId`, `requestedByRole`)
  - updates `order.status = pending_refund`
  - for COD orders, applies seller payout hold (`seller.payoutHold=true`)

**Seller/Admin refund decision endpoints**

- Seller:
  - `POST /api/seller/orders/:id/refund/approve`
  - `POST /api/seller/orders/:id/refund/reject`
- Admin:
  - `POST /api/admin/orders/:id/refund/approve`
  - `POST /api/admin/orders/:id/refund/reject`
- Approve flow (COD):
  - ensures return request and non-expired payout details exist
  - computes refund plan from `refundPlan.service`
  - creates refund record (`provider='cod_payout'`, `status='requested'`)
  - executes payout adapter (`COD_PAYOUT_ADAPTER`; `COD_PAYOUT_PROVIDER` is still accepted as a
    fallback alias for backward compatibility)

For return reasons with `refundShipping=true` (for example, "Wrong item received" and "Defective
item"), COD approval computes refund components for merchandise, tax, and shipping. For reasons such
as "Buyer changed mind", shipping is excluded from the refund. COD payout submission in this flow
follows the current `/payouts/bank` contract (`banks[]` with `countryToSend`, `account_bank`,
`account_number`, `valueInUSD`, plus country-specific `fullname`/`account_name`) and does not use
legacy beneficiary payload shapes.

- emits structured provider lifecycle logs and monitoring counters (attempt/success/failure)
- on immediate provider success: marks refund record completed, updates order to `refunded`,
  releases payout hold
- on async provider result: keeps `pending_refund` while record is `in_progress`/`pending` until
  webhook or reconciliation converges it
- on provider failure: marks refund record failed, keeps order in `pending_refund`, and keeps payout
  hold active

- moves order to `refund_rejected`
- releases payout hold releases payout hold
- no rejection ledger rows are created

**Security & retention for COD payout details**

- COD payout details are stored encrypted (`codRefund.detailsEncrypted`) with masked summary fields
  (`detailsLast4`, `detailsSummary`).
- Decryption supports key-rotation fallback (`COD_REFUND_DETAILS_ENCRYPTION_KEYS`).
- Access to decrypted data is audit-logged to `CodRefundAccessAudit`.
- Encrypted details are purged once stable beneficiary metadata and `detailsLast4` are persisted.
- On refund completion, details expiry is refreshed from completion timestamp
  (`refundCompletedAt + COD_REFUND_DETAILS_TTL_DAYS`).
- A scheduled purge job clears expired encrypted details while retaining redacted summary metadata.

**COD payout reconciliation cron (stuck payout detector)**

- Purpose: detect stale COD payouts, poll provider status, auto-complete safe cases, and flag manual
  review failures.
- Local/dev recommended values:
  - `COD_PAYOUT_RECONCILIATION_ENABLED=true`
  - `COD_PAYOUT_RECONCILIATION_CRON=*/1 * * * *`
  - `COD_PAYOUT_RECONCILIATION_TIMEZONE=UTC`
  - `COD_PAYOUT_RECONCILIATION_AGE_MINUTES=1`
  - `CHIMONEY_PAYOUT_STATUS_PATH=/payouts/{payoutId}`
- Production safe values:
  - `COD_PAYOUT_RECONCILIATION_ENABLED=true`
  - `COD_PAYOUT_RECONCILIATION_CRON=*/20 * * * *`
  - `COD_PAYOUT_RECONCILIATION_TIMEZONE=UTC`
  - `COD_PAYOUT_RECONCILIATION_AGE_MINUTES=45`
  - `CHIMONEY_PAYOUT_STATUS_PATH=/payouts/{payoutId}`
- Quick test: create stale in-progress COD refund record, run job, verify either `completed` or
  `reconciliation_*` failure code and matching logs.

**Stuck refund detection + safe retry**

1. Find refund by `orderId`, `idempotencyKey`, and `providerRefundId`.
2. Confirm provider-side status before any retry.
3. If provider succeeded, replay webhook or run reconciliation (avoid new payout request).
4. If provider failed, keep payout hold active and create a new approval attempt with a new
   idempotency key after fixing beneficiary data.

**API key / webhook secret rotation**

1. Generate new `CHIMONEY_API_KEY` and `CHIMONEY_WEBHOOK_SECRET`.
2. Update secrets and deploy.
3. Replay a recent webhook to validate verification and processing.
4. Remove old credentials and monitor for 401 webhook failures.

**Webhook replay instructions (admin/on-call)**

1. Re-send original payload to `POST /api/chimoney/webhook`.
2. Pass `x-chimoney-webhook-secret` with active secret.
3. Confirm response has `received: true`.
4. Verify matching via idempotency key (fallback provider refund id), and confirm order transition.

### Provider architecture & adapter contract (COD payouts)

This project uses a provider-router + adapter pattern for COD payout execution:

- **Router service:** `backend/src/services/payments/codPayoutProvider.service.js`
  - Reads active adapter id from `COD_PAYOUT_ADAPTER`.
  - Backward compatibility: falls back to `COD_PAYOUT_PROVIDER` when `COD_PAYOUT_ADAPTER` is unset.
  - Selects adapter from `COD_PAYOUT_ADAPTERS`.
  - Enforces normalized output shape (`ok`, `status`, `provider`, `idempotencyKey`, optional
    `providerRefundId`, `errorCode`, `errorMessage`, `metadata`).
  - Emits provider selection and outcome metrics/logs.
- **Adapter registry:** `backend/src/services/payments/providers/index.js`
  - Registers named adapters (`chimoney`, `manual`, `disabled`).
  - Ensures adapter return statuses map to allowed statuses:
    - `pending`
    - `completed`
    - `failed`
- **Provider adapter implementation:** `backend/src/services/payments/providers/chimoney.adapter.js`
  - Handles provider-specific HTTP calls, retries, validation, quote handling, beneficiary reuse,
    idempotent payout submission, and response mapping.

#### Adapter input contract

All COD payout adapters are called with:

```ts
{
  order: OrderDocumentLike,
  amountCents: number,
  method: string,               // currently bank_transfer
  details: Record<string, any>, // decrypted payout details
  idempotencyKey: string
}
```

For `method='bank_transfer'`, adapters should expect and validate these `details` keys before
provider calls:

- `countryToSend` (destination country code used to derive provider payout rails/rules).
- `account_bank` (provider bank identifier / bank code).
- `account_number` (beneficiary account number).
- Payer/account holder name key (country-specific rule):
  - `fullname` is required for US (`countryToSend='US'`).
  - `account_name` is required for NG (`countryToSend='NG'`).

`valueInUSD` is the canonical payout value sent to the provider request builder. COD adapters do
**not** send a root-level `currency` field in the payout body; FX/currency handling is delegated to
provider-specific quote/bank routing logic that derives from destination/bank context.

For bank payout submission, adapters should send payloads shaped like:

```ts
{
  banks: [
    {
      // bankPayout
      id: string,
      countryToSend: string,
      account_bank: string,
      account_number: string,
      valueInUSD: number,
      // plus country-specific name key: fullname (US) or account_name (NG)
    },
  ];
}
```

This payload is posted to `/payouts/bank`.

#### Adapter output contract

All adapters must return normalized results:

```ts
{
  ok: boolean,
  status: 'pending' | 'completed' | 'failed',
  providerRefundId?: string,
  errorCode?: string,
  errorMessage?: string,
  metadata?: {
    rawResponse?: Record<string, any>
    [key: string]: any
  }
}
```

Normalization rules:

- `status` must always be one of `completed | pending | failed` (provider-native states are mapped
  into this set).
- `providerRefundId` should carry the provider's canonical transfer/payout identifier whenever
  available.
- `metadata.rawResponse` should retain a sanitized provider response envelope for investigations,
  replay debugging, and reconciliation support.

#### Error handling notes (adapter layer)

- Use `redactPii(...)` before persisting/logging provider request/response fragments.
- Parse provider error payloads defensively (nested/message/code variants) and map to stable
  `errorCode` + `errorMessage` values exposed by the normalized adapter contract.
- When provider parsing is ambiguous, keep the best-effort normalized fields but include sanitized
  `metadata.rawResponse` to preserve forensic context without leaking sensitive details.

#### Design goals

The COD provider adapter layer is designed around these goals:

1. **Provider-agnostic orchestration** Order/refund workflow code should not depend on
   Chimoney-specific payload formats.
2. **Deterministic idempotency** A single business refund attempt should map to a stable idempotency
   key and never create duplicate payout side effects during retries/replays.
3. **Auditability & operability** Every critical transition should be traceable through structured
   logs + metrics using `orderId`, `idempotencyKey`, `providerRefundId`, and `refundRecordId`.
4. **Progressive consistency** In-flight payouts may remain `pending`/`in_progress` and later
   converge via webhook or reconciliation cron without forcing unsafe synchronous assumptions.
5. **Least retention of sensitive data** Encrypted payout details are retained only while necessary
   and purged once stable provider identity metadata is present.

#### Runtime architecture (modules and responsibilities)

- `services/refunds/codRefundWorkflow.service.js`
  - orchestrates approval execution lifecycle,
  - manages refund record transitions (`requested → in_progress → completed|failed`),
  - coordinates seller payout hold release and order transition.
- `services/payments/codPayoutProvider.service.js`
  - provider router + result normalizer,
  - emits provider-selection and provider-outcome observability.
- `services/payments/providers/chimoney.adapter.js`
  - Chimoney-specific implementation (validation, beneficiary, quote, payout request, retry,
    idempotency replay mapping).
- `services/payments/chimoneyWebhook.service.js`
  - authoritative async status convergence from provider callbacks.
- `jobs/codPayoutReconciliation.js`
  - stale in-progress recovery path when webhook flow is delayed/missed.
- `utils/codRefund.js`
  - secure detail encryption/decryption, validation enrichment, and persistence helpers.

#### End-to-end lifecycle (happy path)

1. Customer submits return request with COD payout destination details.
2. Details are encrypted and stored in `order.codRefund.detailsEncrypted` plus masked metadata.
3. Admin/seller approves refund:
   - workflow computes refund plan and marks matching record `in_progress`,
   - calls provider router with deterministic idempotency key.
4. Adapter sends payout request to provider and returns normalized result.
5. Webhook / reconciliation finalizes provider status:
   - updates refund record + order status,
   - releases seller payout hold transactionally,
   - triggers accounting/order financial hooks.
6. Encrypted details are purged once stable beneficiary metadata + last4 are available.

#### COD refund lifecycle (state-focused)

1. Return request sets `order.status='pending_refund'` and appends a `refundRecords[]` row with:
   - `provider='cod_payout'` (fixed internal category for COD payouts),
   - `status='requested'`.
2. Approval transitions that record to execution:
   - usually `status='in_progress'`, `providerStatus='in_progress'` at dispatch time,
   - then provider-normalized `providerStatus` may be `pending`, `completed`, or `failed`.
3. Finalization:
   - success path converges to `refundRecords[].status='completed'` and `order.status='refunded'`,
   - failed provider path keeps `order.status='pending_refund'` for manual review/remediation.

#### State model and transitions

**Refund record statuses**

- `requested` → created/queued for execution.
- `in_progress` → execution started; awaiting terminal provider state.
- `completed` → provider terminal success.
- `failed` → provider terminal failure/manual review required.

**Provider statuses (normalized)**

- `pending`
- `completed`
- `failed`

**Order statuses in COD refund context**

- `pending_refund` while unresolved
- `refunded` on terminal success
- remains/returns `pending_refund` on failure paths requiring intervention

#### Data model mapping (what lives where)

- `order.refundRecords[]`
  - per-attempt immutable history and transition metadata
  - key fields: `provider`, `idempotencyKey`, `providerRefundId`, `providerStatus`, `failureCode`,
    `failureMessage`, `providerMetadata`
- `order.codRefund`
  - destination method + encrypted payload + masked summary
  - security fields: `detailsEncrypted`, `detailsLast4`, `detailsSummary`, `detailsExpiresAt`
- `seller.payoutHold*`
  - temporary hold controls while refund payout is unresolved

#### Failure handling strategy

- **Synchronous provider failure** (adapter returns `ok=false`)
  - mark record failed with failure metadata
  - keep payout hold active
- **Webhook missing or delayed**
  - reconciliation cron scans stale `in_progress` records
  - attempts provider status lookup and converges to completed/manual-review failure
- **Idempotent replay**
  - duplicate execution attempts short-circuit by record state + idempotency key checks
  - provider `409` conflict is mapped to replay, not treated as a second payout
- **Unknown provider status**
  - normalized to `pending` to avoid premature terminal failure assumptions

#### Transaction boundaries and consistency model

The system uses DB session/transaction boundaries in terminal completion paths where these updates
must be consistent together:

- order/refund record terminal status update
- seller payout hold release

This avoids partial updates such as “order marked refunded but hold not released” (or vice versa).

#### Observability model

Metrics emitted (via monitoring abstraction):

- provider selection attempt/failure
- provider HTTP attempt/failure
- execution success/failure
- webhook receive/process success/failure
- reconciliation attempt/success/failure
- reconciliation pending-age bucket counters

Structured logs include correlation keys:

- `orderId`
- `idempotencyKey`
- `providerRefundId`
- `refundRecordId`

#### Runtime tuning envs and design impact

These env vars tune adapter/reconciliation runtime behavior without changing business state-machine
logic (`requested → in_progress → completed|failed`): the architecture remains router
(`codPayoutProvider`) → provider adapter (`chimoney.adapter`) → async convergence via
webhook/reconciliation. Field requirements follow **dynamic provider beneficiary rules first**, with
env-based `COD_BANK_FIELDS` / `COD_WALLET_FIELDS` as fallback when dynamic rules are unavailable.

- `CHIMONEY_TIMEOUT_MS`
  - design impact: limits per-request provider wait budget before failing fast.
- `CHIMONEY_MAX_RETRIES`
  - design impact: controls retry depth for retryable provider/network failures.
- `CHIMONEY_RETRY_BASE_DELAY_MS`
  - design impact: controls retry pacing/backoff pressure on provider/API.
- `CHIMONEY_SOURCE_CURRENCY`
  - design impact: overrides quote source currency when provider quote generation requires explicit
    source-currency control.
- `COD_REFUND_CHIMONEY_WEBHOOK_STORE_RAW_DEBUG`
  - design impact: diagnostic visibility toggle; should remain disabled in production except short
    incident windows due to sensitive payload retention risk.

#### Security model

- COD payout details encrypted at rest.
- Key-rotation supported via primary + fallback key list.
- Decryption access is audit logged.
- Encrypted payload purged when no longer required for processing.
- Provider metadata is redacted by default:
  - no raw account numbers are stored,
  - no raw webhook body is persisted unless explicit debug flag is enabled.

### Extending to a new provider (implementation checklist)

1. Add adapter module under `services/payments/providers/`.
2. Implement required input/output contract and status normalization.
3. Register adapter in provider index map.
4. Ensure idempotency header/body mapping for provider API.
5. Emit structured logs + monitoring metrics with standard correlation IDs.
6. Add tests:
   - success/failure mapping,
   - retry and replay behavior,
   - idempotency/duplicate handling.
7. Update README env vars + runbook sections for provider-specific setup.

#### Idempotency behavior (safe retries)

- A refund record carries `idempotencyKey`.
- Approval short-circuits if record is already in-progress/completed or already has
  `providerRefundId`.
- Chimoney payout requests forward idempotency via:
  - HTTP header: `Idempotency-Key`
  - payload reference field: `reference`
- If Chimoney responds with `409` duplicate-idempotency conflict, adapter treats it as replay and
  maps to a non-fatal response with provider metadata (`replay: true`).
- **Operational rule:** retry using the **same** idempotency key for the same business attempt; only
  generate a new key when intentionally creating a new payout attempt after confirmed failure.

#### How to detect stuck refunds / payouts

Use all three signals together:

1. **Order/refund state**
   - `order.status = pending_refund`
   - latest `refundRecords[].provider = cod_payout`
   - latest `refundRecords[].providerStatus = in_progress` or `pending`
2. **Age threshold**
   - record `updatedAt` older than `COD_PAYOUT_RECONCILIATION_AGE_MINUTES`
3. **Reconciliation outputs**
   - manual-review failure codes:
     - `reconciliation_manual_review_required`
     - `reconciliation_unresolved_manual_review`
     - `reconciliation_status_fetch_failed`

Also inspect structured logs with `orderId`, `idempotencyKey`, `providerRefundId`, `refundRecordId`.

#### Safe stuck-refund retry procedure (runbook condensed)

1. Find the current attempt by `idempotencyKey` and `providerRefundId`.
2. Confirm provider-side truth (dashboard/API/webhook deliveries).
3. If provider already succeeded:
   - replay webhook event OR run reconciliation cycle,
   - do **not** create a second payout request.
4. If provider definitively failed:
   - keep seller payout hold active until resolved,
   - correct beneficiary/bank payload issue,
   - create a new attempt with a new idempotency key.
5. Verify final order transition + seller hold release + refund record terminal state.

#### Chimoney sandbox/config expectations

- `CHIMONEY_BASE_URL` must point to the correct environment (use `https://api-sandbox.chimoney.io`
  for testing/sandbox, and production host only in production).
- `CHIMONEY_API_KEY` must match the same environment as `CHIMONEY_BASE_URL`.
- Chimoney API calls use `X-API-KEY: <CHIMONEY_API_KEY>` header authentication.
- `CHIMONEY_WEBHOOK_SECRET` is required for secure webhook verification.
- `CHIMONEY_PAYOUT_STATUS_PATH` controls reconciliation lookup endpoint and should include
  `{payoutId}` token.

#### Chimoney webhook endpoint setup (sandbox dashboard)

When creating the webhook endpoint in Chimoney dashboard, use your backend public URL with:

- Preferred endpoint URL:
  - `https://<your-backend-domain>/api/chimoney/webhook`
- Optional path-secret style endpoint URL (if you prefer URL secret matching):
  - `https://<your-backend-domain>/api/chimoney/webhook/<CHIMONEY_WEBHOOK_SECRET>`

Both routes are supported by backend routing:

- `POST /api/chimoney/webhook`
- `POST /api/chimoney/webhook/:webhookSecret`

Subscribe to these events (the ones currently processed by backend):

- `payout.bank.initiated`
- `payout.bank.completed`
- `payout.bank.failed`

If extra events are sent, they are accepted but ignored as `unsupported_event_type`.

For local testing via tunnel (ngrok/Cloudflare Tunnel), set endpoint URL to your temporary HTTPS URL
plus `/api/chimoney/webhook`, then replay a sample event and confirm `received: true`.

Example sandbox-style config:

```env
COD_PAYOUT_ADAPTER=chimoney
# Backward compatibility: COD_PAYOUT_PROVIDER is still accepted as fallback if ADAPTER is unset.
CHIMONEY_BASE_URL=https://api-sandbox.chimoney.io
CHIMONEY_API_KEY=chimoney_sandbox_key
CHIMONEY_WEBHOOK_SECRET=chimoney_sandbox_webhook_secret
CHIMONEY_PAYOUT_STATUS_PATH=/payouts/{payoutId}
```

### Cron jobs added for COD payout resiliency

#### 1) COD payout reconciliation cron

- **Scheduler env vars**
  - `COD_PAYOUT_RECONCILIATION_ENABLED`
  - `COD_PAYOUT_RECONCILIATION_CRON`
  - `COD_PAYOUT_RECONCILIATION_TIMEZONE`
  - `COD_PAYOUT_RECONCILIATION_AGE_MINUTES`
- **Purpose**
  - Finds stale in-progress COD payout records.
  - Fetches provider payout status.
  - Auto-completes safe terminal-success cases.
  - Marks unresolved/error cases for manual review.
- **How to test**
  1. Create COD refund in progress.
  2. Set record `updatedAt` old enough to be stale.
  3. Run cycle manually or wait for cron trigger.
  4. Verify transition + metrics/log output + failure code or completion.

#### 2) COD refund details purge cron

- **Scheduler env vars**
  - `COD_REFUND_PURGE_CRON`
  - `COD_REFUND_PURGE_TZ`
- **Purpose**
  - Purges expired encrypted COD payout payloads.
  - Preserves redacted metadata (`detailsSummary`, `detailsLast4`) for support trails.
- **How to test**
  1. Seed order with `codRefund.detailsEncrypted` set and expired `detailsExpiresAt`.
  2. Run purge job.
  3. Verify encrypted payload is null and status marked expired while summaries remain.

### Key rotation & webhook replay procedures

#### Rotate Chimoney API key safely

1. Create new API key in Chimoney console (same environment).
2. Update secret manager (`CHIMONEY_API_KEY`).
3. Deploy and validate outbound provider calls.
4. Reconcile one in-progress payout to ensure status lookup still works.
5. Revoke old key after verification window.

#### Rotate Chimoney webhook secret safely

1. Generate new webhook secret in Chimoney.
2. Update `CHIMONEY_WEBHOOK_SECRET`.
3. Replay a known webhook payload against `/api/chimoney/webhook`.
4. Confirm request is accepted and matched to refund record.
5. Remove old webhook secret in provider console.

#### Replay webhook (admin/on-call detailed)

1. Capture original payload + event metadata from provider dashboard.
2. `POST /api/chimoney/webhook` with payload and `x-chimoney-webhook-secret: <active-secret>`.
3. Confirm 2xx response and `received: true`.
4. Validate matching path:
   - first by `idempotencyKey`
   - fallback by `providerRefundId`
5. Confirm order/refund transitions and seller payout-hold state.

#### Troubleshooting: stuck pending refunds

If an order stays in `pending_refund` longer than expected:

1. Confirm webhook delivery and matching keys (`idempotencyKey` first, then `providerRefundId`).
2. Run/verify COD payout reconciliation for stale `in_progress` records.
3. If still unresolved, keep payout hold active and perform manual review before any new payout
   attempt.

#### COD refund record API example (sanitized fields)

```json
{
  "provider": "cod_payout",
  "providerName": "chimoney",
  "providerStatus": "pending",
  "status": "in_progress",
  "idempotencyKey": "cod_refund:orderId:recordIndex:1299",
  "providerRefundId": "pay_123",
  "providerMetadata": {
    "chimoney": {
      "eventType": "payout.bank.initiated",
      "providerRefundId": "pay_123",
      "status": "pending",
      "idempotencyKey": "cod_refund:orderId:recordIndex:1299",
      "bank_name": "Example Bank",
      "account_last4": "1234"
    },
    "payload": {
      "payoutId": "pay_123",
      "status": "INPROCESS",
      "reference": "cod_refund:orderId:recordIndex:1299"
    }
  }
}
```

**Settlement hold enforcement**

- Settlement scheduler excludes sellers with `payoutHold=true` from scheduled payouts until the
  related refund is resolved.

- Seller liabilities endpoint: `GET /api/seller/financials/liabilities`
  - Seller-authenticated dashboard endpoint for liability-only payloads.

  ```json
  {
    "data": {
      "currency": "USD",
      "liabilitiesSummary": {
        "taxCents": 1300,
        "shippingCents": 450,
        "subscriptionFeesCents": 999,
        "totalLiabilitiesCents": 2749
      }
    }
  }
  ```

- Liability-account env vars used by ledger mapping:
  - `PLATFORM_SELLER_ID`: seller id of the platform liability ledger account (tax flows).
  - `SHIPPING_SELLER_ID`: seller id of the shipping liability ledger account (shipping flows).
    **Operational notes**

- Notification cooldown behavior:
  - On first blocked attempt per seller, backend stores a cooldown key and sends email.
  - During cooldown window, additional blocked attempts suppress repeat notifications.
  - After TTL expiry, the next blocked attempt can notify again.
- Monitoring/debug tips for blocked sellers:
  - Query seller summary endpoint to confirm live balance/currency/threshold decision.
  - Compare `blockedSellers[].projectedBalanceCents` vs threshold in checkout 409 responses.
  - Verify env alignment (`COD_BALANCE_CURRENCY`, threshold) across deploy targets.
  - Check cache key TTLs (`cod_block_notice:<sellerId>`) when investigating missing/duplicate
    notifications.

---

### Stock reservation & restoration (variant-aware)

- Reservations are stored as Redis keys with structure:

  ```
  stock_reservation:<productId>:<orderId>:<variantKey>
  ```

  where `<variantKey>` is a stable JSON string of variant attributes (so attribute order doesn't
  break matching).

- `reserveStock`:
  - Saves Redis reservation (TTL \~ 30 min).
  - Opens Mongo transaction and decrements either `product.stock` or `variants.$.stock` for the
    matched variant.

- `restoreStockReservation`:
  - `KEYS stock_reservation:*:<orderId>:*` (production should use `SCAN` for large keyspaces) to
    find reservations.
  - Builds `bulkWrite` ops to increment product or variant stock back by reservation quantity, and
    deletes reservation keys after update.
  - Works with variant attribute matching via `arrayFilters` in Mongo bulk ops.

---

### Failure & expired session handling

- If any step fails before Stripe session creation, the transaction is aborted and the client
  receives a `400`/`500` error (with friendly messages).
- If Stripe session expires without payment:
  - Webhook triggers `handleExpiredSession`, which restores stock, deletes the incomplete order, and
    clears safety keys.
  - Because reservations use Redis keys and the DB update is atomic, this avoids double-selling.

- If a PaymentIntent partially or fully fails after session completion:
  - PaymentIntent inspection in the webhook marks `payment_failed` and restores inventory if
    appropriate.

---

### Seller settlements, ledger lifecycle, and payout execution

The payout system is implemented as a **double-entry style operational ledger** at seller level:

**Ledger dimensions (authoritative contract)**

- `accountingScope`: `merchant | platform`
  - `merchant`: merchant-facing economic activity (sale/commission/refund/tax/shipping/etc.)
  - `platform`: marketplace-operator activity (platform fee, processor fee, recovery, disbursement
    mirrors)
- `entrySide`: `merchant | operator`
  - `merchant`: merchant-side row
  - `operator`: operator-side mirror/offset row
- `merchantSellerId`: merchant identity anchor when `sellerId` is a platform/operator ledger account
  (required for platform-store and mirrored operator flows)
- `counterpartyRole`: `merchant | platform | shipping | processor` (required for fee/recovery/
  liability rows; may be `null` only for generic manual adjustments)

These fields allow deterministic filtering for platform-store flows where merchant and operator rows
coexist for related events.

**Field-level schema notes**

- `LedgerEntry.accountingScope` (`merchant | platform`)
  - Required on writes; drives admin scope switch behavior and seller dashboard merchant-only
    totals.
- `LedgerEntry.entrySide` (`merchant | operator`)
  - Required on writes; distinguishes business-facing rows from platform offset rows.
- `LedgerEntry.merchantSellerId` (`ObjectId|null`)
  - Required when `sellerId` points to a platform/operator account but the row belongs to a specific
    merchant for UI and aggregation rollups.
- `LedgerEntry.counterpartyRole` (`merchant | platform | shipping | processor | null`)
  - Required for fee/recovery/liability diagnostics; `null` is reserved for generic adjustments.
- Reserve fields on `LedgerEntry` (used by reserve APIs and jobs):
  - `reserveMonth` (`YYYY-MM`) — accounting bucket for grouped release decisions.
  - `withheldAt` (`ISO datetime`) — when reserve was originally held.
  - `releaseScheduledAt` (`ISO datetime`) — earliest eligible release timestamp.
  - `metadata.tier`, `metadata.percentage`, `metadata.reason`, `metadata.actor` — policy intent +
    audit context required by admin reserves UI.
- Order/COD refund fields now treated as UI contract fields:
  - `order.codRefund.status` (`pending | processing | completed | failed | expired`) for state
    chips.
  - `order.codRefund.failureCode` / `order.codRefund.failureMessage` for actionable support copy.
  - `order.codRefund.providerStatus`, `order.codRefund.providerRefundId` for reconciliation
    drilldown.
  - `order.codRefund.detailsSummary`, `order.codRefund.detailsLast4` for redacted evidence after
    encrypted payload purge.

**Idempotency + replay protection**

- Ledger deduplication is `idempotencyKey`-centric.
- Ledger writes rely on `idempotencyKey` as the single replay-protection contract across merchant
  and operator scoped rows.

**Outstanding-balance scoping behavior**

Outstanding balance services accept optional:

- `referenceOrderIds` to constrain aggregation to specific orders
- `scope` (`merchant | platform | all`) to constrain by accounting scope

This is used by platform-store seller dashboard flows so merchant-only balances are computed from
platform-owned merchant rows.

**Processor-fee recovery (canonical behavior)**

- Canonical baseline key: `processor_fee_recovery:<orderId>`
- If later events report a different recovered amount, the service writes delta rows using
  `processor_fee_recovery:<orderId>:delta:<targetAmountCents>`
- Exact same-amount replay is a no-op

**Delivery ledger posting behavior (prepaid + COD)**

- Delivery posting is idempotent using deterministic keys and `deliveryLedgerPostedAt`.
- Prepaid paths write merchant rows and operator mirror rows with explicit scope/side metadata.
- COD paths write seller-side delivery rows plus COD reimbursement/disbursement/refund rows.
- Successful posting stamps `deliveryLedgerPostedAt`, which is also used by refund/eligibility
  logic.

- Every sale/refund/fee action emits immutable `LedgerEntry` rows (`sale`, `commission`, `refund`,
  `commission_reversal`, `adjustment`) with integer `amountCents`, currency, and idempotent
  references to the originating order/refund path.
- Seller totals are **netted across all currently pending ledger rows in the eligible period**.
- Scheduler eligibility includes **all pending entries with `createdAt <= periodEnd`** (not just
  rows created inside the current window), so prior non-paid rows carry forward automatically until
  included in a positive-net schedule.
- If a seller's net is non-positive for that run, no payout row is created for that seller and those
  ledger rows remain `payoutStatus=pending` (carry-forward debt/offset behavior).
- When the same seller later has enough positive entries to make the net amount positive, scheduler
  creates a payout only for the positive net result and then marks only included rows as
  `payoutStatus=scheduled` with `settlementBatchId`.
- Once payout execution succeeds, scheduled rows move to `paid`; pending carry-forward rows remain
  pending until included by a future positive-net schedule.

**Settlement lifecycle**

1. Scheduler computes the next period window (`SETTLEMENT_PERIOD_DAYS`).
2. Eligibility is checked against period end + hold days (`SETTLEMENT_HOLD_DAYS`).
3. A `SettlementBatch` is created with seller totals and entry count.
4. One `SettlementPayout` row is created per seller for positive net amounts.
5. If `SETTLEMENT_AUTO_EXECUTE=true`, the cron cycle immediately executes the new batch.
6. Admin can always execute manually via `POST /api/admin/settlements/:id/execute` (manual override
   path remains available even when auto-execute is on).
7. Manual/admin schedule trigger dedupe behavior is explicit and non-erroring:
   - Existing batch for same `(periodStart, periodEnd, currency)` may return
     `created: false, deduplicated: true`.
   - Duplicate-key race safety returns `created: false, reason: 'duplicate'`.
8. Cron logs also report duplicate/no-op outcomes as `No settlement batch scheduled by cron` with
   `reason` and `deduplicated` metadata for operator visibility.
9. Payout outcome handling is explicit:
   - Stripe transfer success => payout `paid`, ledger rows `paid`.
   - Missing prerequisites / non-positive amount => payout `manual_required`.
   - Non-positive payouts are never sent to Stripe transfer creation; they are short-circuited to
     manual handling.

- Stripe API / balance failures => payout `failed`.

**Batch statuses (`SettlementBatch.status`)**

- `calculated`: reserved for pre-scheduled calculation states.
- `scheduled`: payouts are queued and awaiting execution.
- `paid`: all payouts in the batch are terminally paid.
- `failed`: at least one payout failed and needs operator intervention.

**Payout statuses (`SettlementPayout.status`)**

- `scheduled`: ready to execute.
- `paid`: transfer completed (`externalTransferId` recorded).
- `failed`: Stripe transfer attempt failed.
- `manual_required`: cannot auto-pay (for example, missing Stripe account); export CSV and process
  manually.

See the dedicated operator guide for fallback payout + CSV procedures:
[`docs/settlements-runbook.md`](docs/settlements-runbook.md).

**Settlement/payout environment variables**

- `SETTLEMENT_SCHEDULER_ENABLED` (`true` by default): disables/enables scheduler job when set to
  `false`.
- `SETTLEMENT_CRON` (default `0 0 * * *`): cron expression for scheduling windows.
- `SETTLEMENT_TZ` (default `UTC`): settlement display timezone metadata for UI rendering only. It
  does not change UTC settlement boundary calculation or UTC persistence (does not control cron
  boundary logic).
- `SETTLEMENT_SCHEDULER_TZ` (default `UTC`): scheduler timezone for cron registration.
- `SETTLEMENT_PERIOD_DAYS` (default `15`): settlement window size.
- `SETTLEMENT_HOLD_DAYS` (default `0`): release delay after period end.
- `SETTLEMENT_CURRENCY` (default `USD`): settlement currency.
- `SETTLEMENT_AUTO_EXECUTE` (default `false`): auto-runs transfer execution for newly created cron
  batches.
- `SETTLEMENT_LOCK_KEY` (default `lock:settlement_scheduler`): distributed lock key shared across
  scheduler instances that should coordinate the same environment.
- `SETTLEMENT_LOCK_TTL_SECONDS` (default `120`): lock TTL in seconds; should exceed worst-case
  scheduler cycle duration with safety buffer.
- `SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY` (default `skip_cycle`): accepted values are `skip_cycle`
  or `run_unlocked`.

**Settlement Timezone Policy**

- Settlement periods are always computed on UTC boundaries (`00:00:00` start through UTC day-end)
  and stored in UTC fields.
- Sellers in non-UTC locales will still see UTC-aligned settlement periods; period boundaries do not
  shift to local midnights.
- `SETTLEMENT_TZ` only affects admin/UI date formatting (UTC -> display timezone conversion). It is
  never used for settlement boundary calculation or persistence.

**Scheduler lock behavior and operations guidance**

- The settlement scheduler uses Redis lock acquisition so only one runner schedules per cycle when
  multiple app instances are active.
- Redis is a dependency for strict single-runner behavior. If Redis lock acquisition fails:
  - `SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=skip_cycle` skips the cycle (safest for financial jobs).
  - `SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=run_unlocked` proceeds without lock (dev/test
    convenience, higher duplicate-risk in multi-runner deployments).
- Alerting expectation:
  - Treat repeated logs `Settlement scheduler lock acquisition failed.` or
    `Settlement scheduler cycle skipped because lock backend is unavailable.` as operational alerts.
  - Treat frequent `Settlement scheduler lock not acquired. Skipping cycle.` as signal to validate
    runner topology and lock TTL sizing.
- Keep lock key namespaces isolated per environment to avoid cross-environment collisions.

**Lock env presets by environment (copy-ready)**

```env
# Development (recommended values)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:dev # Lock key namespace for local dev; avoids collisions with staging/prod
SETTLEMENT_LOCK_TTL_SECONDS=120 # Lock auto-expiry in seconds; enough for local runs while keeping stale locks short
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=run_unlocked # If local Redis is unavailable, still run scheduler for easier development


# Production (safe default, avoid duplicate runs)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler # Redis distributed lock key; must be shared by all prod scheduler instances
SETTLEMENT_LOCK_TTL_SECONDS=300 # Lock expiry in seconds to avoid deadlocks; should cover worst-case cycle duration with buffer
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=skip_cycle # If Redis lock cannot be acquired due to Redis outage, skip this cycle (safest for financial jobs)


# Staging (prod-like behavior, isolated keyspace)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:staging # Same purpose as prod, but namespaced so staging never collides with prod
SETTLEMENT_LOCK_TTL_SECONDS=300 # Keep prod-like TTL to validate behavior under realistic conditions
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=skip_cycle # Match production safety policy during validation


# Testing (unit/integration convenience)
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:test # Test-only lock namespace to isolate parallel/local test runs
SETTLEMENT_LOCK_TTL_SECONDS=30 # Short TTL keeps tests fast and reduces stale-lock wait during repeated runs
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=run_unlocked # Allows tests to proceed when Redis is absent/unreliable (except lock-specific test cases)
```

**How to test payouts and what to expect**

1. Keep `SETTLEMENT_AUTO_EXECUTE=false`, schedule a batch, then call
   `POST /api/admin/settlements/:id/execute`.
   - Expect only `status='scheduled'` payouts to be processed.
   - Expect response fields `processed`, `paidCount`, `manualRequiredCount`, and optional
     `manualExport` CSV data.
2. Set `SETTLEMENT_AUTO_EXECUTE=true` and run the scheduler cycle.
   - Expect newly created batches to execute immediately in the same cycle.
   - Expect cron logs containing `Settlement auto-execution completed.` with counts.
3. Validate payout outcomes:
   - Seller missing Stripe/bank linkage => payout becomes `manual_required`.
   - Stripe transfer error => payout becomes `failed`.
   - Successful transfer => payout becomes `paid`, `externalTransferId` is stored, related ledger
     rows become `paid`.
4. Manual remediation path:
   - Retry failed payout(s) from admin retry endpoints.
   - Complete manual-required payouts from manual completion endpoint and verify ledger rows move

**Admin/manual scheduling response outcomes (quick reference)**

- **Created batch (HTTP 201)**

  ```json
  {
    "batch": { "id": "...", "status": "scheduled" },
    "scheduledEntries": 24,
    "sellerCount": 3
  }
  ```

- **Skipped batch window (HTTP 200, no work this cycle)**

  ```json
  {
    "created": false,
    "reason": "period_not_eligible"
  }
  ```

- **Duplicate/deduplicated outcome (HTTP 200, safe no-op)**

  ```json
  {
    "created": false,
    "reason": "duplicate"
  }
  ```

Common `created: false` reasons include `no_pending_entries`, `period_not_eligible`,
`no_positive_payouts`, and `duplicate`.

### Admin marketplace financials API (`/api/admin/platform-financials/*`)

Authorization for all routes below: `protectRoute` + `restrictTo('admin', 'staff')`.

#### `GET /api/admin/platform-financials/summary`

**Required query params**

- `scope`: `marketplace | platform_store | seller`
- `sellerId`: required when `scope=seller`

**Optional query params**

- `from`, `to` (ISO date/datetime)
- `type` (ledger type filter)
- `payoutStatus` (`pending|scheduled|paid|reserved`)
- `orderId`
- `page`, `limit`

**Response shape**

- `data.totals`: total entries/net/credit/debit/outstanding amounts
- `data.marketplaceMetrics`: GMV, platform fee revenue, processor fee cost/recovery, liabilities,
  payouts breakdown, reserve metrics, subscription fee receivables
- `data.byType[]`, `data.byPayoutStatus[]`
- `meta.generatedAt`
- `filters` echo envelope

#### `GET /api/admin/platform-financials/ledger`

Same filters as summary.

**Response shape**

- `data[]`: ledger rows
  - `id, sellerId, accountingScope, entrySide, merchantSellerId, counterpartyRole, type, amountCents, currency, payoutStatus, reason, actor, reference, reserveMonth, withheldAt, releaseScheduledAt, createdAt`
- `meta`: pagination (`page, limit, total, totalPages, hasNextPage, hasPrevPage`)
- `filters`: normalized filter envelope

#### `GET /api/admin/platform-financials/liabilities`

Same filters as summary/ledger; liability rows are constrained to liability types
(`tax/tax_refund/shipping_fee/shipping_refund/reserve/reserve_used/cod_refund_*`).

**Response shape**

- `data.summary`
  - `totalLiabilityAmountCents`
  - `outstandingLiabilityAmountCents`
  - `entryCount`
- `data.items[]`
  - grouped by `type + payoutStatus`
  - `type, payoutStatus, totalAmountCents, entryCount, lastCreatedAt`
- `meta` + `filters`

### Seller financial summary behavior updates

Endpoint: `GET /api/seller/financials/summary`

- For normal sellers, summary reflects the seller's rows in requested/default scope.
- For platform-store seller context (admin acting as platform seller), default scope is
  merchant-only and summary is restricted to platform-owned order IDs.

**New merchant subtotal fields in summary**

- `merchantGrossCents`
- `merchantTaxCents`
- `merchantShippingCents`
- `merchantCommissionCents`
- `merchantNetCents`
- `merchantOutstandingBalanceCents`

These are derived from unpaid merchant-side settlement types and are intended as seller-operator
readable business totals.

**Exact formulas (cents, aligned to current backend implementation)**

- Settlement balance = `merchantOutstandingBalanceCents` (same value currently returned as
  `outstandingBalanceCents` in summary response).
- Net proceeds before processor fees = `merchantNetCents` where
  `merchantNetCents = merchantGrossCents + merchantTaxCents + merchantShippingCents + merchantCommissionCents`.
- Deductions total = sum of values in `deductionTotalsCents` (currently seeded with `commission`,
  `processing_fee`, `cod_refund_reimbursement`, plus any additional deduction keys returned by API).
- Sign convention: positive amounts increase seller receivable; negative amounts reduce it.

**Outstanding balance interpretation**

- `outstandingBalanceCents` / `merchantOutstandingBalanceCents` are unsettled ledger totals for
  included types and scope, not "cash available now".
- Positive values generally indicate net receivable; negative values indicate net owed/offset state.
- COD policy consumes this scoped value against `negativeThresholdCents`.

### Frontend financial documentation updates

#### Seller dashboard financials scoping

- Seller financial summary (`/seller/financials/summary`) is consumed as merchant subtotals + COD
  policy block.
- For platform-store seller context, backend defaults scope to merchant and limits rows to
  platform-owned orders, so dashboard cards align with seller ledger merchant view.
- Dashboard card formulas should be applied in cents exactly as returned:
  - Settlement balance = `merchantOutstandingBalanceCents` (`outstandingBalanceCents` alias).
  - Net proceeds before processor fees = `merchantNetCents`, with
    `merchantNetCents = merchantGrossCents + merchantTaxCents + merchantShippingCents + merchantCommissionCents`.
  - Deductions total = sum of `deductionTotalsCents` values (`commission`, `processing_fee`,
    `cod_refund_reimbursement`, plus any additional deduction keys returned by API).
  - Sign convention: positive amounts increase seller receivable; negative amounts reduce it.

#### Operator rows in seller ledger

- Default seller ledger view is merchant-oriented.
- Operator mirror rows can be included for debugging/audit using `includeOperatorRows=true` in
  seller ledger queries.
- When included, descriptions are prefixed/labeled as platform-side offset entries to avoid
  confusion with merchant sales activity.

#### Admin marketplace financials scope switcher

`AdminMarketplaceFinancialsPanel` supports three modes:

1. `marketplace`: aggregate all sellers/scopes
2. `platform_store`: platform store merchant scope (platform-owned orders)
3. `seller`: single seller (requires seller selector)

The panel re-fetches summary, ledger, and liabilities whenever scope (or seller in seller mode)
changes.

#### UI diagram (scope flow)

```text
Admin Marketplace Financials
┌─────────────────────────────────────────────────────────────┐
│ Scope: [ marketplace | platform_store | seller ]           │
│                     └─ seller => requires sellerId         │
├─────────────────────────────────────────────────────────────┤
│ Summary cards (scope-aware)                                │
│ Ledger table (scope + filters)                             │
│ Liabilities breakdown (scope + filters)                    │
└─────────────────────────────────────────────────────────────┘

Seller Dashboard > Financials
┌─────────────────────────────────────────────────────────────┐
│ Summary cards (merchant subtotals + outstanding + COD)     │
│ Reserve card                                                │
│ Tabs: Ledger | Payouts | Liabilities                        │
│ Ledger default: merchant rows                               │
│ Optional: include operator rows (platform offset labeled)   │
└─────────────────────────────────────────────────────────────┘
```

### Release notes — accounting and financial UI refactor

#### Consolidated changes in this release

**1) Schema / data-contract changes**

- Ledger rows now explicitly rely on:
  - `accountingScope`, `entrySide`, `merchantSellerId`, `counterpartyRole`.
- Reserve lifecycle fields are first-class API/UI contract fields:
  - `reserveMonth`, `withheldAt`, `releaseScheduledAt`, and reserve `metadata` audit keys.
- COD refund UI now depends on status/failure metadata:
  - `codRefund.status`, `codRefund.failureCode`, `codRefund.failureMessage`,
    `codRefund.providerStatus`, `codRefund.providerRefundId`, and redacted detail helpers
    (`detailsSummary`, `detailsLast4`).
- Ledger replay safety is idempotency-key based across all ledger write paths.

**2) New / updated admin routes**

- Reserve operations under `/api/admin/reserves/*`:
  - `GET /summary`
  - `GET /sellers/:sellerId`
  - `POST /release`
  - `POST /increase`
  - `POST /update-config`
- Marketplace financials under `/api/admin/platform-financials/*`:
  - `GET /summary`
  - `GET /ledger`
  - `GET /liabilities`

**3) Frontend adjustments included**

- Scope-switch behavior standardized (`marketplace | platform_store | seller`) with seller-required
  fetch gating in seller mode.
- Liabilities rendering now consumes grouped liability summaries returned by admin liabilities API.
- Reserve audit/history rendering consumes actor/reason/tier/percentage/timestamp fields from
  reserve action metadata.
- Summary cards map directly to server totals/merchant subtotals without recomputing scope client-
  side.

#### Frontend action required (short checklist)

- Consume and persist these response keys in admin + seller financial views:
  - `accountingScope`, `entrySide`, `merchantSellerId`, `counterpartyRole`
  - `reserveMonth`, `withheldAt`, `releaseScheduledAt`, `metadata.{tier,percentage,reason,actor}`
  - `codRefund.status`, `codRefund.failureCode`, `codRefund.failureMessage`,
    `codRefund.providerStatus`, `codRefund.providerRefundId`, `codRefund.detailsSummary`,
    `codRefund.detailsLast4`
- Backward compatibility behavior:
  - If `accountingScope` is missing, treat row as `merchant` for seller dashboard display.
  - If `entrySide` is missing, render as `merchant` and hide operator badge.
  - If reserve audit metadata is missing, show `"system"` actor and `"not_provided"` reason
    fallback.
  - If COD failure fields are missing, keep a generic failure message and avoid hard-failing status
    UI.

### Reserve / Holdback Policy (implemented architecture)

The reserve system is designed to reduce platform exposure to post-payout refunds/disputes while
keeping seller payouts predictable and auditable.

#### 1) Policy model implemented in this repo

This implementation currently uses a **fixed-percentage reserve** model at settlement time:

- For each seller with positive net payable in `scheduleNextBatch`, the system computes:
  - `reserveAmountCents = floor(netPayable * reservePercentage)`
  - `payoutAmountCents = netPayable - reserveAmountCents`
- A negative `LedgerEntry(type='reserve', payoutStatus='reserved')` is created for the withheld
  amount.
- A `SettlementPayout` is created only for `payoutAmountCents`.
- Reserve ledger entries are intentionally not attached to a settlement batch (`settlementBatchId`
  stays unset), preserving reserve lifecycle independence.

Core files:

- `backend/src/services/settlements/settlementScheduler.service.js`
- `backend/src/services/reserves/reserve.service.js`
- `backend/src/models/ledgerEntry.model.js`
- `backend/src/models/seller.model.js`

#### 2) Ledger contract for reserves

`LedgerEntry` now includes reserve movement types:

- `reserve` (negative): withheld funds
- `reserve_release` (positive): funds released back to payout flow
- `reserve_used` (negative): reserve consumed to offset refund/debt

Reserve tracking fields:

- `reserveMonth` (`YYYY-MM`)
- `withheldAt`
- `releaseScheduledAt`
- `metadata` (reason, actor, tier, percentage, original months)
- `payoutStatus='reserved'` for hold-state entries

Auditability guarantee: reserve rows are immutable movement records; releases/usages add new rows
instead of mutating historical reserve entries.

#### 3) Risk tiers, percentage resolution, and env defaults

Per-seller config is stored on `Seller.reserveConfig`:

- `percentage`
- `tier` (`new | established | pro | high_risk`)
- `updatedAt`, `updatedBy`

Percentage resolution order (implemented):

1. Seller override (`seller.reserveConfig.percentage`)
2. Tier env default (`RESERVE_PERCENTAGE_<TIER>`)
3. Fallback `RESERVE_PERCENTAGE_DEFAULT` (or `0` if unset)

Recommended env defaults:

```env
# Reserve policy (production)
RESERVE_PERCENTAGE_NEW=15
RESERVE_PERCENTAGE_ESTABLISHED=5
RESERVE_PERCENTAGE_PRO=2
RESERVE_PERCENTAGE_HIGH_RISK=25
DEFAULT_RESERVE_TIER=new
DEFAULT_RESERVE_PERCENTAGE_NEW=15
RESERVE_MIN_HOLD_DAYS=14
RESERVE_RELEASE_CRON="0 2 * * *"
RESERVE_RELEASE_TZ=UTC
RESERVE_RELEASE_JOB_ENABLED=true


# Reserve policy (dev/test fast cycle)
RESERVE_PERCENTAGE_NEW=10
RESERVE_PERCENTAGE_ESTABLISHED=5
RESERVE_PERCENTAGE_PRO=2
RESERVE_PERCENTAGE_HIGH_RISK=20
DEFAULT_RESERVE_TIER=new
DEFAULT_RESERVE_PERCENTAGE_NEW=10

# Make reserves releasable quickly
RESERVE_MIN_HOLD_DAYS=0

# Run release job every 2 minutes
RESERVE_RELEASE_CRON="*/2 * * * *"
RESERVE_RELEASE_TZ=UTC
RESERVE_RELEASE_JOB_ENABLED=true

```

#### Per-seller special agreements

When operations/legal approves a special reserve arrangement for a specific seller, use **Admin →
Reserves** and follow this workflow so pricing logic and audits stay consistent:

1. Open the seller in the reserves tab and choose the reserve tier (`new`, `established`, `pro`,
   `high_risk`). This tier maps to the configured environment defaults (`RESERVE_PERCENTAGE_<TIER>`)
   and is the recommended option when the seller should follow policy baseline.
2. Decide how percentage is applied in UI:
   - **Use tier default**: keep percentage aligned to the selected tier's configured env value.
   - **Custom override**: choose custom and set a seller-specific percentage in the same reserves
     tab when a negotiated exception is approved.
3. Record a clear business reason in the admin action reason field (for example:
   `signed_enterprise_addendum_q3` or `temporary_chargeback_ramp_down`) so finance/support can trace
   why the override exists.

Runtime resolution still follows the documented order (seller override first, then tier env
default), so UI choices are deterministic and auditable.

#### 4) Reserve release cron job (design + operations)

A dedicated daily cron job (`backend/src/jobs/reserveRelease.js`) executes `runReserveReleaseJob()`.

Eligibility logic used by the job:

- `type='reserve'`
- `payoutStatus='reserved'`
- `releaseScheduledAt <= now`
- `reserveMonth != currentMonth`

Processing behavior:

- Groups eligible reserves by seller/currency/month set
- Creates `reserve_release` entries as `payoutStatus='pending'` so normal settlements can pick them
  up
- If seller has negative pending non-reserve balance, reserves are applied via `reserve_used`
  instead of releasing

Operational notes:

- Start from `backend/src/server.js` through `startReserveReleaseJob()`.
- Invalid cron expression logs warning and skips job registration.
- Job is independent from settlement scheduler cadence.

#### 5) Fixed reserve vs rolling reserve (when to choose each)

**Implemented here: Fixed reserve (recommended baseline)**

- Each settlement withholds a deterministic percentage of that period's payable amount.
- Simpler to explain to sellers/admins and easier to audit from ledger rows.
- Best when you want transparent policy and straightforward monthly release accounting.

**Alternative: Rolling reserve (not currently implemented)**

- Holds a moving window of recent payouts (e.g., 10% of last 90 days) and continuously recalculates
  target reserve.
- Better for highly volatile risk profiles or long-tail dispute windows.
- More complex reconciliation, seller communication, and accounting treatment.

Choose rolling when risk volatility materially changes within the hold horizon; otherwise fixed is
usually easier operationally.

#### 6) Admin reserve APIs and UI

Backend endpoints (`/api/admin/reserves`):

- `GET /summary`
- `GET /sellers/:sellerId`
- `POST /release`
- `POST /increase`
- `POST /update-config`

Authorization for all routes below: `protectRoute` + `restrictTo('admin', 'staff')`.

##### `GET /api/admin/reserves/summary`

**Request**

```http
GET /api/admin/reserves/summary?range=30d&currency=USD HTTP/1.1
Authorization: Bearer <ADMIN_OR_STAFF_TOKEN>
Accept: application/json
```

**Response (HTTP 200)**

```json
{
  "success": true,
  "data": {
    "totals": {
      "currency": "USD",
      "reservedAmountCents": 1842500,
      "releasableAmountCents": 392000,
      "usedAmountCents": 118000,
      "sellerCount": 42
    },
    "byTier": [
      { "tier": "new", "reservedAmountCents": 962000, "sellerCount": 17 },
      { "tier": "established", "reservedAmountCents": 580500, "sellerCount": 14 },
      { "tier": "pro", "reservedAmountCents": 199000, "sellerCount": 8 },
      { "tier": "high_risk", "reservedAmountCents": 101000, "sellerCount": 3 }
    ],
    "generatedAt": "2026-04-21T10:40:12.201Z"
  }
}
```

##### `GET /api/admin/reserves/sellers/:sellerId`

**Request**

```http
GET /api/admin/reserves/sellers/664f0cc0f9e53a001f8ab123?limit=10&page=1 HTTP/1.1
Authorization: Bearer <ADMIN_OR_STAFF_TOKEN>
Accept: application/json
```

**Response (HTTP 200)**

```json
{
  "success": true,
  "data": {
    "seller": {
      "_id": "664f0cc0f9e53a001f8ab123",
      "shopName": "Acme Gadgets",
      "reserveConfig": {
        "tier": "established",
        "percentage": 5,
        "updatedAt": "2026-04-10T12:15:40.000Z",
        "updatedBy": "664eff40d9c53a001f8aa9a2"
      }
    },
    "summary": {
      "currency": "USD",
      "currentlyReservedCents": 156400,
      "releasableCents": 23000,
      "usedCents": 9000
    },
    "ledger": [
      {
        "_id": "66500152bc41cc001fbbe771",
        "type": "reserve",
        "amountCents": -6400,
        "payoutStatus": "reserved",
        "reserveMonth": "2026-03",
        "withheldAt": "2026-03-31T23:59:40.000Z",
        "releaseScheduledAt": "2026-04-14T00:00:00.000Z",
        "metadata": {
          "tier": "established",
          "percentage": 5,
          "reason": "scheduled_settlement_withhold",
          "actor": "system"
        }
      }
    ]
  },
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 37,
    "totalPages": 4
  }
}
```

##### `POST /api/admin/reserves/release`

**Request**

```http
POST /api/admin/reserves/release HTTP/1.1
Authorization: Bearer <ADMIN_OR_STAFF_TOKEN>
Content-Type: application/json

{
  "sellerId": "664f0cc0f9e53a001f8ab123",
  "amountCents": 20000,
  "currency": "USD",
  "reason": "manual_partial_release_after_chargeback_window"
}
```

**Response (HTTP 200)**

```json
{
  "success": true,
  "message": "Reserve released successfully.",
  "data": {
    "sellerId": "664f0cc0f9e53a001f8ab123",
    "releasedAmountCents": 20000,
    "currency": "USD",
    "ledgerEntry": {
      "_id": "66500217bc41cc001fbbe9e0",
      "type": "reserve_release",
      "amountCents": 20000,
      "payoutStatus": "pending",
      "metadata": {
        "reason": "manual_partial_release_after_chargeback_window",
        "actor": "admin:664eff40d9c53a001f8aa9a2"
      }
    }
  }
}
```

##### `POST /api/admin/reserves/increase`

**Request**

```http
POST /api/admin/reserves/increase HTTP/1.1
Authorization: Bearer <ADMIN_OR_STAFF_TOKEN>
Content-Type: application/json

{
  "sellerId": "664f0cc0f9e53a001f8ab123",
  "amountCents": 15000,
  "currency": "USD",
  "reason": "temporary_risk_buffer_for_open_dispute"
}
```

**Response (HTTP 200)**

```json
{
  "success": true,
  "message": "Reserve increased successfully.",
  "data": {
    "sellerId": "664f0cc0f9e53a001f8ab123",
    "increasedAmountCents": 15000,
    "currency": "USD",
    "ledgerEntry": {
      "_id": "66500265bc41cc001fbbea40",
      "type": "reserve",
      "amountCents": -15000,
      "payoutStatus": "reserved",
      "metadata": {
        "reason": "temporary_risk_buffer_for_open_dispute",
        "actor": "admin:664eff40d9c53a001f8aa9a2",
        "source": "manual_increase"
      }
    }
  }
}
```

##### `POST /api/admin/reserves/update-config`

**Request**

```http
POST /api/admin/reserves/update-config HTTP/1.1
Authorization: Bearer <ADMIN_OR_STAFF_TOKEN>
Content-Type: application/json

{
  "sellerId": "664f0cc0f9e53a001f8ab123",
  "tier": "high_risk",
  "percentage": 18,
  "reason": "chargeback_spike_q2_review"
}
```

**Response (HTTP 200)**

```json
{
  "success": true,
  "message": "Reserve config updated.",
  "data": {
    "sellerId": "664f0cc0f9e53a001f8ab123",
    "reserveConfig": {
      "tier": "high_risk",
      "percentage": 18,
      "updatedAt": "2026-04-21T10:44:51.328Z",
      "updatedBy": "664eff40d9c53a001f8aa9a2"
    },
    "audit": {
      "action": "update-config",
      "reason": "chargeback_spike_q2_review",
      "actor": "admin:664eff40d9c53a001f8aa9a2"
    }
  }
}
```

Frontend admin UI:

- `frontend/src/components/AdminReservesPanel.jsx`
- `frontend/src/stores/useAdminReservesStore.js`
- Admin tab mounted in `frontend/src/pages/AdminPage.jsx`.

Manual overrides require reason fields and capture actor metadata for audit trails.

**Audit model:** reserve audit is designed as immutable movement + action history.

- Financial movement audit: ledger rows (`reserve`, `reserve_release`, `reserve_used`) are append-
  only records, so historical reserve activity is preserved instead of rewritten.
- Admin action audit: each manual reserves action (`update-config`, `release`, `increase`) stores
  actor identity, reason text, selected tier, effective percentage, and operation timestamp.
- Visibility in UI: in **Admin → Reserves**, open seller details and inspect the audit/history rows
  to review who changed what, why it was changed, and which tier/custom percentage was applied.
- Operational intent: finance/compliance can reconstruct both policy intent (tier/env baseline vs
  custom override) and monetary effect (ledger movements) without mutating prior records.

#### 7) Manual test plan (reserve flows)

1. Configure reserve env vars (`RESERVE_PERCENTAGE_*`, `RESERVE_MIN_HOLD_DAYS`) and run backend.
2. Ensure a seller has positive pending ledger net.
3. Trigger scheduler (`POST /api/admin/settlements/schedule`).
4. Verify new `reserve` ledger row exists and payout amount is net of reserve.
5. Move time forward (or seed old `releaseScheduledAt`) and run reserve release cycle.
6. Verify `reserve_release` appears as `payoutStatus='pending'`.
7. Trigger next settlement schedule and confirm released amount is included in payable pipeline.
8. Simulate refund after payout and verify `reserve_used` is created before remaining debt carry.
9. Use admin reserve endpoints to test manual release/increase/update-config with audit metadata.

Operator checklist (quick):

- update config (tier env default or custom value in Admin → Reserves)
- verify audit row (actor, reason, tier, percentage, timestamp)

---

### Ledger Adjustment Service

This section is about a dedicated admin adjustment service for non-order financial corrections.

#### What it does

`createAdjustment({ sellerId, amountCents, currency, reason, actor })` in
`backend/src/services/ledger/adjustment.service.js` creates immutable
`LedgerEntry(type='adjustment')` rows for manual corrections (credits/debits).

Key behavior:

- Validates `sellerId`, integer/non-zero `amountCents`, and non-empty `reason`
- Resolves currency fallback in this order:
  1. explicit `currency` argument
  2. `PLATFORM_CURRENCY`
  3. `SETTLEMENT_CURRENCY`
  4. `USD`
- Uses non-order references (`referenceType='adjustment'`, `reference.orderId=null`)
- Persists `reason` and `actor` for audit/filtering

#### API surface

Admin endpoints:

- `POST /api/admin/ledger/adjustments` – create adjustment
- `GET /api/admin/ledger/adjustments` – list/filter adjustments (seller/date/reason/pagination)

Controller: `backend/src/controllers/ledger.controller.js` Route wiring:

#### Frontend admin panel

- Store: `frontend/src/stores/useAdminAdjustmentsStore.js`
- UI: `frontend/src/components/AdminAdjustmentsPanel.jsx`
- Mounted in Admin tabs via `frontend/src/pages/AdminPage.jsx`

Supported workflows:

- Create debit/credit adjustments per seller
- Filter history by seller, date range, reason
- Review paginated, audit-oriented adjustment history

---

### Settlement Reconciliation & Payout Recovery

This section documents the dedicated reconciliation + recovery layer on top of settlement
scheduling/execution so operators can continuously verify data integrity between internal records
and Stripe.

#### 1) Reconciliation Architecture (Backend)

The reconciliation service runs two primary checks:

- **Batch consistency checks** (`checkBatchConsistency`):
  - Compares total `LedgerEntry.amountCents` values (including `paid` sums) against
    `SettlementBatch` and summed `SettlementPayout` totals.
  - Flags mismatches such as:
    - `batch_total_vs_ledger_total_mismatch`
    - `batch_total_vs_payout_total_mismatch`
    - `seller_payout_mismatch`
- **Payout-vs-Stripe checks** (`checkPayoutWithStripe`):
  - Validates payout amount and status against Stripe transfer state.
  - Detects missing/failed transfer lookups, amount mismatches, and status drift:
    - `stripe_transfer_lookup_failed`
    - `stripe_transfer_missing`
    - `stripe_amount_mismatch`
    - `stripe_status_mismatch`
- **Stuck processing / in-progress recovery intent**:
  - Payouts expose `status`, `processingAt`, `inProgress`, `inProgressAt`, `idempotencyKey`, and
    `externalTransferId` so operators can identify stuck or partial executions and safely drive
    retry/manual intervention paths.
  - Admin UI highlights long-running processing rows (15+ minutes) to prompt investigation before
    duplicate actions.

**Data flow (what reconciliation traverses):**

`SettlementBatch` -> `SettlementPayout` rows -> `LedgerEntry` rows (`settlementBatchId`) -> Stripe
transfer (`externalTransferId` or payout/batch metadata lookup).

**Expected outputs:**

- `ok: true` means the reconciliation result is effectively **OK**.
- `ok: false` returns discrepancy categories/codes and actionable details (`batchId`, `payoutId`,
  expected vs actual amounts, Stripe lookup method, statuses, and timestamps) so finance/admin can
  triage without DB inspection.

#### 2) Cron Job for Reconciliation

- Reconciliation cron is controlled by `startSettlementReconciliationJob` and runs on
  `SETTLEMENT_RECONCILIATION_CRON`.
- Every cycle computes a rolling scan window using `SETTLEMENT_RECONCILIATION_SCAN_DAYS` and scans:
  - recent `SettlementBatch` records,
  - recent terminal payouts (`paid`, `failed`, `manual_required`).
- For each entity, it executes batch/payout reconciliation checks and:
  - logs warnings for discrepancies,
  - logs a cycle summary (`ok` vs `discrepancies`),
  - sends a role-based reconciliation report email to users resolved from
    `SETTLEMENT_RECONCILIATION_REPORT_ROLES`.

**Where results are reported + review ownership:**

- Application logs (`Settlement reconciliation cycle started/completed`, discrepancy warnings).
- Reconciliation summary emails (via configured role recipients).
- Expected reviewers: admin/ops/finance users mapped to report roles (`admin,staff` is the current
  recommended baseline).

**Operational cadence recommendation:**

- **Development:** run every 15 minutes with 1-day scan window for fast feedback.
- **Production:** run once daily (e.g., `15 2 * * *`) with 2-day window for late Stripe updates and
  overnight reconciliation.

#### 3) Environment Variables

Copy/paste-ready reconciliation config:

```env
# settlements reconciliation config vars (recommended development values)
SETTLEMENT_RECONCILIATION_ENABLED=true
SETTLEMENT_RECONCILIATION_REPORT_ROLES=admin,staff
SETTLEMENT_RECONCILIATION_CRON=*/15 * * * *
SETTLEMENT_RECONCILIATION_TIMEZONE=UTC
SETTLEMENT_RECONCILIATION_SCAN_DAYS=1

# settlements reconciliation config vars (recommended production values)
SETTLEMENT_RECONCILIATION_ENABLED=true
SETTLEMENT_RECONCILIATION_REPORT_ROLES=admin,staff
SETTLEMENT_RECONCILIATION_CRON=15 2 * * *
SETTLEMENT_RECONCILIATION_TIMEZONE=UTC
SETTLEMENT_RECONCILIATION_SCAN_DAYS=2
```

Variable notes:

- `SETTLEMENT_RECONCILIATION_ENABLED`: enables/disables the reconciliation scheduler.
- `SETTLEMENT_RECONCILIATION_REPORT_ROLES`: comma/semicolon-separated roles used to build recipient
  lists from user records.
- `SETTLEMENT_RECONCILIATION_CRON`: cron expression for cycle timing.
- `SETTLEMENT_RECONCILIATION_TIMEZONE`: timezone applied by the cron scheduler.
- `SETTLEMENT_RECONCILIATION_SCAN_DAYS`: retrospective window for scanned batches/payouts.
  - Use **2 days** for faster production scans with reasonable delayed-event tolerance.
  - Use **7 days** when delayed Stripe updates/webhook lag is common and safety coverage is favored
    over scan cost.

#### 4) Admin Reconciliation Endpoints and UI Usage

**Permissions:** all `/api/admin/*` settlement reconciliation endpoints are protected by
`protectRoute` + `restrictTo('admin')`.

**Endpoints:**

- `GET /api/admin/settlements/:batchId/reconcile`
- `GET /api/admin/settlements/payouts/:payoutId/reconcile`

Expected response shapes:

```json
{
  "ok": true,
  "batchId": "...",
  "status": "scheduled",
  "totals": {
    "batchTotalAmountCents": 12345,
    "payoutTotalAmountCents": 12345,
    "ledgerTotalAmountCents": 12345
  },
  "discrepancies": [],
  "checkedAt": "2026-01-01T00:00:00.000Z"
}
```

```json
{
  "ok": false,
  "payoutId": "...",
  "batchId": "...",
  "internalStatus": "paid",
  "externalTransferId": "tr_...",
  "lookupMethod": "externalTransferId",
  "discrepancies": [
    {
      "code": "stripe_amount_mismatch",
      "expectedAmountCents": 1000,
      "actualAmountCents": 900
    }
  ],
  "checkedAt": "2026-01-01T00:00:00.000Z"
}
```

**Admin UI behavior:**

- Batch and payout rows expose **Run Reconciliation** actions.
- Results are attached to the row and rendered in mismatch detail tables (type/internal ID/external
  ID/reason), making discrepancies visible inline for operations triage.
- Stale `processing` rows are visibly flagged to nudge recovery actions (retry/manual/review) before
  further payout operations.

#### 5) Payout Reversals

Reverse payout endpoint:

- `POST /api/admin/settlements/payouts/:payoutId/reverse`

Behavior + prerequisites:

- Requires valid payout id and existing payout.
- Requires `externalTransferId` (Stripe transfer reference).
- Reversal is only allowed when payout status is `paid`.
- Already reversed payouts return idempotent no-op semantics (`status: reversed`, existing
  `reversalId` returned).
- Service calls Stripe `transfers.createReversal(...)` and persists `reversalId`,
  `reversalRequestedAt`, and `reversedAt`.

Status transitions + ledger impact:

- `paid` -> `reversed` on success.
- Creates a negative ledger entry:
  - `type: payout_reversal`
  - `amountCents: -abs(original payout amount)`
  - `referenceType: payout`, `referenceId: payoutId`
  - idempotency key `settlement-reversal:<payoutId>`
- This preserves auditability of reversal intent + execution without mutating historical ledger
  rows.

Operator safeguards:

- UI presents a confirmation modal before issuing reversal.
- Endpoint is admin-only.
- Reversal flow is designed for idempotent/deduplicated retries: once reversed, subsequent calls do
  not create additional reversal ledger effects.

#### 6) Runbook Linkage

For incident handling, remediation sequencing, and operator SOP details, use:

- [`docs/settlements-runbook.md`](docs/settlements-runbook.md)

### Settlement Payout Retry & Cron Operations

This section captures the payout resiliency behavior and how to operate it safely.

#### 1) What changed

##### `executeStripePayout` retry behavior

- **Retryable Stripe/transient classes:** transfer retries are attempted for transient connectivity
  and throttling failures (`api_connection_error`, `rate_limit_error`) and timeout-style transport
  failures (`etimedout`, `timeout`, `timed_out`, `request_timeout`, `ecconnaborted`, `econnaborted`,
  `econnreset`, or timeout-like error messages).
- **Exponential backoff sequence:** retries use `100ms -> 500ms -> 2000ms` delays, giving up after
  the final attempt.
- **Idempotency reuse across attempts:** the same payout idempotency key is reused for all attempts
  (`payout.idempotencyKey` if present, otherwise `settlement:<batchId>:<sellerId>:<currency>`),
  preventing duplicate transfers when Stripe eventually accepts a retried request.
- **Terminal behavior after max retries:** when attempts are exhausted, payout is persisted as
  `failed` with enriched `methodSnapshot.retry` context (`attemptCount`, per-attempt history,
  `retriesExhausted`, and final Stripe error metadata including request id). A reconciliation alert
  payload is also emitted with `payoutFailureContext` so operators can triage from logs/email
  without DB forensics.

##### `reverseStripePayout` idempotency and retry behavior

- **Skip if already reversed:** if reversal evidence already exists (`reversalId` or
  `methodSnapshot.reversal.id`), reversal is treated as idempotent no-op completion and the payout
  is normalized to `status: reversed` without creating duplicate financial effects.
- **Reversal idempotency key format:** Stripe reversal calls use `settlement-reversal:<payoutId>`.
- **Retry + escalation:** reversal calls share the same transient retry strategy/backoff as
  execution (`100ms -> 500ms -> 2000ms`). If retries exhaust, the service logs a structured failure,
  returns `failureContext`, and sends reconciliation-report alerts so operations can manually
  investigate/resolve.

##### `SettlementPayout` schema/index hardening

- Added `retryCount` (non-negative, indexed) to track payout retry lifecycle.
- Added unique partial index for `externalTransferId` so any non-empty transfer id is globally
  unique.
- `methodSnapshot` now explicitly documents canonical execution audit fields:
  - `methodSnapshot.executedAt`
  - `methodSnapshot.idempotencyKey`

##### New payout retry cron job

- **Selection criteria:** cron scans payouts where `status = failed` and
  `retryCount < SETTLEMENT_PAYOUT_RETRY_MAX_RETRIES`.
- **Success path:** on successful retry, payout is confirmed `paid` and `retryCount` is reset to
  `0`.
- **Failure path:** unsuccessful attempts increment `retryCount`; once threshold is reached, payout
  transitions to `manual_required` and emits an operator alert.

##### Job-health monitoring

- **Stale scheduler detection:** job-health raises alerts if `settlement_scheduler` has not
  completed inside `JOB_HEALTH_SCHEDULER_STALE_HOURS`.
- **Payout retry failure-rate alerts:** payout retry runs are windowed and failure-rate alerts are
  triggered when configured minimum-run and threshold conditions are met.

#### 2) Environment variables reference

| Variable                                   | Purpose                                                                          | Accepted format / range                                                                   | Operational effect                                                                                                           |
| ------------------------------------------ | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `SETTLEMENT_PAYOUT_RETRY_ENABLED`          | Enables/disables payout retry cron scheduling.                                   | Boolean-like (`true/false`, `1/0`, `yes/no`, `on/off`).                                   | `false` disables automatic recovery, so failed payouts remain failed/manual until operators act.                             |
| `SETTLEMENT_PAYOUT_RETRY_CRON`             | Cron cadence for retry cycles.                                                   | Valid cron expression (node-cron syntax).                                                 | Faster cadence reduces time-to-recovery but increases Stripe/API pressure and log volume.                                    |
| `SETTLEMENT_PAYOUT_RETRY_MAX_RETRIES`      | Max failed retry attempts before escalation.                                     | Positive integer (`>=1`).                                                                 | Lower values escalate to `manual_required` faster; higher values increase auto-retry persistence before manual intervention. |
| `SETTLEMENT_SCHEDULER_TZ`                  | Timezone used by settlement-related cron jobs.                                   | Valid IANA timezone (e.g., `UTC`, `America/New_York`). Invalid values fall back to `UTC`. | Controls when cron expressions fire in wall-clock terms; mismatches can cause off-hours execution.                           |
| `JOB_HEALTH_SCHEDULER_STALE_HOURS`         | Staleness threshold for scheduler-health alerting.                               | Positive integer hours.                                                                   | Lower values alert sooner for stuck schedulers; overly low values can create noisy false positives.                          |
| `JOB_HEALTH_PAYOUT_FAILURE_RATE_THRESHOLD` | Failure-rate alert threshold for payout retry job.                               | Decimal between `0` and `1` inclusive.                                                    | Lower thresholds make alerting more sensitive to degradation; higher thresholds reduce alert frequency.                      |
| `JOB_HEALTH_PAYOUT_FAILURE_WINDOW_RUNS`    | Number of latest runs evaluated for payout failure rate.                         | Positive integer.                                                                         | Smaller windows react quickly to spikes; larger windows smooth volatility and react slower.                                  |
| `JOB_HEALTH_PAYOUT_FAILURE_MIN_RUNS`       | Minimum runs required before failure-rate alerts can trigger.                    | Positive integer.                                                                         | Prevents early noisy alerts on low sample sizes; too high can delay real incident detection.                                 |
| `SETTLEMENT_RECONCILIATION_REPORT_EMAILS`  | Explicit recipient list for reconciliation/retry/reversal failure notifications. | Comma/semicolon separated email list.                                                     | Empty value suppresses direct email notifications; configured value provides immediate operator visibility.                  |

#### 3) Recommended value profiles

**Development / testing profile** (fast feedback, aggressive retry testing)

```env
# Settlement payout retry behavior (dev): fast loops + quick escalation for test cycles
SETTLEMENT_PAYOUT_RETRY_ENABLED=true
SETTLEMENT_PAYOUT_RETRY_CRON=*/5 * * * *                 # run every 5 minutes for rapid feedback
SETTLEMENT_PAYOUT_RETRY_MAX_RETRIES=2                    # intentionally low for quicker manual_required testing
SETTLEMENT_SCHEDULER_TZ=UTC                              # stable scheduler timing across developer machines/CI

# Job-health sensitivity (dev): detect regressions quickly
JOB_HEALTH_SCHEDULER_STALE_HOURS=2                       # flag scheduler stalls quickly while testing
JOB_HEALTH_PAYOUT_FAILURE_RATE_THRESHOLD=0.20            # alert when >=20% of recent runs fail
JOB_HEALTH_PAYOUT_FAILURE_WINDOW_RUNS=6                  # evaluate small window for fast signal
JOB_HEALTH_PAYOUT_FAILURE_MIN_RUNS=3                     # allow alerting after limited warmup

# Alert routing (dev/test inboxes)
SETTLEMENT_RECONCILIATION_REPORT_EMAILS=devops@example.com,qa@example.com
```

**Production profile** (safe cadence, stable alerting)

```env
# Settlement payout retry behavior (prod): balanced recovery and API stability
SETTLEMENT_PAYOUT_RETRY_ENABLED=true
SETTLEMENT_PAYOUT_RETRY_CRON=30 * * * *                 # run hourly at minute 30
SETTLEMENT_PAYOUT_RETRY_MAX_RETRIES=3                   # recommended production default
SETTLEMENT_SCHEDULER_TZ=UTC                             # predictable cross-region scheduling baseline

# Job-health sensitivity (prod): stable signal, lower alert noise
JOB_HEALTH_SCHEDULER_STALE_HOURS=24                     # alert if scheduler misses a full-day completion window
JOB_HEALTH_PAYOUT_FAILURE_RATE_THRESHOLD=0.30           # alert when >=30% failure rate across evaluation window
JOB_HEALTH_PAYOUT_FAILURE_WINDOW_RUNS=20                # smooth short spikes via wider run window
JOB_HEALTH_PAYOUT_FAILURE_MIN_RUNS=5                    # require enough samples before paging

# Alert routing (ops/on-call)
SETTLEMENT_RECONCILIATION_REPORT_EMAILS=finance-ops@example.com,oncall@example.com
```

**Why dev uses `2` retries while production uses `3`:** development optimizes for rapid failure-path
validation and faster operator handoff testing, while production keeps one extra retry attempt to
absorb transient Stripe/network instability and reduce unnecessary manual escalations.

#### 4) Runbook / troubleshooting

- **Detect stuck scheduler:** monitor job-health for stale `settlement_scheduler` completion age
  breaching `JOB_HEALTH_SCHEDULER_STALE_HOURS`, and corroborate with missing settlement/retry cycle
  logs.
- **Detect rising retry failure rate:** watch payout retry job-health `alerts.payoutFailureRate` and
  threshold-breach logs; rising trend usually indicates Stripe dependency, connectivity, or
  account-state issues.
- **When to manually intervene (`manual_required`):** intervene when payouts cross retry threshold,
  have persistent linkage/configuration failures, or repeated Stripe hard errors; use admin
  reconciliation + manual completion/export workflow.
- **Temporary incident tuning:** during active incidents, you can (a) shorten
  `SETTLEMENT_PAYOUT_RETRY_CRON` for tighter retry loops, (b) increase/decrease
  `SETTLEMENT_PAYOUT_RETRY_MAX_RETRIES` based on whether you prefer auto-recovery vs explicit manual
  control, and (c) relax/tighten job-health thresholds to match incident noise tolerance. Revert to
  baseline values after stabilization.

#### 5) Validation checklist (post-deploy)

- [ ] Confirm payout retry cron is enabled and scheduled (`SETTLEMENT_PAYOUT_RETRY_ENABLED=true`,
      schedule log emitted).
- [ ] Confirm `SettlementPayout` indexes are present (including unique partial `externalTransferId`)
      via migration/check.
- [ ] Confirm job-health records/metrics are updating for `settlement_scheduler` and `payout_retry`.
- [ ] Confirm alert channel/email recipients receive a controlled test notification.

### Stripe Connect prerequisites (seller payouts)

For automatic settlement transfers, each seller must have:

1. A Stripe Connect account ID saved on `seller.payout.stripeAccountId`.
2. A connected external bank account tokenized via Stripe and attached as
   `seller.payout.bank.externalAccountId`.
3. Approved payout/bank details in seller profile (`seller.payout.bank.*`) after any required review
   flow.

**Account/external-account flow**

- On first payout-method onboarding, backend creates a Stripe Connect Custom account if missing.
- Backend attaches the submitted bank token with `accounts.createExternalAccount`.
- `STRIPE_CONNECT_COUNTRY` controls the single-country payout rail for MVP (default: `US`).
- On onboarding and bank updates, submitted `bankCountry` must match `STRIPE_CONNECT_COUNTRY` (after
  ISO-2 normalization).
- Existing external account can be replaced; previous account cleanup is attempted when Stripe
  supports it.
- If no `stripeAccountId` exists at execution time, payout is marked `manual_required` and included
  in CSV fallback export.

---

## Products, variants & model behavior

- Products support:
  - Multiple languages for textual fields (e.g., `name.en`, `name.ar`).
  - `images` (array) and per-variant `image` override.
  - `price` and per-variant `price` override.
  - `stock` (global) and per-variant `stock`.
  - `deal` (discount percentage) applied to item price calculation.

- Variants:
  - Stored as objects with `attributes` (Map-like), `price`, `stock`, and optional `images` (up to
    10 URLs).
  - `image` remains as the first entry of `images` for compatibility with legacy consumers.
  - Matching helper `findMatchingVariant(product, variantAttributes)` is used widely to:
    - Determine per-item price on checkout and line items.
    - Decrement / increment the exact variant stock on reserve/restore.

- The frontend displays variant-specific price and stock and adds per-variant images when available.

- Product badges:
  - **Deal**, **Best Seller**, **New Release** badges.
  - **Stock badge**: shown when stock ≤ 5.
  - **Discount badge**: supports discount values from **1% to 80%**.

- Ability to **hide** items from the UI (instead of deleting) so you can retain product data and
  reinstate when restocked.

---

## Dynamic categories system (tree + drag-and-drop)

The store now supports a **feature-flagged, nested category tree** with drag-and-drop management,
localized labels, Cloudinary-hosted images, and leaf-aware browsing. Enable it in both clients by
setting `FEATURE_CATEGORY_TREE_V2=true` on the backend and `VITE_FEATURE_CATEGORY_TREE_V2=true` on
the frontend.

### Data model & flags

- Categories are stored in `Category` documents with localized `name` fields, unique `slug`,
  optional `parentCategory` for nesting, `imageUrl`, `isActive`, and `position` for
  ordering.【F:backend/src/models/category.model.js†L5-L25】
- The backend exposes `GET /categories/tree` only when `FEATURE_CATEGORY_TREE_V2` is enabled; it
  builds a tree sorted by `position`, calculates per-node product counts, and can optionally hide
  inactive or empty branches via `includeHidden` / `hideEmpty` query
  params.【F:backend/src/controllers/category.controller.js†L48-L114】【F:backend/src/controllers/category.controller.js†L132-L155】
- Product creation/editing requires `categoryId` and `subCategoryId` when the flag is enabled; both
  IDs and their slugs are stored on each product for quick
  lookup.【F:backend/src/controllers/product.controller.js†L429-L459】【F:backend/src/controllers/product.controller.js†L820-L843】

### Admin workflow (drag-and-drop + image upload)

- The **Admin → Categories** tab renders only when the flag is on and provides a drag-and-drop tree
  editor powered by the `useCategoryStore` Zustand slice (fetch, create, update, delete,
  reorder).【F:frontend/src/pages/AdminPage.jsx†L9-L82】【F:frontend/src/components/CategoryManager.jsx†L1-L205】
- Drag handles let admins move categories before/after/into siblings; the client builds a flat
  `{ id, parentCategory, position }[]` payload and posts it to `POST /categories/reorder` with cycle
  detection on both client and
  server.【F:frontend/src/components/CategoryManager.jsx†L160-L211】【F:backend/src/controllers/category.controller.js†L304-L355】
- Create/edit forms accept localized names, slug, parent selection, activation toggle, and either a
  direct `imageUrl` or uploaded file; uploads are processed via Sharp, stored temporarily under
  `UPLOADS_BASE_DIR/CATEGORY_IMAGES_DIR`, and pushed to the `CLOUDINARY_CATEGORY_FOLDER` path before
  cleanup.【F:frontend/src/components/CategoryManager.jsx†L33-L139】【F:backend/src/controllers/category.controller.js†L72-L122】
- Deletion is blocked when a category still has children to prevent orphaned
  branches.【F:backend/src/controllers/category.controller.js†L236-L260】

### Customer browsing (categories-first, then products)

- Home and category pages consume the tree from `/categories/tree` (with `hideEmpty=true` by
  default) and render localized labels via
  `useCategoryStore`.【F:frontend/src/pages/HomePage.jsx†L5-L83】【F:frontend/src/stores/useCategoryStore.js†L23-L77】
- Category routes call `GET /categories/browse/:slug`, which returns either child categories or a
  paginated product list with breadcrumb-style ancestors. Pagination is handled client-side by
  requesting successive pages while retaining prior
  results.【F:backend/src/controllers/category.controller.js†L260-L355】【F:frontend/src/pages/CategoryPage.jsx†L1-L174】
- Navbar dropdowns still fall back to the static locale-based list when the flag is off, so the
  legacy flat categories remain functional alongside the new
  system.【F:frontend/src/components/Navbar.jsx†L1-L113】
- A **feature-flagged "All" menu (V2)** uses the dynamic tree with hide-empty rules; set
  `VITE_FEATURE_NAV_CATEGORY_MENU_V2=true` in `frontend/.env` to enable the accordion-style dropdown
  that routes directly to leaf categories and expands non-leaf nodes on row click (flag off keeps
  the legacy
  menu).【F:frontend/src/components/Navbar.jsx†L1-L130】【F:frontend/src/components/nav/NavAllMenuV2.jsx†L1-L138】

---

### Product creation & image upload pipeline (frontend → backend → Cloudinary)

This project implements a **robust, rate-limit-friendly** image flow for both product creation and
editing. It works the same in local dev and production (Railway).

#### 1) Frontend (Create / Edit Product forms)

- Users can attach gallery images (`images[]`) and optional **per-variant images**
  (`colorImages[]`). Each variant can store **up to 10** images of its own, with total uploads
  bounded by the configured **`MAX_UPLOAD_FILES`** limit (default: `120`). For workflows that need
  roughly **50 images across gallery + variants**, set `MAX_UPLOAD_FILES=120` to leave retry
  headroom.
- When variants include a Color attribute, the form pairs each uploaded variant image with a
  **`colorKeys`** string (one per file). Multiple files may use the same color key so that a single
  variant (e.g. "Red") can receive an image gallery.
- All data is submitted as `multipart/form-data`:
  - `name.*`, `description.*`, `category.*` (multi-lang fields)
  - `price` (base product price)
  - `images` (0—10 gallery images)
  - `variants` (JSON of the variant grid with attributes/stock/price)
  - `colorImages` (0—N files, one per colored variant you want an override image for)
  - `colorKeys` (0—N strings, aligned to `colorImages`)
- **Editing** additionally sends:
  - `keepImages` — array of existing gallery URLs that should stay
  - `keepVariantImages` — array (by variant index) of URL arrays to retain for each variant

> Frontend uses a generous Axios timeout to accommodate 10-image batches. Gallery/variant files are
> appended individually; the backend processes them **sequentially** to avoid burst limits.

#### 2) Upload middleware (backend)

`backend/src/middleware/upload.middleware.js`

- Uses **Multer** with disk storage:
  - Destination: `UPLOADS_BASE_DIR` (env) — defaults to `uploads` at project root
  - Safe filename format: `${Date.now()}-<uuid>.<ext>`
  - **Type guard**: only `jpeg/jpg/png/webp` allowed
  - **Limits**: `files: MAX_UPLOAD_FILES` (env, default `120`), `fileSize: MAX_UPLOAD_MB` (env,
    default `25MB`). Admins uploading ~50 images per product should set `MAX_UPLOAD_FILES` to around
    `120` to leave space for retries/bursty uploads.
- After the request passes the middleware, `req.files` contains all uploaded files ready for
  processing.

#### 3) Image processing (Sharp)

`backend/src/utils/imageProcessor.js`

- Converts each raw upload to an optimized **WebP**:
  - `800x800` **cover** crop, then `toFormat('webp')`
  - Writes to **`uploads/products/`** (folder ensured if missing)
  - Removes the **original raw** temp file
  - Returns the **filename** (e.g., `product-<uuid>.webp`)

> Design note: We always write to `uploads/products` locally (even in production) as a **temporary
> processing step**; the system of record is **Cloudinary**.

#### 4) Cloudinary upload & cleanup

In `product.controller.js` we use a single helper:

- `processAndUpload(file)`:
  1. Calls `processProductImage(file)` → get processed WebP filename
  2. Uploads `<UPLOADS_BASE_DIR>/<PRODUCT_IMAGES_DIR>/<filename>` to Cloudinary
     - `folder: CLOUDINARY_FOLDER` (defaults to `products`)
     - `resource_type: 'image'`, `unique_filename: true`, `overwrite: false`
  3. **Deletes** the processed WebP after upload
  4. Returns `secure_url`
- **Sequential** uploads (for both gallery and variant images) avoid Cloudinary rate-limit spikes
  and make 10-image batches reliable on slower networks.
- A best-effort `cleanupTempUploads(req.files)` runs in `finally{}` to remove any Multer temps.

#### 5) Variants & image mapping

**Create** (`POST /api/products`):

- Gallery: every file in `images` → `processAndUpload` → push to `product.images`.
- Variants:
  - Parse `variants` JSON (attributes/stock/price per row).
  - Pair `colorImages` with `colorKeys` to build a `color → [urls...]` lookup. Duplicate keys are
    allowed so a single color can receive multiple images.
  - For each variant, merge uploaded URLs + existing `variant.images` (if provided) and keep the
    first 10 unique entries. The first URL is mirrored to `variant.image` for legacy flows.
  - **No gallery images?** Seed the main gallery from the flattened variant galleries so the product
    still renders images.

**Edit** (`PUT /api/products/:id`):

- **Deletes** Cloudinary images that are _not_ listed in `keepImages`.
- Uploads any new `images`, then sets `product.images = [...keepImages, ...newUrls]`.
- For variants: uploads new `colorImages` and maps them to attributes like create; otherwise uses
  the corresponding `keepVariantImages[idx]` value.

#### 6) Storage of record

- The **authoritative** product image URLs live in MongoDB (`product.images` and `variant.images`).
- The **only** persistent storage for image binaries is **Cloudinary**. The `uploads/` folder is
  **ephemeral** (processing temp).

#### 7) Environment (what to set)

- **Backend (Railway / local)**

  ```env
  # Cloudinary credentials
  CLOUDINARY_CLOUD_NAME=...
  CLOUDINARY_API_KEY=...
  CLOUDINARY_API_SECRET=...
  # Upload pipeline
  UPLOADS_BASE_DIR=uploads           # temp processing base dir (no leading ./ in prod)
  PRODUCT_IMAGES_DIR=products        # temp subfolder for processed webp files
  CLOUDINARY_FOLDER=products         # Cloudinary target folder
  MAX_UPLOAD_MB=25                   # per-file limit enforced by Multer
  ```

- **CSP** (already configured): `img-src` allows `https://res.cloudinary.com` so images render in
  all environments.

#### 8) Operational notes

- **Batch size**: 10 gallery images (plus per-variant images) are supported; uploads run
  **sequentially** to avoid rate-limit bursts.
- **Timeouts**: The frontend Axios instance uses extended timeouts for create/edit product calls to
  accommodate slower networks + large images.
- **Safety**: Any Multer temp that somehow survives processing is cleaned in a `finally{}` block.
- **Editing deletions**: When an admin removes a gallery image, the backend issues a
  `cloudinary.uploader.destroy("products/<publicId>")` before saving the new list.

> TL;DR — Drop up to 10 images (+ optional per-variant images), the backend processes to WebP,
> uploads to Cloudinary (one by one), cleans temp files, and persists Cloudinary URLs on the
> product.

## Auth & session persistence — how it works (plain language)

- Upon login/signup the server issues:
  - `access token` cookie, short-lived (e.g., 15 minutes).
  - `refresh token` cookie, long-lived (sliding, e.g., up to 30 days).
  - Server stores the refresh token value keyed by `refresh_token:<userId>:<jti>` in Redis.
  - A `session_start:<userId>` key is set in Redis to implement a **maximum sliding lifetime**
    (default 30 days).

- Token rotation:
  - `/api/auth/refresh-token` validates the cookie against Redis and atomically rotates to a new
    `jti` to prevent reuse of stolen tokens.

- Sliding window:
  - Each successful refresh or keep-alive `EXPIRE`s the `session_start:<userId>` key for another 30
    days.
  - If a user is inactive for 30 days (no refresh / keep-alive), `session_start` expires — server
    rejects further refreshes and requires re-login.

- Mobile friendliness:
  - Frontend implements mobile-aware keep-alive intervals and additional activity listeners
    (touchstart, focus) to counteract mobile background throttling.
  - The server provides both `/auth/refresh-token` and `/auth/keep-alive` paths — keep-alive
    refreshes cookie TTLs and can issue a fresh access token.

**UX rule implemented:** sessions persist for up to 30 days of inactivity; any visit/activity
refreshes the 30-day window.

---

## Order fulfilment, PDF export & canceled order labeling

- Admin UI supports exporting order details to PDF for fulfilment and label printing. Export
  includes order lines, shipping address, and order metadata.
- Canceled orders are explicitly labeled in the admin (status `cancelled` / `payment_failed` /
  `refunded`) and included in exports as such for tracking.
- PDF export utility is implemented server-side (PDF generation libraries) and exposed to the admin
  for download/printing.

---

## GDPR & user data export

- Users can request their data export via the `exportUserData` endpoint. Export includes orders,
  profile, and related user metadata in JSON format.
- Account deletion/anonymize flow:
  - When requested, PII is removed/anonymized and orders are either retained as anonymized records
    or deleted per config/policy.

- The system provides audit logging and an admin review path for data exports to comply with privacy
  regulations.

---

## Security & observability

- Cookies: `HttpOnly`, `Secure`, `SameSite=None` (when cross-site), and domain configured to allow
  cross-subdomain cookies for frontend + API.
- CSRF protection enabled for state-changing endpoints (`/api/csrf-token` endpoint available).
- Rate limiting on auth endpoints and refresh endpoint.
- Sanitization and NoSQL injection protection for request bodies and query params.
- Logging: Winston with daily rotation, sensitive data redaction, and optional Sentry forwarding for
  uncaught exceptions and errors.

---

## Testing & CI

- Tests: Jest unit & integration tests with mocks for Redis / email / cloudinary, supertest for
  route-level tests, and Cypress for frontend E2E flows.
- Coverage: Backend coverage is reported at **89.28%** (unit + integration).

### How to run tests

- **Unit + integration**:

  ```bash
  # From backend/
  npm run test
  ```

- **E2E (configured Jest e2e runner)**:

  ```bash
  # From backend/
  npm run test:e2e
  ```

- **Run coverage report**:

  ```bash
  npm run test:coverage
  ```

- **Watch tests** (dev):

  ```bash
  npm run test:watch
  ```

- **Debug tests**:

  ```bash
  npm run test:debug
  ```

> The test suite uses `NODE_ENV=test` and in many places relies on an in-memory Mongo test server
> (MongoMemoryServer) and a mocked Redis in CI. Ensure the test environment can start the in-memory
> Mongo (no other DB required for tests).

### How to test the COD refund purge cron job

You can test both the **pure purge function** and the **scheduled cron execution**:

1. **Unit test the purge behavior**

   ```bash
   npm run -w backend test -- src/tests/jobs/codRefundDetailsPurge.job.spec.js
   ```

2. **Manual dev verification (without waiting for real cron)**
   - Start backend with a fast cron expression:

     ```bash
     COD_REFUND_PURGE_CRON="* * * * *" COD_REFUND_PURGE_TZ="UTC" npm run -w backend dev
     ```

   - Recommended cron values by environment:
     - **Production:** `COD_REFUND_PURGE_CRON="15 3 * * *"` and `COD_REFUND_PURGE_TZ="UTC"` (once
       daily during off-peak hours).
     - **Development/QA:** `COD_REFUND_PURGE_CRON="* * * * *"` and `COD_REFUND_PURGE_TZ="UTC"`
       (every minute for fast validation).

   - Seed a test order with:
     - `codRefund.detailsEncrypted` set
     - `codRefund.detailsExpiresAt` in the past
   - Wait up to 1 minute and verify:
     - `codRefund.detailsEncrypted === null`
     - `codRefund.status === 'expired'`
     - `codRefund.detailsSummary` / `detailsLast4` remain available

3. **Operational check in logs**
   - Look for:
     - `COD refund details purge job started`
     - `COD refund details purge cycle completed` with `purgedCount` -
       `COD refund details purge cycle completed` with `purgedCount`

### Test status

- **Coverage:** Backend test coverage: ~89% (unit + integration). Full test suite includes
  authorization flows, stock reservation, Stripe webhooks, and end-to-end checkout scenarios —
  automated in CI with deterministic mocks for external services.

```bash
# from repo root

# run all workspaces and forward args
npm -ws run test -- --coverage --coverageReporters=text-summary

Per-package (useful while debugging)
# backend only
npm run -w backend test -- --coverage --coverageReporters=text-summary

# frontend only
npm run -w frontend test -- --coverage --coverageReporters=text-summary
```

---

## How to run locally (no Docker)

> Local setup does **not** require Docker. Use local MongoDB & Redis, or managed services.

1. Clone and install:

```bash
# from repo root
npm install
```

> Workspace-specific scripts (e.g., `npm -w backend run dev`) assume dependencies were installed
> from the repository root.

1. Create `.env` files:

- `backend/.env` — list of env variables (see the list below), **do not** commit secrets to repo.
- `frontend/.env` — `VITE_API_BASE_URL` pointing at your backend (e.g.,
  `http://localhost:5000/api`).

1. Start backend:

```bash
cd backend
npm run dev
```

1. Start frontend:

```bash
cd frontend
npm run dev
```

1. Visit the frontend URL printed by Vite (commonly `http://localhost:5173`).

---

### Local HTTPS for development (optional but recommended)

This project can run the backend locally over **HTTPS** to better mimic production (certs are stored
under `backend/cert/` in this repo). If you enable HTTPS you will typically open the backend URL
first and accept the certificate in your browser; afterwards open the frontend at
`https://localhost:5173`.

**Quick note:** you can instead open **`https://localhost:5001`** (your backend) in the browser and
accept the certificate there **before** opening **`https://localhost:5173`** — accepting the cert
for the backend often prevents browser warnings when you then visit the frontend dev server.

1. **Recommended (mkcert)** — easiest, trusted CA for local development

```bash
# install mkcert (example: macOS/Homebrew)
brew install mkcert nss
mkcert -install

# generate SAN cert files used by the server (writes to backend/cert/)
mkcert -key-file backend/cert/localhost2-key.pem \
       -cert-file backend/cert/localhost2.pem \
       localhost 127.0.0.1 ::1 www.localhost
chmod 600 backend/cert/localhost2-key.pem
```

1. **OpenSSL (manual alternative)** — create a SAN cert without mkcert

```bash
cat > san.cnf <<'EOF'
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_ca
prompt = no

[req_distinguished_name]
CN = localhost

[v3_ca]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = 127.0.0.1
DNS.3 = ::1
DNS.4 = www.localhost
EOF

openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout backend/cert/localhost2-key.pem \
  -out backend/cert/localhost2.pem \
  -days 825 -config san.cnf
chmod 600 backend/cert/localhost2-key.pem
```

1. **Trust the cert (make browser accept it)**

Use the OS-specific commands below to add the generated certificate to the system/browser trust
store so browsers accept `https://localhost:5000` and `https://localhost:5173` without warnings.

- **macOS**

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain backend/cert/localhost2.pem
```

- **Ubuntu / Debian**

```bash
sudo cp backend/cert/localhost2.pem /usr/local/share/ca-certificates/localhost2.crt
sudo update-ca-certificates
```

- **Fedora / RHEL**

```bash
sudo trust anchor --store backend/cert/localhost2.pem
```

- **Windows (Admin PowerShell)**

```powershell
# Add to LocalMachine\Root
Import-Certificate -FilePath .\backend\cert\localhost+2.pem -CertStoreLocation Cert:\LocalMachine\Root
```

If you used `mkcert` (recommended) it will automatically install the CA into the system/browser
stores for you. After trusting the cert, restart your browser.

**Restart servers:** after replacing cert files restart the backend dev server (nodemon will usually
pick changes; otherwise `npm run dev` from repo root).

---

---

## Environment variables used

**Fee engine note:** Commission rates, minimum fees, and the Pro plan Stripe Price ID are managed
via **FeeConfig records in the database**, not environment variables. Use env vars only for secrets
or scheduling. The optional `STRIPE_PRO_PRICE_ID` value is a fallback if no active FeeConfig exists;
when both are present, the DB value always wins.

**Authentication & cookies**

```
ACCESS_TOKEN_EXPIRE
ACCESS_TOKEN_SECRET
REFRESH_TOKEN_EXPIRE
SESSION_SECRET
SESSION_REDIS_PREFIX
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
STRIPE_PRO_PRICE_ID          # optional fallback if FeeConfig has no stripePriceId
SENDGRID_API_KEY
CLOUDINARY_CLOUD_NAME
CLOUDINARY_API_KEY
CLOUDINARY_API_SECRET
SENTRY_DSN
SENTRY_RELEASE
```

**Payment provider + COD refund/payout policy**

```env
PAYMENT_PROVIDER=stripe
STRIPE_PERCENT_FEE=2.9
STRIPE_FIXED_FEE_CENTS=30
STRIPE_FEE_SOURCE=balance_transaction
COD_REFUND_METHOD=bank_transfer

# Prefer COD_PAYOUT_ADAPTER.
# Backward compatibility: COD_PAYOUT_PROVIDER is accepted as a fallback alias
# only when COD_PAYOUT_ADAPTER is unset.

COD_PAYOUT_ADAPTER=chimoney
COD_PAYOUT_PROVIDER=

# Chimoney environment alignment
# Sandbox:    CHIMONEY_BASE_URL=https://api-sandbox.chimoney.io with sandbox API key
# Production: CHIMONEY_BASE_URL=https://api.chimoney.io with production API key
CHIMONEY_BASE_URL=https://api-sandbox.chimoney.io
CHIMONEY_API_KEY=replace-with-chimoney-api-key
CHIMONEY_WEBHOOK_SECRET=replace-with-chimoney-webhook-secret
CHIMONEY_PAYOUT_STATUS_PATH=/payouts/{payoutId}
COD_REFUND_DETAILS_TTL_DAYS=14
ORDER_RETURN_WINDOW_DAYS=14
COD_REFUND_DETAILS_ENCRYPTION_KEY=replace-with-strong-secret
COD_REFUND_DETAILS_ENCRYPTION_KEYS=active-key,previous-key
COD_REFUND_DEFAULT_CURRENCY=USD
COD_BANK_FIELDS=countryCode,accountHolderName,accountNumber,routingNumber,bankCode/bankId,fullName,accountHolderAddress1,accountHolderCity,accountHolderRegion,accountHolderPostal,bankName,bankAddressLine1,bankCity,bankRegion,bankPostal
COD_WALLET_FIELDS=walletProvider,walletId
COD_REFUND_PURGE_CRON=15 3 * * *
COD_REFUND_PURGE_TZ=UTC

# Reconciliation cron
COD_PAYOUT_RECONCILIATION_ENABLED=true
COD_PAYOUT_RECONCILIATION_CRON=*/20 * * * *
COD_PAYOUT_RECONCILIATION_TIMEZONE=UTC
COD_PAYOUT_RECONCILIATION_AGE_MINUTES=45

# Purge cron
COD_REFUND_PURGE_CRON=15 3 * * *
COD_REFUND_PURGE_TZ=UTC
# Production recommended: COD_REFUND_PURGE_CRON=15 3 * * * / COD_REFUND_PURGE_TZ=UTC
# Development/testing recommended: COD_REFUND_PURGE_CRON=* * * * * / COD_REFUND_PURGE_TZ=UTC
```

These are optional tuning/debug vars used by the Chimoney adapter, webhook processor, and
Production:

- `COD_REFUND_PURGE_CRON` / `COD_REFUND_PURGE_TZ` and all Chimoney tuning vars above are
  **optional** (the app has internal defaults), but are recommended to set explicitly for rs above
  are **optional** (the app has internal defaults), but are recommended to set explicitly for
  predictable operations.
- Core COD settings (adapter/currency/encryption + return/refund policy vars) remain the mandatory
  setup surface for production readiness.

**Purge + optional Chimoney tuning vars (split by environment)**

Production:

```env
COD_REFUND_PURGE_CRON=15 3 * * *
COD_REFUND_PURGE_TZ=UTC
CHIMONEY_TIMEOUT_MS=12000
CHIMONEY_MAX_RETRIES=3
CHIMONEY_RETRY_BASE_DELAY_MS=500
CHIMONEY_SOURCE_CURRENCY=USD
COD_REFUND_CHIMONEY_WEBHOOK_STORE_RAW_DEBUG=false
```

Development/testing:

```env
COD_REFUND_PURGE_CRON=* * * * *
COD_REFUND_PURGE_TZ=UTC
CHIMONEY_TIMEOUT_MS=8000
CHIMONEY_MAX_RETRIES=1
CHIMONEY_RETRY_BASE_DELAY_MS=200
CHIMONEY_SOURCE_CURRENCY=USD
COD_REFUND_CHIMONEY_WEBHOOK_STORE_RAW_DEBUG=true
```

**Production vs development guidance for the keys above**

- **Production**
  - Keep `PAYMENT_PROVIDER=stripe`.
  - Keep Stripe defaults (`STRIPE_PERCENT_FEE=2.9`, `STRIPE_FIXED_FEE_CENTS=30`) unless your Stripe
    contract differs.
  - Keep `STRIPE_FEE_SOURCE=balance_transaction` to favor settled Stripe balance transaction data.
  - Use your real payout adapter ID for `COD_PAYOUT_ADAPTER` (or keep `manual` if refunds are
    operationally handled).
  - Legacy compatibility: `COD_PAYOUT_PROVIDER` is still read as a fallback alias if
    `COD_PAYOUT_ADAPTER` is unset.
  - Keep `ORDER_RETURN_WINDOW_DAYS=14` unless your customer policy differs.
  - Keep `COD_REFUND_DETAILS_TTL_DAYS=14` (or your compliance SLA value).
  - Set `COD_REFUND_DETAILS_ENCRYPTION_KEY` to a strong random secret (required).
  - Set `COD_REFUND_DETAILS_ENCRYPTION_KEYS` with at least two values during key rotation:
    `new_active_key,previous_key`.
  - Keep `COD_REFUND_PURGE_CRON="15 3 * * *"` (daily low-traffic sweep at 03:15 UTC is a safe
    default).
  - Keep `COD_REFUND_PURGE_TZ=UTC` unless your operations team has a required timezone.
    - Use the **production baseline block below** and pair production `CHIMONEY_BASE_URL` with
      production `CHIMONEY_API_KEY`.
- **Development**
  - Keep `PAYMENT_PROVIDER=stripe` (unless testing a mocked provider).
  - Keep the same Stripe fee defaults so local calculations match production expectations.
  - Keep `STRIPE_FEE_SOURCE=balance_transaction`.
  - Keep `COD_PAYOUT_ADAPTER=manual` unless you have a dedicated sandbox bank adapter.
  - Keep `ORDER_RETURN_WINDOW_DAYS=14` for realistic behavior, or shorten for edge-case testing.
  - You may shorten `COD_REFUND_DETAILS_TTL_DAYS` (for example `1`) for faster expiry-path testing.
  - Set `COD_REFUND_DETAILS_ENCRYPTION_KEY` (required outside `NODE_ENV=test`); for local dev this
    can be a stable non-production secret.
  - Set `COD_REFUND_DETAILS_ENCRYPTION_KEYS` to the same value as primary unless you're testing key
    rotation.
  - For quick local purge verification use `COD_REFUND_PURGE_CRON="* * * * *"` (every minute).
  - Keep `COD_REFUND_PURGE_TZ=UTC` for deterministic tests.
  - Use the **development/testing baseline block below** and keep sandbox `CHIMONEY_BASE_URL` paired
    with sandbox `CHIMONEY_API_KEY`.

Recommended COD baseline for **production**:

```env
COD_PAYOUT_ADAPTER=chimoney
# Optional backward-compat alias only; safe to leave unset when ADAPTER is set.
# COD_PAYOUT_PROVIDER=chimoney
COD_REFUND_DETAILS_TTL_DAYS=14
ORDER_RETURN_WINDOW_DAYS=14
COD_REFUND_DETAILS_ENCRYPTION_KEY=replace-with-strong-secret
COD_REFUND_DETAILS_ENCRYPTION_KEYS=active-key,previous-key
COD_REFUND_DEFAULT_CURRENCY=USD
COD_BANK_FIELDS=countryCode,accountHolderName,accountNumber,routingNumber,bankCode/bankId,fullName,accountHolderAddress1,accountHolderCity,accountHolderRegion,accountHolderPostal,bankName,bankAddressLine1,bankCity,bankRegion,bankPostal
COD_WALLET_FIELDS=walletProvider,walletId
COD_REFUND_PURGE_CRON=15 3 * * *
COD_REFUND_PURGE_TZ=UTC
```

Recommended COD baseline for **development/testing**:

```env
COD_PAYOUT_ADAPTER=chimoney
# Optional backward-compat alias only; safe to leave unset when ADAPTER is set.
# COD_PAYOUT_PROVIDER=chimoney
COD_REFUND_DETAILS_TTL_DAYS=1
ORDER_RETURN_WINDOW_DAYS=14
COD_REFUND_DETAILS_ENCRYPTION_KEY=dev_cod_refund_primary_key
COD_REFUND_DETAILS_ENCRYPTION_KEYS=dev_cod_refund_primary_key
COD_REFUND_DEFAULT_CURRENCY=USD
COD_BANK_FIELDS=countryCode,accountHolderName,accountNumber,routingNumber,bankCode/bankId,fullName,accountHolderAddress1,accountHolderCity,accountHolderRegion,accountHolderPostal,bankName,bankAddressLine1,bankCity,bankRegion,bankPostal
COD_WALLET_FIELDS=walletProvider,walletId
COD_REFUND_PURGE_CRON=* * * * *
COD_REFUND_PURGE_TZ=UTC
```

- These values are fallback **field-name schema keys** for customer payout details.
- Primary source is the dynamic beneficiary rules endpoint
  (`GET /api/cod-refund-config/beneficiary-rules/:countryCode`).
- Use `COD_BANK_FIELDS` / `COD_WALLET_FIELDS` only when dynamic provider rules are unavailable.
- Do **not** place platform bank credentials in these vars.
- Currency is controlled by `COD_REFUND_DEFAULT_CURRENCY`.

**Encryption key notes (`COD_REFUND_DETAILS_ENCRYPTION_KEY` / `...KEYS`)**

- `COD_REFUND_DETAILS_ENCRYPTION_KEY` is the required **primary** key used for new encryption.
- `COD_REFUND_DETAILS_ENCRYPTION_KEYS` is an optional comma-separated rotation list used for
  decryption compatibility.
- In non-test environments, the backend now fails fast at startup if no COD refund key material is
  configured.
- `APP_SECRET`, `JWT_SECRET`, and hardcoded dev fallbacks are **not** used for COD refund encryption
  anymore.
- Encryption uses the primary key (`COD_REFUND_DETAILS_ENCRYPTION_KEY`), and decryption attempts all
  configured rotation keys.
- Rotation example:
  1. Current: `COD_REFUND_DETAILS_ENCRYPTION_KEY=old_key`
  2. Deploy with:
     - `COD_REFUND_DETAILS_ENCRYPTION_KEY=new_key`
     - `COD_REFUND_DETAILS_ENCRYPTION_KEYS=new_key,old_key`
  3. New writes use `new_key`, old data remains decryptable
  4. After migration window, remove `old_key`

**Using Stripe sandbox (Test mode) for development**

- In Stripe Dashboard, toggle **Test mode** and copy: `STRIPE_SECRET_KEY` (starts with
  `sk_test_...`) and `STRIPE_WEBHOOK_SECRET` (`whsec_...`) from your test webhook endpoint.
- Keep `STRIPE_FEE_SOURCE=balance_transaction` so fee numbers come from Stripe balance transaction
  records when available.
- `PAYMENT_PROVIDER`/Stripe keys control card payments; COD refund destination fields are still your

**Subscription renewal policy (server-side)**

```
SUBSCRIPTION_RENEWAL_ENABLED         # default true; set to false to disable the cron job
SUBSCRIPTION_RENEWAL_CRON            # default "0 * * * *" (hourly sweep)
SUBSCRIPTION_RENEWAL_TZ              # default "UTC"
SUBSCRIPTION_RENEWAL_LOOKAHEAD_DAYS  # default 3 (renew before expiry)
SUBSCRIPTION_RENEWAL_MAX_RETRIES     # default 3
SUBSCRIPTION_RENEWAL_RETRY_DELAY_HOURS # default 12 (space retries)
SUBSCRIPTION_RENEWAL_DOWNGRADE_GRACE_HOURS # default 24 (grace before downgrade)
```

**Settlement scheduler policy (server-side)**

```
SETTLEMENT_SCHEDULER_ENABLED         # default true; set false to disable scheduler cron registration
SETTLEMENT_PERIOD_DAYS               # default 15 (window size per month, in days)
SETTLEMENT_HOLD_DAYS                 # default 0 (days after period end before eligible)
SETTLEMENT_CRON                      # default "0 0 * * *" (daily sweep)
SETTLEMENT_TZ                        # optional, default "UTC"; admin/UI display formatting only
SETTLEMENT_SCHEDULER_TZ              # optional, default "UTC"; cron scheduler timezone
SETTLEMENT_CURRENCY                  # optional, default "USD"
SETTLEMENT_AUTO_EXECUTE              # default false; executes newly created batches in same cron cycle
SETTLEMENT_LOCK_KEY                  # default "lock:settlement_scheduler" (Redis distributed lock key)
SETTLEMENT_LOCK_TTL_SECONDS          # default 120 (lock TTL in seconds)
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY # default "skip_cycle"; allowed: skip_cycle | run_unlocked
```

**Settlement Timezone Policy**

- Settlement boundaries are fixed to UTC midnight/UTC day-end and persisted in UTC.
- Sellers across timezones see the same UTC-aligned settlement periods.
- `SETTLEMENT_TZ` is display-only for admin/UI conversion and does not affect boundary calculation
  \_TZ` is display-only for admin/UI conversion and does not affect boundary calculation or database
  persistence.

**Recommended env presets**

Production-like:

```env
SETTLEMENT_PERIOD_DAYS=15
SETTLEMENT_HOLD_DAYS=3
SETTLEMENT_CRON="0 0 * * *"      # once daily at 00:00 UTC
SETTLEMENT_TZ="Africa/Cairo"  # display formatting only; boundaries remain UTC
SETTLEMENT_SCHEDULER_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
SETTLEMENT_AUTO_EXECUTE=false
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler
SETTLEMENT_LOCK_TTL_SECONDS=300
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=skip_cycle
```

Local/manual testing (fast feedback):

```env
SETTLEMENT_PERIOD_DAYS=1
SETTLEMENT_HOLD_DAYS=0
SETTLEMENT_CRON="*/2 * * * *"    # every 2 minutes
SETTLEMENT_TZ="Africa/Cairo"  # display formatting only; boundaries remain UTC
SETTLEMENT_SCHEDULER_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
SETTLEMENT_AUTO_EXECUTE=false
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:dev
SETTLEMENT_LOCK_TTL_SECONDS=120
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=run_unlocked
```

> For local manual verification, keep hold days at `0` and run a frequent cron cadence
> (`*/1`-`*/5 * * * *`) so batches become eligible quickly.

**Migration note (timezone behavior change)**

- `SETTLEMENT_TZ` is now treated as **display metadata only** for admin/UI rendering (UTC conversion
  for display).
- Cron scheduling timezone control moved to `SETTLEMENT_SCHEDULER_TZ` (default `UTC`).
- Settlement period calculations (`periodStart` / `periodEnd`) remain UTC-only.
- If an existing deployment previously relied on `SETTLEMENT_TZ` to shift cron execution windows,
  set `SETTLEMENT_SCHEDULER_TZ` to the prior value to preserve behavior.

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
RATE_WINDOW_MS           # shared limiter window in ms (default 3600000)
SELLER_REQ_LIMIT         # legacy seller write cap alias (fallback)
SELLER_READ_LIMIT        # seller GET/HEAD budget (default 1000/hr)
SELLER_WRITE_LIMIT       # seller POST/PATCH/DELETE budget (default 1000/hr)
ADMIN_REQ_LIMIT          # legacy admin write cap alias (fallback)
ADMIN_READ_LIMIT         # admin GET/HEAD budget (default 10000/hr)
ADMIN_WRITE_LIMIT        # admin POST/PATCH/DELETE budget (default 10000/hr)
VITE_TAX_RATE
VITE_SHIPPING_COST
VITE_FEATURE_CATEGORY_TREE_V2
VITE_FEATURE_NAV_CATEGORY_MENU_V2
UPLOADS_BASE_DIR         # temp processing base dir (e.g., "uploads")
PRODUCT_IMAGES_DIR       # temp subdir for processed webp (e.g., "products")
CLOUDINARY_FOLDER        # Cloudinary folder for final assets (e.g., "products")
MAX_UPLOAD_MB            # Multer per-file size limit in MB (default 25)
MAX_UPLOAD_FILES         # Multer max file count (default 120; raise for large galleries)
COOKIE_DOMAIN
USE_HTML_REDIRECT

```

Remember to configure these per environment (Railway, Vercel, local).

---

### Example `.env` for local development (copy & fill)

Below are **copy-ready** example files for local development. Put the backend block in
`backend/.env` and the frontend block in `frontend/.env`.

**backend/.env (local dev example)**

```env
# Server
NODE_ENV=development
PORT=5001
CLIENT_URL=https://localhost:5173
SERVER_URL=https://localhost:5001
DEV_ORIGIN=https://localhost:5173,https://localhost:3000
COOKIE_DOMAIN=
ENFORCE_HTTPS=false
TRUST_PROXY=false
FORCE_SECURE_COOKIES=false

# Database
MONGO_URI=mongodb://localhost:27017/your-db-name

# Redis (Optional)
REDIS_DRIVER=rest
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=
# Or use TCP locally instead of REST:
# REDIS_URL=redis://localhost:6379

# Sessions (optional — falls back to MemoryStore)
SESSION_REDIS_URL=redis://127.0.0.1:6379
SESSION_REDIS_PREFIX=session:dev:

# JWT / Auth
ACCESS_TOKEN_SECRET=changeme_access_secret
REFRESH_TOKEN_SECRET=changeme_refresh_secret
SESSION_SECRET=changeme_session_secret

ACCESS_TOKEN_EXPIRE=15m
REFRESH_TOKEN_EXPIRE=7d
MAX_SESSION_LIFETIME_DAYS=30
CSRF_SECRET=changeme_csrf
COOKIE_SECRET=changeme_cookie_secret
JWT_SECRET=changeme_jwt_secret
DEBUG_REFRESH=false

# API rate limits
RATE_WINDOW_MS=3600000
SELLER_REQ_LIMIT=1000
SELLER_READ_LIMIT=1000
SELLER_WRITE_LIMIT=1000
ADMIN_REQ_LIMIT=10000
ADMIN_READ_LIMIT=10000
ADMIN_WRITE_LIMIT=10000

# OAuth Providers
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_CALLBACK_URL=http://localhost:5001/api/auth/google/callback
GOOGLE_LINK_CALLBACK_URL=http://localhost:5001/api/auth/google/link/callback
FACEBOOK_APP_ID=
FACEBOOK_APP_SECRET=
FACEBOOK_CALLBACK_URL=http://localhost:5001/api/auth/facebook/callback
FACEBOOK_LINK_CALLBACK_URL=http://localhost:5001/api/auth/facebook/link/callback

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
# Optional fallback only; FeeConfig in DB takes precedence
STRIPE_PRO_PRICE_ID=

# Payment provider + COD policy
PAYMENT_PROVIDER=stripe
STRIPE_PERCENT_FEE=2.9
STRIPE_FIXED_FEE_CENTS=30
STRIPE_FEE_SOURCE=balance_transaction
COD_REFUND_METHOD=bank_transfer
COD_PAYOUT_ADAPTER=chimoney
# Optional backward-compat alias only; safe to leave unset when ADAPTER is set.
# COD_PAYOUT_PROVIDER=chimoney
COD_REFUND_DETAILS_TTL_DAYS=1
ORDER_RETURN_WINDOW_DAYS=14
COD_REFUND_DETAILS_ENCRYPTION_KEY=dev_cod_refund_primary_key
COD_REFUND_DETAILS_ENCRYPTION_KEYS=dev_cod_refund_primary_key
COD_REFUND_DEFAULT_CURRENCY=USD
COD_BANK_FIELDS=countryCode,accountHolderName,accountNumber,routingNumber,bankCode/bankId,fullName,accountHolderAddress1,accountHolderCity,accountHolderRegion,accountHolderPostal,bankName,bankAddressLine1,bankCity,bankRegion,bankPostal
COD_WALLET_FIELDS=walletProvider,walletId
COD_REFUND_PURGE_CRON=* * * * *
COD_REFUND_PURGE_TZ=UTC

# Subscription renewal policy (server-side)
SUBSCRIPTION_RENEWAL_ENABLED=true
SUBSCRIPTION_RENEWAL_CRON="0 * * * *"
SUBSCRIPTION_RENEWAL_TZ="UTC"
SUBSCRIPTION_RENEWAL_LOOKAHEAD_DAYS=3
SUBSCRIPTION_RENEWAL_MAX_RETRIES=3
SUBSCRIPTION_RENEWAL_RETRY_DELAY_HOURS=12
SUBSCRIPTION_RENEWAL_DOWNGRADE_GRACE_HOURS=24

# Settlement scheduler policy (server-side)
SETTLEMENT_SCHEDULER_ENABLED=true
SETTLEMENT_PERIOD_DAYS=15
SETTLEMENT_HOLD_DAYS=3
SETTLEMENT_CRON="0 0 * * *"
SETTLEMENT_TZ="Africa/Cairo"  # display formatting only; boundaries remain UTC
SETTLEMENT_SCHEDULER_TZ="UTC"
SETTLEMENT_CURRENCY="USD"
SETTLEMENT_AUTO_EXECUTE=false
SETTLEMENT_LOCK_KEY=lock:settlement_scheduler:dev
SETTLEMENT_LOCK_TTL_SECONDS=120
SETTLEMENT_LOCK_REDIS_FALLBACK_POLICY=run_unlocked

# Local testing override (manual):
# SETTLEMENT_PERIOD_DAYS=1
# SETTLEMENT_HOLD_DAYS=0
# SETTLEMENT_CRON="*/2 * * * *"

# Email Service (SendGrid)
SENDGRID_API_KEY=
SENDGRID_FROM_EMAIL=your-email@example.com
SENDGRID_FROM_NAME="Ecommerce store"
SUPPORT_EMAIL=your-support@example.com

# Cloudinary
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

# Sentry
SENTRY_DSN=
SENTRY_RELEASE=backend@local

# File uploads / pricing
UPLOADS_BASE_DIR=uploads # use bare path (no ./) to match prod
PRODUCT_IMAGES_DIR=products
CLOUDINARY_FOLDER=products
TAX_RATE=0.14
SHIPPING_COST=70
MAX_UPLOAD_FILES=120 # allows ~50 gallery + variant images with retry headroom
# optionally raise this if you use very large originals
MAX_UPLOAD_MB=25

# Local HTTPS cert usage flags (optional)
USE_LOCAL_HTTPS=true
LOCAL_CERT_PATH=./cert/localhost2.pem
LOCAL_KEY_PATH=./cert/localhost2-key.pem
```

**frontend/.env (local dev example for Vite)**

```env
VITE_API_BASE_URL=https://localhost:5001
VITE_CLIENT_BASE_URL=https://localhost:5173
VITE_KEEPALIVE_INTERVAL_MS=300000
VITE_STRIPE_PUBLISHABLE_KEY=
VITE_SENTRY_DSN=
VITE_SENTRY_RELEASE=frontend@local
VITE_SENTRY_TRACES_SAMPLE_RATE=0.1
VITE_TAX_RATE=0.14
VITE_SHIPPING_COST=70
```

Notes:

- Keep secrets out of source control — commit only `.env.example` with placeholders.
- If you run backend locally over HTTPS (see "Local HTTPS for development" above), set
  `USE_LOCAL_HTTPS=true` and ensure the `LOCAL_CERT_PATH` / `LOCAL_KEY_PATH` point to the cert files
  (or let server code load `backend/cert/localhost+2.pem` by default).

---

### Deployment platform variables (Railway / Vercel)

The README already lists the primary deployment variables in groups. To be explicit:

- **Railway / server-side**: populate the backend keys shown in `backend/.env` (MONGO*URI, REDIS,
  STRIPE*\_, SENDGRID\_\_, SENTRY\_\*, CALLBACK URLs, COOKIE_DOMAIN, etc.) in Railway's environment
  variables panel.
- **Vercel / frontend**: populate the `VITE_*` variables (API base, stripe publishable key, Sentry
  keys, client base URL and any feature flags).

## Redis (Optional)

Redis backs refresh tokens, stock reservations, cart backups, and other cache-like features. The
Redis helpers live in `backend/src/lib/cache/*` and expose a **driver abstraction** while banning
raw Redis usage in controllers/services.

### Architecture overview

**1) Cache + app state (driver-based)**

- `backend/src/lib/cache/index.js` chooses a driver:
  - `REDIS_DRIVER=rest|tcp|auto` (default: `auto`)
  - `rest` uses Upstash REST (`UPSTASH_REDIS_REST_URL` + `UPSTASH_REDIS_REST_TOKEN`)
  - `tcp` uses ioredis TLS (`REDIS_URL` or `UPSTASH_REDIS_URL`)
- Controllers/services should only use:
  - `cacheGetJSON`, `cacheSetJSON`, `cacheSetNXJSON`, `cacheDel`, `cacheKeys`
  - and/or higher-level helpers (e.g. `authCache.js`, stock reservation helpers)
- Benefit: controllers don’t depend on vendor/library details (Upstash REST vs ioredis TCP). TTL,
  key naming, JSON serialization, retries, and logging live in one place. Easier tests + migrations.

**2) Express sessions (explicit, separate, TCP-only)**

- Sessions use a dedicated client: `backend/src/lib/sessionRedisClient.js`
- Sessions are **explicit** and **never auto-use** cache Redis settings:
  - Sessions use **ONLY** `SESSION_REDIS_URL`
  - No implicit `localhost` default
  - No fallback to `REDIS_URL` / `UPSTASH_REDIS_URL`
- Behavior:
  - If `SESSION_REDIS_URL` is missing → MemoryStore (OK in dev)
  - If `SESSION_REDIS_URL` is set but unreachable/wrong → fail fast in logs and still fall back to
    MemoryStore (keeps dev usable; production should treat this as misconfig)

> Important nuance: REST is great for cache-style operations. For strict atomicity (Lua / multi /
> strong concurrency guarantees), TCP is preferred in production. That’s the intended split:
> **Railway/Prod = TCP**, dev can use REST for cache-read-heavy paths if TCP is blocked.

### Environment variables

**Cache / driver-based Redis**

- `REDIS_DRIVER=rest|tcp|auto`
- REST:
  - `UPSTASH_REDIS_REST_URL`
  - `UPSTASH_REDIS_REST_TOKEN`
- TCP:
  - `REDIS_URL` (or `UPSTASH_REDIS_URL`) as `rediss://default:<password>@<host>:6379`

**Sessions (TCP-only)**

- `SESSION_REDIS_URL=redis://127.0.0.1:6379` (local) OR `rediss://...` (managed)
- `SESSION_REDIS_PREFIX` (recommended)
  - `session:dev:` for dev
  - `session:prod:` for prod

### Redis configuration examples

**Local development (cache via Upstash REST, sessions via local Redis):**

```env
# Cache/auth/stock/etc
REDIS_DRIVER=rest
UPSTASH_REDIS_REST_URL=https://your-upstash-endpoint.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-upstash-rest-token

# Sessions (local)
SESSION_REDIS_URL=redis://127.0.0.1:6379
SESSION_REDIS_PREFIX=session:dev:
```

**Production (Railway, Upstash TCP with TLS):**

```env
NODE_ENV=production

# Cache/auth/stock/etc
REDIS_DRIVER=tcp
REDIS_URL=rediss://default:<password>@<your-upstash-host>:6379

# Sessions (managed TCP)
SESSION_REDIS_URL=rediss://default:<password>@<your-upstash-host>:6379
SESSION_REDIS_PREFIX=session:prod:
```

### Redis debugging quick commands (WSL/Ubuntu)

#### Install Redis locally (dev sessions)

```bash
sudo apt update
sudo apt install -y redis-server redis-tools
```

#### Start Redis

If your WSL has systemd enabled:

```bash
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

If systemd is NOT enabled (common on WSL):

```bash
sudo service redis-server start
```

#### Verify it’s running

```bash
redis-cli ping
# PONG
```

#### Useful inspection commands

```bash
# List keys (small dev only). Prefer SCAN in real environments.
redis-cli --scan --pattern "session:*"
redis-cli --scan --pattern "session:dev:*"

# Check TTL / value (debugging)
redis-cli ttl "session:dev:somekey"
redis-cli get "session:dev:somekey"

# Delete all dev session keys (careful!)
redis-cli --scan --pattern "session:dev:*" | xargs -r redis-cli del

# Check TTL for session_start
TTL session_start:<userId>

# Get refresh token stored server-side (debug only)
GET refresh_token:<userId>:<jti>

# Reservation keys
KEYS stock_reservation:_:<orderId>:_

# Cart backup
GET cart_backup:<userId>
```

Then set in `backend/.env`:

```env
SESSION_REDIS_URL=redis://127.0.0.1:6379
SESSION_REDIS_PREFIX=session:dev:
```

---

## Screenshots

> Admin panel screenshots (admin access required)

### Orders & Fulfilment

![Orders — Paginated list](./docs/screenshots/admin_panel_orders_pagination.png) _Orders list with
pagination._

![Orders — Dark mode list](./docs/screenshots/admin_panel_orders_dark.png) _Orders list (dark
theme)._

![Orders — Multi-order status update (light)](./docs/screenshots/admin_panel_orders_multi_order_status_update_light.png)
_Bulk update order statuses (multi-select)._

![Order — Expanded single order (light)](./docs/screenshots/admin_ordresList_expanded_order.png)
_Expanded order row showing details (light theme)._

![Order — Expanded & status change (light)](./docs/screenshots/adm_ordrs_expanded_order_statuschange_light.png)
_Expanded order with status change controls (light theme)._

![Order — Expanded & status change (dark)](./docs/screenshots/admin_expandedOrdr_statuschange_dark.png)
_Expanded order with status change controls (dark theme)._

![Order — Expanded customer view (dark)](./docs/screenshots/admin_expOrdr_custView_dark.png)
_Expanded order with customer view (dark theme)._

---

### Create Product & Variants

![Create product — Admin panel](./docs/screenshots/admin_panel_create_product.png) _Admin UI —
Create product (base fields)._

![Create product — Variants editor](./docs/screenshots/admin_panel_create_product_variants.png)
_Admin UI — Create product with variants (light theme)._

![Create product — Variants editor (dark)](./docs/screenshots/admin_panel_create_product_variants_dark.png)
_Admin UI — Create product with variants (dark theme)._

---

### Analytics & Campaigns

![Admin — Analytics dashboard (dark)](./docs/screenshots/admin_panel_analytics_dark.png) _Admin
analytics dashboard — sales, users, revenue._

![Admin — Campaigns](./docs/screenshots/admin_campaigns.png) _Marketing campaigns / mailing list
management._

---

### Emails

![Email Campaign Subscription Confirmation](./docs/screenshots/email-campaign-subscription-confirmation-email.png)
_Email campaign subscription confirmation email._

![Emails Sent to Inbox (Not Spam)](./docs/screenshots/emails-sent-to-inbox-not-spam.png) _Emails
successfully delivered to inbox (not marked as spam)._

![Order Confirmation Email (COD with Variants)](./docs/screenshots/order-confirmation-email-cashOnDelivery-some-products-with-variants.png)
_Order confirmation email for cash-on-delivery orders with product variants._

![Order Confirmation Email (Paid with Variants)](./docs/screenshots/order-confirmation-email-paid-order-products-have%20variants.png)
_Order confirmation email for paid orders with product variants._

![Order Cancellation Confirmation](./docs/screenshots/order_cancelation_confirmation_email.png)
_Order cancellation confirmation email._

![Reset Password Link Email](./docs/screenshots/reset_password_link_email.png) _Password reset link
email._

![Verify Email](./docs/screenshots/local_sigunp_verify_email.png) _Local signup Email verification
request email._

---

### Mobile

![Mobile — Home light](./docs/screenshots/mobile-app-homepage-light.jpg) _Native home screen with
featured sliders._

![Mobile — Category grid](./docs/screenshots/mobile-app-category-page-dark.jpg) _Category browsing
with hero banners._

![Mobile — Cart (light)](./docs/screenshots/mobile-app-cart-page-light.jpg) _Cart summary with
variant chips and quantity controls._

![Mobile — Order history](./docs/screenshots/mobile-app-order-history-page-dark.jpg) _Order history
with status tracking._

![Mobile — Profile tools](./docs/screenshots/mobile-app-profile-page-delete-acount&exportData-section-light.jpg)
_Profile management, GDPR export, and delete account actions._

## Commercial license (proprietary) & selling notes

Check LICENSE_PROPRIETARY.txt

**Selling notes**:

- Deliverables to include: source code, deployment scripts, environment setup, 30 days of
  post-delivery support, documentation, and optional maintenance plan.
- For buyers: the app is horizontally scalable — to increase capacity, upgrade MongoDB/Redis/hosting
  plans and scale server replicas behind a load balancer.

---

## Demo credentials (web & mobile)

All credentials are scoped to the staging environment (`https://www.ahmedmonib-eshop-demo.com` and
the Expo mobile client). Update or rotate as needed before sharing broadly.

| Role            | Email / Username      | Password |
| --------------- | --------------------- | -------- |
| Regular shopper | `amonib831@gmail.com` | `123456` |

---

## Additional documentation

Extra reference material that complements this README:

- [`frontend/README.md`](frontend/README.md) — frontend-specific architecture, scripts, and tooling.
- [`mobile/dev-setup.md`](mobile/dev-setup.md) — Expo environment setup and troubleshooting.
- [`mobile/custom-dev-setup.md`](mobile/custom-dev-setup.md) — deeper dive into native module
  development and custom client rebuilds.
- [`mobile/docs/branding.md`](mobile/docs/branding.md) — how to apply bespoke branding assets in the
  Expo app.
- [`mobile/docs/build-and-install.md`](mobile/docs/build-and-install.md) — reproducible build and
  installation steps for QA and stakeholders.
- [`mobile/docs/android-release-play-store.md`](mobile/docs/android-release-play-store.md) — end-to
  end checklist for preparing a Play Store build.
- [`mobile/docs/internalTesting-and-productionAABinstall.md`](mobile/docs/internalTesting-and-productionAABinstall.md)
  — AAB sideload/testing guide.
- [`mobile/docs/mobile-checkout-flow.md`](mobile/docs/mobile-checkout-flow.md) — annotated mobile
  checkout sequence with deeplink notes.
- [`mobile/certs/README.md`](mobile/certs/README.md) — explains the local HTTPS certificates and how
  to rotate them.
- [`docs/DAISYUI-THEME.md`](docs/DAISYUI-THEME.md) — shared DaisyUI theming internals for web and
  mobile.
- [`docs/THEME.md`](docs/THEME.md) — native token architecture for both platforms.
- [`eslint-cheatsheet.md`](eslint-cheatsheet.md) — linting reference for contributors.
- [`prettier-configuration-cheatSheet.md`](prettier-configuration-cheatSheet.md) — Prettier usage
  guide tailored to this repo.

---

## Contact / commercial enquiries

For purchase, licensing, deployment assistance, or a demo with admin credentials contact:
**[ahmedmounib2@gmail.com](mailto:ahmedmounib2@gmail.com)**

---
