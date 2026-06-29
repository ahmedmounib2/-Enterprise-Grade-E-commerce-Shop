# Vexflare — Enterprise-Grade E-commerce Shop

> Production-ready, enterprise-grade e-commerce storefront (React + Vite frontend, Node.js + Express
> backend) with:
>
> - Secure rotated refresh tokens and sliding 30-day sessions (mobile-aware keep-alive)
> - Product variants, per-variant stock & pricing, cart reservation and automatic stock restore on
>   cancel/expire
> - Stripe Checkout integration with robust webhook handling (PaymentIntent verification, refunds,
>   expired sessions)
> - Order export (PDF) & fulfilment tooling, GDPR data export, Sentry observability, and 90.02%
>   backend test line coverage across 3,333 automated tests.

---

## Project Rebranding

This project was originally published under:

<https://ahmedmonib-eshop-demo.com>

and was later rebranded to:

<https://vexflare.com>

Some screenshots and historical references may still contain the previous domain.

---

## Tooling

- [AI-Assisted Design Workflow](docs/CLAUDE-FIGMA-SETUP.md) — Claude Desktop + Figma MCP integration
  for design creation and design-to-code conversion.
- [AI Development Guidelines](CLAUDE.md) — Structured ruleset for AI-assisted development (commit
  hygiene, i18n, theming, approval gates, Figma workflow).

---

## Table of contents

- [Vexflare — Enterprise-Grade E-commerce Shop](#vexflare--enterprise-grade-e-commerce-shop)
  - [Project Rebranding](#project-rebranding)
  - [Tooling](#tooling)
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
      - [Onboarding approval and product-readiness flowchart](#onboarding-approval-and-product-readiness-flowchart)
      - [Entry points \& UX design](#entry-points--ux-design)
      - [Data model design (Seller + Store)](#data-model-design-seller--store)
      - [API flow (step-by-step)](#api-flow-step-by-step)
      - [State machine (clear, enforced transitions)](#state-machine-clear-enforced-transitions)
      - [Security \& ownership controls](#security--ownership-controls)
      - [Post-approval seller experience](#post-approval-seller-experience)
      - [Operational notes](#operational-notes)
      - [Seller withdrawal](#seller-withdrawal)
      - [Audit trail collections](#audit-trail-collections)
      - [Seller bank-update review lifecycle (pending/approved/rejected + seller cancel)](#seller-bank-update-review-lifecycle-pendingapprovedrejected--seller-cancel)
      - [Seller dashboard behavior for bank-update reviews](#seller-dashboard-behavior-for-bank-update-reviews)
      - [Bank-update review lifecycle flowchart](#bank-update-review-lifecycle-flowchart)
      - [Backend contract for bank-update records](#backend-contract-for-bank-update-records)
      - [Bank-update API routes (seller + admin)](#bank-update-api-routes-seller--admin)
      - [KYC document handling](#kyc-document-handling)
    - [Seller store settings, storefront policies, and admin review flow](#seller-store-settings-storefront-policies-and-admin-review-flow)
      - [Seller settings: editable fields \& review gates](#seller-settings-editable-fields--review-gates)
      - [What happens when a seller updates policies or branding](#what-happens-when-a-seller-updates-policies-or-branding)
      - [Policy update + immutable order snapshot flowchart](#policy-update--immutable-order-snapshot-flowchart)
      - [Admin review workflow (store profile + policy updates)](#admin-review-workflow-store-profile--policy-updates)
      - [Storefront policy rendering (public store page + checkout)](#storefront-policy-rendering-public-store-page--checkout)
      - [Pause store (Hide All) behavior](#pause-store-hide-all-behavior)
    - [Seller store closure lifecycle](#seller-store-closure-lifecycle)
      - [Phase 1 — Eligibility check \& closure request](#phase-1--eligibility-check--closure-request)
      - [Phase 2 — Immediate deactivation (soft close)](#phase-2--immediate-deactivation-soft-close)
      - [Phase 3 — Waiting period \& permanent deletion (finalizer job)](#phase-3--waiting-period--permanent-deletion-finalizer-job)
      - [Admin override](#admin-override)
      - [Closure email notifications](#closure-email-notifications)
      - [i18n](#i18n)
    - [Seller product creation \& admin moderation (end-to-end)](#seller-product-creation--admin-moderation-end-to-end)
      - [Flow overview (from seller draft to storefront)](#flow-overview-from-seller-draft-to-storefront)
      - [Product creation + moderation pipeline flowchart](#product-creation--moderation-pipeline-flowchart)
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
      - [Payment grace period](#payment-grace-period)
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
      - [Store reviews \& rating aggregate](#store-reviews--rating-aggregate)
  - [Deployment surfaces \& release workflow](#deployment-surfaces--release-workflow)
    - [Frontend (Vercel)](#frontend-vercel)
    - [Backend (Railway Docker image)](#backend-railway-docker-image)
    - [Mobile app (Expo custom dev client)](#mobile-app-expo-custom-dev-client)
  - [Mobile deep-link flow overview](#mobile-deep-link-flow-overview)
    - [Payment return (Stripe checkout) — direct 303 redirect](#payment-return-stripe-checkout--direct-303-redirect)
    - [OAuth login (Google/Facebook) — in-process interception](#oauth-login-googlefacebook--in-process-interception)
    - [Why two strategies?](#why-two-strategies)
    - [Supported return schemes](#supported-return-schemes)
    - [Key files](#key-files)
    - [Testing matrix](#testing-matrix)
    - [Password reset deep-link flow](#password-reset-deep-link-flow)
    - [Other deep-linked flows](#other-deep-linked-flows)
  - [Mobile app architecture \& features](#mobile-app-architecture--features)
    - [Key capabilities](#key-capabilities)
    - [Native integrations](#native-integrations)
    - [Mobile documentation](#mobile-documentation)
    - [Mobile screenshots](#mobile-screenshots)
  - [What this repo contains](#what-this-repo-contains)
  - [Key features (summary)](#key-features-summary)
  - [Per-store shipping \& admin delivery regions](#per-store-shipping--admin-delivery-regions)
    - [Per-store rate configuration (`store.shipping`)](#per-store-rate-configuration-storeshipping)
    - [Seller shipping UI](#seller-shipping-ui)
    - [Admin-managed delivery regions (operating countries + subdivisions)](#admin-managed-delivery-regions-operating-countries--subdivisions)
    - [Endpoints \& seed](#endpoints--seed)
    - [Ship-from address requirement](#ship-from-address-requirement)
  - [Category-based tax engine (internal rules + Avalara)](#category-based-tax-engine-internal-rules--avalara)
    - [Internal `TaxRule` engine](#internal-taxrule-engine)
    - [TaxProvider interface, Avalara, and the fallback chain](#taxprovider-interface-avalara-and-the-fallback-chain)
    - [Order tax audit](#order-tax-audit)
    - [Endpoints \& UI](#endpoints--ui)
  - [Address autocomplete \& draggable-pin checkout map](#address-autocomplete--draggable-pin-checkout-map)
    - [Google Places autocomplete via a backend proxy](#google-places-autocomplete-via-a-backend-proxy)
    - [Draggable-pin map (web)](#draggable-pin-map-web)
    - [Draggable-pin map (mobile, WebView)](#draggable-pin-map-mobile-webview)
    - [Ship-from address autocomplete (seller settings)](#ship-from-address-autocomplete-seller-settings)
    - [Profile address management](#profile-address-management)
  - [New environment variables (shipping, tax, address \& resilience)](#new-environment-variables-shipping-tax-address--resilience)
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
      - [3) Dispute SLA escalation cron](#3-dispute-sla-escalation-cron)
      - [4) Liability reconciliation cron](#4-liability-reconciliation-cron)
    - [Carrier shipping webhook](#carrier-shipping-webhook)
    - [Key rotation \& webhook replay procedures](#key-rotation--webhook-replay-procedures)
      - [Rotate Stripe webhook secret safely](#rotate-stripe-webhook-secret-safely)
      - [Rotate Chimoney API key safely](#rotate-chimoney-api-key-safely)
      - [Rotate Chimoney webhook secret safely](#rotate-chimoney-webhook-secret-safely)
      - [Replay webhook (admin/on-call detailed)](#replay-webhook-adminon-call-detailed)
      - [Troubleshooting: stuck pending refunds](#troubleshooting-stuck-pending-refunds)
      - [COD refund record API example (sanitized fields)](#cod-refund-record-api-example-sanitized-fields)
    - [Stock reservation \& restoration (variant-aware)](#stock-reservation--restoration-variant-aware)
    - [Failure \& expired session handling](#failure--expired-session-handling)
    - [Seller settlements, ledger lifecycle, and payout execution](#seller-settlements-ledger-lifecycle-and-payout-execution)
      - [System relationship diagram (simplified)](#system-relationship-diagram-simplified)
      - [Settlement and payout flow](#settlement-and-payout-flow)
      - [Ledger lifecycle narrative (delivery -\> payout -\> reserve release)](#ledger-lifecycle-narrative-delivery---payout---reserve-release)
      - [Seller ledger exports (CSV + PDF) and filter](#seller-ledger-exports-csv--pdf-and-filter)
        - [Export data shape](#export-data-shape)
        - [Amount formatting fix](#amount-formatting-fix)
      - [Double-entry ledger and accounting](#double-entry-ledger-and-accounting)
    - [Admin marketplace financials API (`/api/admin/platform-financials/*`)](#admin-marketplace-financials-api-apiadminplatform-financials)
      - [`GET /api/admin/platform-financials/summary`](#get-apiadminplatform-financialssummary)
      - [`GET /api/admin/platform-financials/ledger`](#get-apiadminplatform-financialsledger)
      - [`GET /api/admin/platform-financials/liabilities`](#get-apiadminplatform-financialsliabilities)
      - [`GET /api/admin/platform-financials/tax-remittances`](#get-apiadminplatform-financialstax-remittances)
      - [`POST /api/admin/platform-financials/tax-remittances`](#post-apiadminplatform-financialstax-remittances)
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
        - [Distributed Redis locks (non-scheduler cron jobs)](#distributed-redis-locks-non-scheduler-cron-jobs)
        - [Job-health monitoring](#job-health-monitoring)
      - [2) Environment variables reference](#2-environment-variables-reference)
      - [3) Recommended value profiles](#3-recommended-value-profiles)
      - [4) Runbook / troubleshooting](#4-runbook--troubleshooting)
      - [5) Validation checklist (post-deploy)](#5-validation-checklist-post-deploy)
    - [Stripe Connect prerequisites (seller payouts)](#stripe-connect-prerequisites-seller-payouts)
  - [Products, variants \& model behavior](#products-variants--model-behavior)
    - [Size-chart visibility and category slug naming (Product Page)](#size-chart-visibility-and-category-slug-naming-product-page)
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
    - [Order status transition enforcement](#order-status-transition-enforcement)
    - [Shipment model](#shipment-model)
    - [Carrier service abstraction](#carrier-service-abstraction)
    - [Seller shipment management](#seller-shipment-management)
    - [Customer order tracking timeline](#customer-order-tracking-timeline)
    - [Carrier webhook endpoint](#carrier-webhook-endpoint)
    - [Admin shipment visibility](#admin-shipment-visibility)
  - [Product Reviews \& Ratings](#product-reviews--ratings)
    - [Review model](#review-model)
    - [Profanity filter](#profanity-filter)
    - [Customer review endpoints](#customer-review-endpoints)
    - [Seller review endpoint](#seller-review-endpoint)
    - [Admin review moderation](#admin-review-moderation)
    - [Email notification on new review](#email-notification-on-new-review)
    - [Web \& mobile review UI](#web--mobile-review-ui)
  - [Store Reviews](#store-reviews)
    - [Store review routes](#store-review-routes)
  - [Dispute Resolution](#dispute-resolution)
    - [Dispute model](#dispute-model)
    - [Customer dispute flow](#customer-dispute-flow)
    - [Seller dispute endpoints](#seller-dispute-endpoints)
    - [Admin dispute management](#admin-dispute-management)
    - [SLA enforcement](#sla-enforcement)
    - [Stripe chargeback integration](#stripe-chargeback-integration)
    - [Evidence upload](#evidence-upload)
    - [Email notifications (disputes)](#email-notifications-disputes)
    - [i18n keys (disputes)](#i18n-keys-disputes)
  - [Transactional email system](#transactional-email-system)
  - [In-app notification centre](#in-app-notification-centre)
    - [Notification model](#notification-model)
    - [REST API](#rest-api)
    - [Web bell UI](#web-bell-ui)
    - [Mobile bell UI](#mobile-bell-ui)
    - [`order_shipped` notification](#order_shipped-notification)
    - [Wiring new notification types](#wiring-new-notification-types)
  - [Admin user management](#admin-user-management)
  - [Admin audit-log dashboard](#admin-audit-log-dashboard)
  - [GDPR \& user data export](#gdpr--user-data-export)
  - [Security \& observability](#security--observability)
    - [Field-level PII encryption](#field-level-pii-encryption)
    - [Role \& Permission Matrix](#role--permission-matrix)
  - [Technical SEO](#technical-seo)
    - [Crawl control (robots.txt)](#crawl-control-robotstxt)
    - [Dynamic sitemap](#dynamic-sitemap)
    - [Structured data (JSON-LD)](#structured-data-json-ld)
    - [Canonical URLs \& hreflang](#canonical-urls--hreflang)
    - [Page indexing coverage](#page-indexing-coverage)
    - [SEO roadmap (Stage 4)](#seo-roadmap-stage-4)
  - [Testing \& CI](#testing--ci)
    - [Coverage](#coverage)
    - [Critical workflows covered](#critical-workflows-covered)
    - [How to run tests](#how-to-run-tests)
    - [How to test the COD refund purge cron job](#how-to-test-the-cod-refund-purge-cron-job)
    - [Test status](#test-status)
    - [CI/CD pipeline](#cicd-pipeline)
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
    - [Seller Onboarding Flow](#seller-onboarding-flow)
    - [Seller Dashboard](#seller-dashboard)
    - [Admin Dashboard](#admin-dashboard)
    - [Mobile](#mobile)
  - [Commercial license (proprietary) \& selling notes](#commercial-license-proprietary--selling-notes)
  - [Demo credentials (web \& mobile)](#demo-credentials-web--mobile)
  - [Additional documentation](#additional-documentation)
  - [Contact / commercial enquiries](#contact--commercial-enquiries)

---

## Maintainer context

This project is designed, built, and maintained end-to-end by a single freelance full-stack
developer — from system architecture and database schema through to CI/CD pipelines, mobile
releases, and production security hardening.

Documentation in [`docs/`](docs/) is not scaffolding — it covers actively used operational runbooks,
security rotation procedures, settlement and payout operations, and CI/CD setup that are referenced
during day-to-day development and deployment. The codebase is production-grade in both scope and
discipline: distributed Redis locks, field-level PII encryption, HMAC-signed OAuth bridge codes,
multi-stage Docker builds, and a full 235-suite / 3,333-test backend test suite with CI enforcement
on every PR.

---

## One-line pitch

Enterprise-grade e-commerce web storefront and Expo mobile app sharing a single Node/Express API,
complete with robust session security, per-variant inventory, reservation/restore logic, Stripe
payments, order fulfilment tooling, and production observability.

---

## Live demo & hosted domains

- Frontend (production): `https://vexflare.com` (Vercel)
- Frontend (alternate / staging): `https://ecommerce-mern-website-seven.vercel.app`
- API (production): `https://api.vexflare.com` (Railway)
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

**High-level system architecture**

```mermaid
graph TB
    Browser["React SPA"]
    Mobile["React Native (Expo)"]
    API["Express API"]
    MongoDB[("MongoDB Atlas")]
    Redis[("Redis")]
    Stripe["Stripe"]
    Shippo["Shippo"]
    Cloudinary["Cloudinary"]

    Browser --> API
    Mobile --> API
    API --> MongoDB
    API --> Redis
    API --> Stripe
    API --> Shippo
    API --> Cloudinary
```

**Request flow through the middleware pipeline**

```mermaid
graph TD
    A["Browser Request"] --> B["Helmet + CSP"]
    B --> C["CORS"]
    C --> D["Rate Limiting"]
    D --> E["CSRF Token"]
    E --> F["Authentication"]
    F --> G["Route Controller"]
    G --> H["Service Layer"]
    H --> I[("MongoDB / Redis")]
```

**Background jobs started at boot (server.js)**

```mermaid
graph TB
    S["server.js startup"]
    S --> J1["Settlement Scheduler"]
    S --> J2["Subscription Renewal"]
    S --> J3["Reserve Release"]
    S --> J4["Settlement Reconciliation"]
    S --> J5["COD Refund Purge"]
    S --> J6["Payout Retry"]
    S --> J7["COD Payout Reconciliation"]
    S --> J8["Seller Closure Finalizer"]
    S --> J9["Dispute SLA Escalation"]
    S --> J10["Liability Reconciliation"]
    S --> J11["Job Health Monitor"]
    J1 --> JH[("JobHealth")]
    J2 --> JH
    J3 --> JH
    J4 --> JH
    J5 --> JH
    J6 --> JH
    J7 --> JH
    J8 --> JH
    J9 --> JH
    J10 --> JH
    J11 --> JH
```

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
- The platform admin seller/store is provisioned automatically at server startup via
  `backend/src/lib/ensureAdminStore.js`. No manual migration step is required — the function runs
  idempotently on every boot (existing records are updated safely; missing records are created).
- Public store lookup (slug-based) is handled by `publicStore.controller.js` at
  `GET /api/public/stores/{slug}` and surfaces only `StorePublic` fields when the multi-seller flag
  is enabled. The authenticated store route (`store.route.js`) exposes `GET /:id` and requires a
  MongoDB ObjectId, not a slug.
- Multi-store tenant flags:
  - `FEATURE_MULTI_SELLER=true`
  - `FEATURE_SELLER_KYC=true`
  - `FEATURE_SELLER_ORDERS=true`
  - `FEATURE_SELLER_CLOSURE_V1=true` — enables the seller-initiated store closure flow (eligibility
    check, closure request, pending-closure UI, and the finalizer cron job). When unset or `false`,
    all `/api/seller/close-store/*` and `/api/admin/sellers/:id/close-store` routes are inaccessible
    and the dashboard panel is hidden.
  - `FEATURE_DISPUTES_V1=true` — enables the full dispute-resolution surface: all `/api/disputes/*`,
    `/api/seller/disputes/*`, and `/api/admin/disputes/*` routes, the SLA escalation cron, and all
    dispute-related panels on web and mobile. When unset or `false`, dispute routes are inaccessible
    and all dispute UI is hidden.
  - `FEATURE_IN_APP_NOTIFICATIONS=true` — enables the in-app notification centre: REST endpoints at
    `/api/notifications/*`, the bell icon with unread badge in the Navbar, the dropdown panel and
    full-list page on web (60-second polling), and the bottom sheet with AppState-aware polling on
    mobile. When unset or `false`, all notification routes return 404 and the bell UI is hidden.
  - `FEATURE_ADMIN_AUDIT_DASHBOARD=true` — enables the admin audit-log tab surfacing five audit
    types: seller lifecycle, seller profile access, COD refund access, user deletion, and document
    access. When unset or `false`, audit endpoints and the dashboard tab are inaccessible.
- **Silent-off behaviour:** when `FEATURE_MULTI_SELLER` is unset (or any value other than `"true"`),
  the entire seller marketplace surface is silently disabled — `/api/seller/*`, `/api/sellers/*`,
  `/api/stores/*`, and `/api/public/stores/*` routes all return 404 or are never registered. A fresh
  production deploy without this flag set will have no visible seller or store functionality.
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

#### Onboarding approval and product-readiness flowchart

```mermaid
flowchart TD
  Start["User wants to become seller"] --> PreApply["Open /seller/pre-apply"]
  PreApply --> Apply["Submit /api/sellers/apply (draft or pending)"]
  Apply --> Docs["Upload KYC docs /api/sellers/:id/docs"]
  Docs --> KycReview["Admin KYC review /api/sellers/admin/:id/verify"]

  KycReview -->|rejected or action_required| Rework["Seller updates application and resubmits"]
  Rework --> Apply

  KycReview -->|verified| Activate["Seller status=active + kyc.status=verified"]
  Activate --> StoreProvision["Auto-provision Store profile"]

  StoreProvision --> Consents["Seller accepts terms/privacy in dashboard"]
  Consents --> StoreEdit["Seller updates store profile (logo/banner/policies/description)"]
  StoreEdit --> StoreReview["Admin store policy/profile review"]

  StoreReview -->|rejected or action_required + notes| StoreFix["Seller applies requested changes"]
  StoreFix --> StoreEdit

  StoreReview -->|approved| Ready["Seller can add products and submit for moderation"]

  StoreProvision --> Payout["Complete Stripe Connect onboarding"]
  Payout --> PayoutEligible["Seller becomes payout-eligible for order settlements"]
```

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
  - `status` (`active` / `inactive` / `suspended`)
  - `kyc.status` (`draft`, `pending`, `verified`, `rejected`, `action_required`)
  - `closure` — nested object tracking the store closure lifecycle; includes `status` (`active` /
    `pending_closure` / `closed_archived`), `initiatedAt`, `scheduledFor`, `actor` (`self` /
    `admin`), `actorUserId`, `reason`, `closedAt`, `adminOverride`, and `notificationEmail`.
    Compound index on `{ 'closure.status': 1, 'closure.scheduledFor': 1 }`.
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
   - The endpoint validates the requested transition against the KYC state machine before writing;
     invalid transitions return `422`.
   - Rejections store both `kyc.rejectionReason` and a frozen `kyc.rejectedApplication` snapshot for
     auditability.
   - For `action_required`, sellers can re-submit without losing their previous submission context.
   - Full approval (status change, document marks, store creation, role grant) executes inside a
     single MongoDB transaction; any failure triggers automatic rollback.
5. **Store provisioning** (auto after approval)
   - On approval, the backend ensures a Store exists for the seller, using defaults derived from the
     seller profile (store name, slug, branding defaults, and basic policies).
   - Sellers can then configure their store via the seller dashboard settings page.

#### State machine (clear, enforced transitions)

Transitions are enforced by `backend/src/services/seller/kycStateMachine.js`. The allowed transition
matrix is:

```
draft            → pending
pending          → verified | rejected | action_required
action_required  → pending
rejected         → pending
```

The seller `apply` endpoint blocks all mutations when `kyc.status === 'verified'`. Profile changes
for verified sellers must go through an admin-mediated flow.

| Seller `status` | KYC `status`      | Meaning / Access                                                         |
| --------------- | ----------------- | ------------------------------------------------------------------------ |
| `inactive`      | `draft`           | Application saved but not submitted. Seller dashboard access is blocked. |
| `inactive`      | `pending`         | Submitted for review; admin action required.                             |
| `inactive`      | `action_required` | Seller must re-submit updates; previous data is preserved.               |
| `inactive`      | `rejected`        | Rejected with reason; seller must re-apply.                              |
| `active`        | `verified`        | Approved; full seller access unlocked and store provisioning enabled.    |

#### Security & ownership controls

- **Ownership enforcement:** All seller endpoints use `protectRoute` + role restriction plus
  `requireSeller`, ensuring only the seller who owns a seller profile can access seller APIs.
- **KYC gate (`requireVerifiedSeller`):** Product write, order management, and store-settings edit
  endpoints are additionally gated by `requireVerifiedSeller` (defined in
  `backend/src/middleware/seller.middleware.js`), which requires `kyc.status === 'verified'` and
  `status === 'active'`. Non-verified sellers retain access to read-only, bank-account, Stripe
  Connect onboarding, and KYC document upload routes.
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
- Rejections preserve a snapshot of the submitted application to support reviews or audits.

#### Seller withdrawal

Sellers in `draft`, `pending`, `action_required`, or `rejected` KYC states may withdraw their
application via the `/withdraw` endpoint. Verified sellers (`kyc.status === 'verified'`) receive
`422` and must use the dedicated store closure flow instead (see
[Seller store closure lifecycle](#seller-store-closure-lifecycle)).

Withdrawal does **not** hard-delete documents. The controller sets `lifecycleStatus='archived'`,
`archiveReason='withdrawn'`, and `supersededAt=now` on seller documents. All document queries filter
on `lifecycleStatus: 'active'` by default so archived records are excluded from normal reads.

A purge script (`backend/src/scripts/purge-archived-seller-documents.js`) permanently removes
archived documents older than the configured retention window. Set `KYC_DOCUMENT_RETENTION_DAYS` to
control the window (default `2555`, equivalent to 7 years).

#### Audit trail collections

**`SellerLifecycleAudit`** (`backend/src/models/sellerLifecycleAudit.model.js`) — append-only record
of every admin decision in the seller lifecycle. Covered actions include: KYC
verify/reject/action_required, bank-update approve/reject, store review, payout-hold toggle,
reserve-tier change, store-status change, and store closure lifecycle events (`closure_initiated`,
`closure_cancelled`, `closure_finalized`). Each record captures actor, action, `fromStatus`,
`toStatus`, notes, a data snapshot, and the actor's IP address.

**`SellerProfileAccessAudit`** (`backend/src/models/sellerProfileAccessAudit.model.js`) —
append-only record of admin access to tier-1 seller PII (govIdNumber, birthDate, addresses, banking
last-four). Entries are created when admin views the `/admin/requests` or `/admin/approved` list
endpoints that expose this data.

#### Seller bank-update review lifecycle (pending/approved/rejected + seller cancel)

Bank updates for already-verified sellers now run on a dedicated review lifecycle that is separate
from seller-application KYC statuses. The current lifecycle is:

`pending` → (`approved` | `rejected` | `action_required`) with a seller-side cancel action available
from `pending`, `rejected`, and `action_required`.

| Bank update review status | Who sets it           | Meaning                                                                           |
| ------------------------- | --------------------- | --------------------------------------------------------------------------------- |
| `pending`                 | Seller submits update | Admin review is required; storefront/product creation remains paused.             |
| `approved`                | Admin approve action  | New bank details are promoted to approved payout bank fields.                     |
| `rejected`                | Admin reject action   | Submitted bank update is denied; admin notes explain why.                         |
| `action_required`         | Admin action          | Admin requests more information without outright rejecting; seller must resubmit. |
| `cancelled` (UI outcome)  | Seller cancel action  | Seller reverts to last approved bank profile and clears active review.            |

State transition summary:

1. Seller submits a bank update (`pending`).
2. Admin approves (`approved`), rejects (`rejected`), or requests changes (`action_required`).
3. Seller can cancel while review is `pending`, `rejected`, or `action_required`; cancel restores
   the last approved bank profile and clears the in-flight review metadata.

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

#### Bank-update review lifecycle flowchart

```mermaid
flowchart TD
  Submit["Seller submits bank update"] --> Pending["updateReview.status=pending"]
  Pending --> Admin{"Admin decision"}
  Admin -->|Approve| Approved["status=approved + payout.bank updated"]
  Admin -->|Reject| Rejected["status=rejected + notes + archived snapshot"]
  Pending -->|Seller cancel| Cancelled["Restore last approved bank + clear active review"]
  Rejected -->|Seller cancel| Cancelled
```

#### Backend contract for bank-update records

The backend model contract is intentionally strict so UI and admin tooling can rely on stable
semantics:

- Seller KYC bank documents use a single canonical type: `bank_statement`. New records must not use
  the legacy `bank` type.
- Bank statement documents now include explicit phase metadata: `isOnboarding=true` for onboarding
  uploads before seller approval and `isOnboarding=false` for post-approval bank-update uploads.
- `bank_statement` `version` is incremented in both onboarding and bank-update upload paths, so the
  latest statement is deterministic without relying on legacy type aliases.
- `seller.updateReview` is the active review envelope for bank updates. The envelope uses only
  `requestType` (the legacy `type` field has been dropped). Run
  `backend/src/scripts/normalize-bank-update-review.js` to migrate any existing records that still
  carry the old `type` field.
- `updateReview.status` for bank updates is constrained to `pending`, `approved`, `rejected`, or
  `action_required`.
- **Stripe bank account retention:** when a seller submits a bank update, the old Stripe external
  account is **not** deleted at submit time. The pending account is created on Stripe but the old
  account remains the default until an admin decision:
  - **Admin approve** → old external account is deleted from Stripe; new account becomes the
    default.
  - **Admin reject / seller cancel** → the pending (new) external account is deleted from Stripe and
    the old account is re-set as the default.
- On admin reject, the submitted payload is copied into an archived record (`ArchivedBankUpdate`)
  and the mutable in-review payload is cleared from the active review envelope.
- `seller.payout.bank` is the approved source of truth for live settlement data. UIs should treat
  this as canonical approved bank information.
- Seller cancel restores `application.banking` from `payout.bank` and clears `updateReview`, so the
  dashboard returns to the last approved bank baseline.

#### Bank-update API routes (seller + admin)

Use the following API routes for the full bank-update lifecycle:

- **Submit bank update (seller):** `POST /api/seller/payment-method`
  - Submit one-time bank token + statement metadata to start/replace a bank-update review.
  - Backend creates the new external account on Stripe (`createExternalAccount`) but keeps the
    existing Stripe bank account as the default until an admin decision is made.
  - Creates/updates `updateReview` with `status=pending` for verified active sellers.
- **Admin approve/deny review:** `PATCH /api/sellers/admin/:id/update-review?action=approve|reject`
  - `approve`: promotes previously persisted non-sensitive bank metadata to `payout.bank` and marks
    review `approved`.
  - Approval does **not** reuse or persist bank token identifiers; token use is submit-time only.
  - `reject`: marks review `rejected`, requires notes, archives denied update snapshot.
- **Seller cancel update review:**
  - `POST /api/seller/payment-method/cancel-bank-update`
  - `POST /api/seller/bank-update/cancel` (alias)
  - Allowed when current bank-update review is `pending`, `rejected`, or `action_required`; restores
    last approved bank fields and clears active review state.
  - The primary cancel endpoint is intentionally excluded from the global API rate limiter to avoid
    blocking seller recovery actions with `429` during repeated cancel/retry attempts.

#### KYC document handling

**Document upload:** Data URI payloads are rejected at the upload endpoint with `400 Bad Request`.
Documents must be sent as multipart file uploads using the `kycUpload` middleware
(`middleware/kycUpload.middleware.js`). The middleware enforces per-file size limits, validates MIME
types, and attaches the processed buffer to `req.file` before the controller runs.

**Server-side document viewing:** The preferred access pattern for KYC documents is server-side
proxying. The backend fetches the document from Cloudinary and pipes it to the client without
exposing the Cloudinary URL to the browser:

- `GET /api/seller/kyc/documents/:documentId/view` — seller-authorized server-side document view.
- `GET /api/sellers/admin/docs/:documentId/view` — admin server-side document view.

The KYC document access layer enforces a **view-first, policy-driven** model:

- **Inline viewer is the default** for all roles. Document access uses short-lived view tokens and
  renders with `Content-Disposition: inline`.
- **Tiered controls are enforced per document**:
  - **Tier 1** (`identity`, `proof_of_address`, `authorization`) is **view-only**. Note:
    `proof_of_address` is classified as tier 1 (previously tier 2).
  - **Tier 2** documents can be viewed and are eligible for tightly controlled admin download.
- `isOnboarding` defaults on document records are now derived from the `ONBOARDING_DOCUMENT_TYPES`
  set rather than being hard-coded per type.
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

**PII decryption helper:** All seller-facing and admin endpoints that return application data call
`decryptApplicationPii(sellerDoc)` before sending the response. This helper decrypts every
AES-256-GCM field-encrypted value on the Seller record (see
[Field-level PII encryption](#field-level-pii-encryption)) and returns a plain object safe for
serialization. Never expose the raw Mongoose document containing encrypted buffer values to the
client.

**Application resume flow:** The seller onboarding form persists a draft to the backend on every
step transition. When a seller returns to a partially completed application, the backend detects the
first incomplete step (the first step whose required fields are not yet filled) and resumes from
there rather than always returning to step 1. This logic is enforced server-side; the frontend reads
the `resumeStep` field from the API response and navigates accordingly. On logout, client-side
onboarding drafts (`sellerPreOnboarding`, `sellerProofOfAddressDraft`) are cleared from local
storage to prevent stale state from leaking to the next user on shared devices.

**Seller onboarding state machine**

```mermaid
stateDiagram-v2
    [*] --> draft
    draft --> pending : submit application
    pending --> verified : admin approves
    pending --> rejected : admin rejects
    pending --> action_required : docs requested
    action_required --> pending : seller resubmits
    verified --> [*]
    rejected --> [*]
```

### Seller store settings, storefront policies, and admin review flow

This section documents how **seller settings and store policies** are edited, when changes require
admin review, and how the public storefront stays consistent while reviews are pending. The flow is
split into seller UI behavior, backend review gating, and storefront policy rendering.

#### Seller settings: editable fields & review gates

Seller settings live in the seller dashboard’s **Store profile** page and are backed by the store
profile API. The page (`SellerStoreSettings.jsx`) is organized into **five collapsible sections**,
built on a shared `CollapsibleSection` component:

- **Store profile** — name, description, logo/banner, support contact, SEO tagline.
- **Store categories** — up to 5 profile categories with optional descriptions.
- **Store policies** — shipping and return/refund policy text.
- **Shipping** — ship-from address, default rate, regional tiers, carrier configuration, and
  excluded regions (see [Seller shipping UI](#seller-shipping-ui) below).
- **Store settings** — pause/resume the store and the share-link/QR code.

Each section is collapsed by default and shows a one-line summary (e.g. the store name, the number
of selected categories, whether policies are complete, the region-tier count, or whether the store
is paused). Expanding a section reveals its fields and its own **Save** button — saving persists
only that section's payload (`PUT /api/stores/:id/policies` merges partial updates onto the existing
document), so editing one section never clears another. The **Store settings** section has no Save
button: its pause/resume action and share-link/QR download apply immediately. The editable fields
are grouped into **reviewed** vs **instant** updates:

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

**Logo/banner image storage (Cloudinary URLs, never inline base64):**

Store logo and banner images are uploaded to Cloudinary and the Store document stores only the
resulting **URL** — never a base64 data URI. The dashboard uploads the chosen file via
`POST /api/stores/:id/image` (multipart; `kind=logo|banner`), which reuses the product-image
pipeline (`uploadMiddleware` → `cloudinary.uploader.upload`, capped at 512px logos / 1600px banners)
and returns `{ url }`; the seller then saves that URL through the normal store-update call.
`updateStorePolicies` **rejects** a `data:` URI for `logoUrl`/`bannerUrl` with a 400.

Why: inlining base64 made a single Store document multiple megabytes (one banner was ~4.7 MB), so
every read that needs branding — the seller dashboard overview and the public storefront — had to
transfer the whole document over Atlas, taking tens of seconds. Storing URLs keeps the document a
few KB. Convert any pre-existing base64 images with the one-time migration:
`npm run migrate:store-images` (dry-run) / `npm run migrate:store-images:apply`.

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

#### Policy update + immutable order snapshot flowchart

```mermaid
flowchart TD
  Edit["Seller edits store policy/branding"] --> Save["Save settings"]
  Save --> Changed{"Reviewed fields changed?"}
  Changed -->|No| Instant["Instant update (non-reviewed fields)"]
  Changed -->|Yes| Pending["Set reviewStatus=pending + lock reviewed fields"]
  Pending --> Snapshot["Keep last approved snapshot for storefront"]
  Snapshot --> Checkout["New checkout reads approved source only"]
  Checkout --> OrderSnap["Persist immutable order policy text on Order"]
  Pending --> AdminDecision{"Admin decision"}
  AdminDecision -->|Approve| Live["New policy/branding goes live"]
  AdminDecision -->|Reject / Action required| OldLive["Storefront keeps last approved snapshot"]
  Live --> FutureOrders["Future orders use newly approved policy snapshot"]
```

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
   - **Action required** → `reviewStatus=action_required` with required notes. The admin may also
     populate `requiresChangesFor` — a structured list of field names that need correction. The
     seller dashboard can use this list to highlight which specific fields must be updated before
     resubmission.
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

- During checkout, `fetchStorePolicySnapshot` resolves the **approved** policy source. It prefers
  `reviewSnapshot` when `reviewStatus !== 'approved'`, matching the storefront's display behavior.
  This ensures a pending policy edit cannot leak into the order snapshot at checkout.
- For performance it `.select()`s **only** the policy fields (not branding/logo/banner), so the
  checkout query stays a few hundred bytes regardless of Store-document size. The
  store/fee/commission lookups run in parallel and outside the order transaction in both the COD and
  Stripe flows.
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
or updated.

**Admin-initiated pause:** When an admin pauses a store, the backend automatically sets
`seller.payoutHold=true` with `payoutHoldReason=’store_paused_admin’`. When the admin reactivates
the store, the payout hold is released. Seller self-pause does **not** affect `payoutHold`.

**In-flight checkout contract:** Pausing a store hides products from new carts and prevents new
checkout sessions from including that store’s items. It does **not** cancel in-flight checkout
sessions that were already initiated before the pause.

### Seller store closure lifecycle

A verified seller can permanently close their store while keeping their user account active for
shopping. Store closure is distinct from user account deletion: deletion removes the user and all
linked seller data in one transaction; store closure closes only the seller/store while the user
account continues to function. See also [GDPR & user data export](#gdpr--user-data-export).

The feature is gated by `FEATURE_SELLER_CLOSURE_V1`.

#### Phase 1 — Eligibility check & closure request

- `GET /api/seller/close-store/eligibility` — evaluates whether the seller can initiate closure.
  Checks: active product listings, recent orders within the waiting period, pending payouts, open
  refund requests, and open disputes. Returns `{ eligible, reasons, earliestClosureAt }`.
- `POST /api/seller/close-store` — validates the store slug confirmation and acknowledgement, then
  calls the closure service. Requires `FEATURE_SELLER_CLOSURE_V1` to be enabled.
- `GET /api/seller/close-store/status` — returns the current closure state for the seller's store.
- **Seller dashboard UI:** "Close Store" panel in the Account Management section, containing an
  eligibility checklist, a typed-slug confirmation input, and an optional reason field. A
  pending-closure banner is displayed at the top of the seller dashboard while `closure.status` is
  `pending_closure`.
- **Waiting period:** `SELLER_CLOSURE_WAITING_DAYS` (default `90`) controls how many days of recent
  order history must be clear before closure is permitted.

#### Phase 2 — Immediate deactivation (soft close)

On `POST /api/seller/close-store`, the `initiateStoreClosure` service executes a single MongoDB
transaction that:

1. Sets `closure.status = 'pending_closure'` and `seller.status = 'suspended'`.
2. Hides all storefronts and products (`pausedByStore: true`).
3. Enables a payout hold (`payoutHoldReason: 'store_closure_pending'`).
4. Triggers a final settlement reconciliation pass.
5. Writes a `SellerLifecycleAudit` row with action `closure_initiated`.

**`blockIfPendingClosure` middleware** (`backend/src/middleware/seller.middleware.js`) returns
`423 Locked` on all mutating seller routes (products, orders, subscriptions, store settings, payment
methods) while closure is pending. Read-only `GET` requests remain allowed.

**Cancel closure:** `POST /api/seller/close-store/cancel` reverses all Phase 2 effects. Self-serve
cancel is allowed only outside a 24-hour pre-deletion lockout window; admin can cancel at any time
via `POST /api/admin/sellers/:id/closure/cancel`.

#### Phase 3 — Waiting period & permanent deletion (finalizer job)

After `closure.scheduledFor` has passed, the daily `sellerClosureFinalizer` cron job
(`backend/src/jobs/sellerClosureFinalizer.job.js`) processes each qualifying seller:

1. **Safety re-check** — re-verifies no new orders or disputes arrived during the waiting period.
2. **PII anonymization** — scrubs seller application fields, Store name, logo, and banner.
3. **Product handling** — hard-deletes products that appear in no orders; tombstones products that
   appear in any Order so order history remains resolvable against the anonymized Store record.
4. **Referential integrity** — Orders retain their `sellerId` / `storeId` references and resolve to
   the anonymized Store; financial and order records are preserved as required by law.
5. **Audit** — writes a `SellerLifecycleAudit` row with action `closure_finalized`.
6. **Email** — sends `sendStoreClosureCompletedEmail` to the pre-anonymization `notificationEmail`.

**Cron schedule:** runs at `02:00 UTC` by default. Override with `SELLER_CLOSURE_FINALIZER_CRON`.
Disable with `SELLER_CLOSURE_FINALIZER_ENABLED=false`. Job health is tracked via
`jobHealth.model.js`.

**Env vars:**

```
SELLER_CLOSURE_WAITING_DAYS          # waiting period before Phase 3 (default: 90 days)
SELLER_CLOSURE_FINALIZER_CRON        # cron expression for the finalizer job (default: "0 2 * * *")
SELLER_CLOSURE_FINALIZER_ENABLED     # set to "false" to disable automatic finalizer (default: true)
```

#### Admin override

- `POST /api/admin/sellers/:id/close-store` — force-close a seller store. Accepts an optional
  `skipEligibility: true` flag to bypass all eligibility checks. Sends the same
  `sendStoreClosureInitiatedEmail` as self-serve closure.
- `POST /api/admin/sellers/:id/closure/cancel` — cancel a pending closure without the 24-hour
  self-serve lockout restriction.
- `GET /api/admin/sellers/:id/closure` — retrieve full closure state for admin review.
- **Admin UI:** "Close store" button in the admin Approved Sellers list, with a reason text input
  and a force-close toggle that sets `skipEligibility`.

#### Closure email notifications

Three transactional emails are sent during the closure lifecycle (all implemented in
`backend/src/lib/email.js`):

| Function                         | Trigger                           | Recipient                             |
| -------------------------------- | --------------------------------- | ------------------------------------- |
| `sendStoreClosureInitiatedEmail` | Closure initiated (self or admin) | Seller email                          |
| `sendStoreClosureCancelledEmail` | Closure cancelled (self or admin) | Seller email                          |
| `sendStoreClosureCompletedEmail` | Phase 3 finalization complete     | Pre-anonymization `notificationEmail` |

#### i18n

24 new translation keys under the `seller.closure.*` namespace were added across all 4 locales
(`en`, `es`, `fr`, `ar`) in `shared/locales/`. Key groups include eligibility messages, dashboard
panel labels, confirmation dialog text, and pending-closure banner copy.

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

#### Product creation + moderation pipeline flowchart

```mermaid
flowchart TD
  Create["Seller creates product draft"] --> Validate["Backend validates ownership + payload"]
  Validate --> Media["Images processed (Sharp) + uploaded (Cloudinary)"]
  Media --> Draft["approvalStatus=draft, visibility=hidden"]
  Draft --> Submit["Seller submits for review"]
  Submit --> Pending["approvalStatus=pending"]
  Pending --> AdminQueue["Admin moderation queue"]
  AdminQueue --> Decision{"Approve?"}
  Decision -->|Yes| Approved["approvalStatus=approved"]
  Approved --> Public["visibility=public (storefront eligible)"]
  Decision -->|No| Rejected["approvalStatus=rejected + notes"]
  Rejected --> Revise["Seller edits product"]
  Revise --> Submit
```

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
   - **Approve** → verifies that `seller.kyc.status === 'verified'` and `seller.status === 'active'`
     before writing; returns `422` if either condition is not met. On success, sets
     `approvalStatus=approved`, `visibility=public` (or scheduled visibility).
   - **Reject** → sets `approvalStatus=rejected`, `visibility=hidden`, and writes a rejection
     reason.
4. **Notification**
   - Seller receives a status update in dashboard (and optionally email).

This workflow ensures every seller product has a clear admin decision before being shown in public
lists or categories, and that approval is blocked if the seller's own KYC has lapsed or been
revoked.

#### State transitions & audit trail

The flow is intentionally explicit to avoid ambiguous states:

| Current status | Action                                 | Next status | Notes                                                                                                                                                                                      |
| -------------- | -------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `draft`        | Seller saves without submitting        | `draft`     | Editable by seller; never visible publicly.                                                                                                                                                |
| `draft`        | Seller submits for review              | `pending`   | Locks fields that require moderation.                                                                                                                                                      |
| `pending`      | Admin approves                         | `approved`  | Eligible for storefront when visibility rules also allow it.                                                                                                                               |
| `pending`      | Admin rejects                          | `rejected`  | Seller receives reason and must edit before re-submit.                                                                                                                                     |
| `rejected`     | Seller edits and re-submits            | `pending`   | Starts a new review cycle; prior rejection context is retained. A `pendingReviewSnapshot` is always captured on submission so moderators can diff against the previously rejected version. |
| `approved`     | Seller edits moderation-sensitive data | `pending`   | Re-review gate for core listing changes.                                                                                                                                                   |
| `approved`     | Admin unpublishes/archives             | `hidden`    | Listing remains in system for audit/reporting but is removed storefront.                                                                                                                   |
| `hidden`       | Admin republishes (if policy permits)  | `approved`  | Returns to approved lifecycle without creating a new product record.                                                                                                                       |

Audit trail expectations:

- Every moderation decision stores actor (`adminId`/seller), timestamp, and reason metadata.
- Rejections preserve reviewer notes for seller feedback and operational follow-up.
- Status history is retained to support disputes, compliance review, and rollback investigations.

**Stale category guard on cancel:** when a seller cancels a pending product edit, the backend
validates that all category references in the edit snapshot still exist. If any referenced category
has since been deleted, the product is forced back to `draft` rather than restoring to the previous
`pending` state. The seller must update the category selection before resubmitting.

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

**Versioning behavior:** creating a new FeeConfig via the admin API automatically closes the
previously active record by setting its `effectiveEndDate` to one second before the new
`effectiveStartDate`. This ensures there is never more than one active config at any point in time
and preserves a complete audit trail of historical rate changes.

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
6. Persist four **fee snapshot fields** on each order item:
   - `appliedFeeConfigId` — ObjectId of the FeeConfig record active at checkout time.
   - `appliedCommissionPct` — commission percentage actually applied.
   - `appliedMinFeeCents` — minimum fee in cents actually applied.
   - `appliedPlatformFeeCents` — platform fee in cents charged for this item.

These snapshot fields are populated by `buildOrderItemsWithCommission` at order creation and are
never recalculated retroactively. The admin Order Details modal displays them under the "Fee
Snapshot" panel (visible to admin scope only). A backfill script for existing orders is available at
`backend/src/scripts/backfill-order-fee-snapshot.js`.

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
- **Failure path (card error):** when a renewal fails with a card-level error (`card_declined`,
  `expired_card`, `no_payment_method`, etc.) and `PAYMENT_GRACE_ENABLED=true`, the seller enters a
  payment grace period instead of being immediately downgraded (see
  [Payment grace period](#payment-grace-period) below). Retry attempts are paused for the duration
  of the grace period. The seller receives an email notification and the dashboard shows a
  non-dismissible amber banner with the deadline and an **Update payment method** link.
- **Failure path (other errors):** retry attempts are scheduled with a configurable delay. After the
  final retry, the seller is downgraded and emailed.
- **Downgrade:** once the grace period (or retry window, for non-card failures) passes, the seller
  is auto-downgraded to Free, expiry is cleared, and subscription limits are re-applied.

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

#### Payment grace period

When a renewal fails due to a card error and `PAYMENT_GRACE_ENABLED=true` (the default), the seller
is given a configurable window (`PAYMENT_GRACE_PERIOD_DAYS`, default 7 days) to fix their payment
method before being downgraded.

**Data model additions (Seller)**

- `paymentGracePeriodEndsAt` (`Date`) — set to `now + PAYMENT_GRACE_PERIOD_DAYS` when the grace
  period starts; cleared on successful recovery or downgrade.
- `paymentFailureReason` (`String`) — stores the Stripe decline code (e.g. `card_declined`) for
  display and auditing.

**Grace period lifecycle**

1. **Entry:** renewal fails with a card error → `paymentGracePeriodEndsAt` is set, retry scheduling
   is paused, seller receives an email notification.
2. **Seller recovers:** seller updates their payment method in the dashboard → a one-time charge is
   attempted immediately for the pending renewal amount.
   - **Success:** grace period fields are cleared, `planExpiresAt` is extended as normal.
   - **Failure:** grace period continues; seller can retry until the deadline.
3. **Expiry:** a scheduled sweep finds sellers whose `paymentGracePeriodEndsAt` is in the past →
   auto-downgrade to Free plan, grace fields cleared, email notification sent.

**Dashboard banner**

While a seller is in a grace period, the seller dashboard renders a non-dismissible amber banner
containing the grace deadline and an **Update payment method** button. The banner uses i18n keys
under `seller.paymentGraceBanner`.

**Grace period env vars**

```env
PAYMENT_GRACE_ENABLED=true         # default true; set false to disable grace period (immediate downgrade)
PAYMENT_GRACE_PERIOD_DAYS=7        # default 7; number of days before auto-downgrade triggers
```

#### Admin/operations notes

- **Update fee defaults:** In the admin dashboard, open **Commission settings** and create/update a
  FeeConfig record with the Free/Pro commission %, minimum fee, and active date range.
- **Update Pro plan price:** Create a new **Stripe Price** in Stripe, then paste the new Price ID
  into FeeConfig. The next checkout/renewal will automatically use the updated price.
- **Rule overrides:** Use the same admin screen to create seller- or category-specific commission
  rules when you need targeted promotions or negotiated fee rates.
- **Payment status column:** The seller list in the admin dashboard displays a **Payment status**
  column showing `OK`, `Grace period` (with the deadline), or `Failed` for each seller. Admins can
  manually end a grace period (triggering an immediate retry) or force-downgrade a seller from this
  view.

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

#### Store reviews & rating aggregate

```
GET /api/stores/:slug/reviews           — public; published store reviews; paginated; sortable.
GET /api/stores/:slug/rating-aggregate  — public; average rating, total count, and 1–5 star
                                          distribution for the store.
```

The `StorefrontPage.jsx` component renders the rating aggregate badge and review list inline.
Eligibility: the authenticated user must have at least one `delivered` order from the store. One
review per customer per store. See [Store Reviews](#store-reviews) for the full model spec.

---

## Deployment surfaces & release workflow

**Deployment topology**

```mermaid
graph TB
    subgraph vercel["Vercel CDN"]
        V["React SPA"]
    end
    subgraph railway["Railway"]
        API["Express API"]
    end
    subgraph atlas["MongoDB Atlas"]
        DB[("Replica Set")]
    end
    subgraph rediscloud["Redis"]
        RD[("Upstash / ioRedis")]
    end
    V --> API
    API --> DB
    API --> RD
```

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

- Required environment variable: `PUBLIC_CLIENT_FALLBACK_URL=https://vexflare.com/reset-password`.

**Frontend build & code-splitting pipeline**

```mermaid
graph LR
    A["Vite Build"] --> B["Code-split chunks\nper page"]
    B --> C["lazyWithRetry\nChunkLoadError recovery"]
    C --> D["Suspense boundary\nspinner fallback"]
    D --> E["Feature-gated routes\nVITE_FEATURE_* flags"]
```

### Backend (Railway Docker image)

- Railway deploys the API using the root [`Dockerfile`](Dockerfile). It installs the backend
  workspace **and** the `shared/` workspace (required for `@eshop/locales` and
  `shared/profanity/profanityFilter.js` with its `bad-words` dependency), then runs
  `backend/src/server.js` in production mode.
- Useful commands:

  ```bash
  docker build -t eshop-backend:latest .
  docker run --rm -p 5000:5000 -e NODE_ENV=production eshop-backend:latest
  ```

- Required Railway variables:

  | Key                          | Example                                                    | Notes                                                 |
  | ---------------------------- | ---------------------------------------------------------- | ----------------------------------------------------- |
  | `SESSION_SECRET`             | `base64:Vh1YhGq7...`                                       | Generate a 32+ byte random string.                    |
  | `SESSION_REDIS_PREFIX`       | `session:`                                                 | Optional; keep consistent across envs.                |
  | `PUBLIC_CLIENT_FALLBACK_URL` | `https://vexflare.com/reset-password`                      | Must match the frontend fallback.                     |
  | `MOBILE_RESET_REDIRECT_URI`  | `vexflare://reset-password`                                | Fallback deep link for reset emails (see note below). |
  | `MOBILE_MAIL_CONFIRM_URI`    | `vexflare://mailing/confirm`                               | Deep link opened by mailing emails.                   |
  | `MAIL_CONFIRM_WEB_URL`       | `https://shop.example.com/mailing/confirm?token={{token}}` | Overrides the browser confirmation URL.               |
  | `MOBILE_OAUTH_REDIRECT_URI`  | `vexflare://oauth`                                         | Deep link used by OAuth providers.                    |

- **`MOBILE_RESET_REDIRECT_URI` note:** Newer app versions send a per-flavor `scheme` field
  (`vexflare`, `vexflare-internal`, or `vexflare-dev`) in the `POST /api/auth/forgot-password`
  request body. When present, the backend uses that scheme to construct the deep link instead of
  `MOBILE_RESET_REDIRECT_URI`. The env var remains the fallback for older app versions that do not
  send `scheme`. See [Password reset deep-link flow](#password-reset-deep-link-flow) for details.
- Redeploy whenever you change env vars, email templates, or native deep-link paths.

### Mobile app (Expo custom dev client)

- Expo project lives in `mobile/` with a custom dev client that registers the `vexflare://` scheme
  and Android intent filters.
- Install/update the dev client after native or deep-link changes:

  ```bash
  npm -w mobile run android     # builds & installs via Gradle
  npm -w mobile run env:tunnel  # copy HTTPS API profile for remote testing
  npm -w mobile run start:tunnel
  ```

- Deep-link smoke test:

  ```bash
  npx uri-scheme open "vexflare://reset-password?token=TEST123&email=user%40example.com" --android
  ```

Refer to [`mobile/env.md`](mobile/env.md) for the full run-mode matrix and env profiles.

---

## Mobile deep-link flow overview

The app returns from external browsers (Stripe Checkout, Google/Facebook OAuth) into the native app
using **two distinct deep-link strategies**, because the two flows hand control back to the app in
fundamentally different ways. Both work across the dev client over tunnel, the internal APK, and
production with a single shared code path each.

### Payment return (Stripe checkout) — direct 303 redirect

- The mobile client builds the return link with `createReturnDeepLink()`
  (`mobile/src/utils/appScheme.js`) and sends it as `successUrl` / `cancelReturnUrl` to
  `POST /payments/create-checkout-session`.
- `createReturnDeepLink()` is **environment-aware** (it does _not_ force a scheme):
  - **Dev server present** (Expo Go / dev client / tunnel — detected via
    `Constants.expoConfig.hostUri`): it uses Expo's bare `ExpoLinking.createURL(path)`, which the
    running client actually re-opens — `vexflare://checkout-success` on a development build, or
    `exp://<host>/--/checkout-success` under Expo Go. Forcing the per-flavor scheme here would emit
    `vexflare-dev://…`, which the tunnelled dev client cannot open (Stripe's redirect dead-ends with
    "this site can't be reached").
  - **Installed standalone build** (no dev server): it forces the active flavor's scheme via
    `getAppScheme()` — `vexflare://` (production), `vexflare-internal://` (internal APK),
    `vexflare-dev://` (dev build) — so the redirect re-opens the exact app that started checkout.
- The backend validates the link against `ALLOWED_RETURN_PROTOCOLS`
  (`backend/src/controllers/payment.controller.js`) and passes it **directly to Stripe as
  `success_url`** — there is **no `/checkout-return` server bridge**.
- Stripe performs a **server-side 303 redirect** to that deep link. A Chrome Custom Tab honors a 303
  all the way to a registered custom scheme, so the OS re-opens the app on `PurchaseSuccessPage`,
  which calls `GET /payments/checkout-success` to confirm the order.
- **Cold-start safety:** `PurchaseSuccessPage` (`mobile/src/screens/PurchaseSuccessPage.js`)
  restores the Bearer token before issuing authenticated requests, so a deep-link cold start cannot
  fire a request without auth.

### OAuth login (Google/Facebook) — in-process interception

- The mobile client opens the provider with `WebBrowser.openAuthSessionAsync(target, redirectUrl)`
  (`mobile/src/screens/LoginScreen.js`, `mobile/src/screens/SignupPage.js`), passing
  `createAppDeepLink('oauth')` as the watched `redirectUrl`. `expo-web-browser` intercepts the
  redirect **in-process** — the OS never has to resolve the scheme from a Custom Tab.
- `createAppDeepLink()` **always forces the per-flavor scheme** via `getAppScheme()`:
  `vexflare://oauth` (production), `vexflare-internal://oauth` (internal APK),
  `vexflare-dev://oauth` (dev build), and the `exp`/`exp+vexflare-mobile` dev-client scheme under
  Expo tooling. Forcing the flavor scheme is safe here precisely because the redirect is intercepted
  in-process.
- After the provider callback the backend redirects to the per-flavor deep link via a bridge page
  (`buildMobileTarget` + `sendMobileBridgePage` in `backend/src/app.js`). On Android the bridge
  emits an **Android Intent URL**
  (`intent://oauth?code=…#Intent;scheme=vexflare;package=com.ahmedmonib.eshop;end`) so the redirect
  survives Chrome Custom Tabs' block on gesture-less custom-scheme navigation.
- The returned URL carries a short-lived, one-shot HMAC **bridge code** (not raw tokens);
  `AuthProvider` exchanges it via `POST /api/auth/oauth-signup`
  (`backend/src/controllers/auth.controller.js`). See the bridge-code details below.

### Why two strategies?

- **Payment** cannot intercept in-process — Stripe owns the hosted checkout page and issues a real
  303 redirect. The link therefore has to be a scheme the **OS** resolves, which is why the dev
  client must use Expo's environment-aware (generic / `exp`) link rather than a forced
  `vexflare-dev://`.
- **OAuth** uses `openAuthSessionAsync`, which resolves the redirect inside the browser session, so
  it can (and should) use the **flavor-specific** scheme to disambiguate between build types when
  several flavors are installed side by side.

### Supported return schemes

```text
Payment — ALLOWED_RETURN_PROTOCOLS (backend/src/controllers/payment.controller.js)
  http:                      web fallback
  https:                     web fallback
  vexflare:                  production APK + dev-client generic link
  vexflare-internal:         internal APK
  vexflare-dev:              development build
  eshop:                     legacy, kept for backward compatibility
  exp* (startsWith 'exp')    exp:// (Expo Go) and exp+vexflare-mobile:// (dev client launcher)

OAuth — mobile deep-link helpers (getAppScheme / createAppDeepLink)
  vexflare://oauth                     production APK
  vexflare-internal://oauth            internal APK
  vexflare-dev://oauth                 development build
  exp+vexflare-mobile://…/oauth        Expo dev client
  exp://…/oauth                        Expo Go

Password reset — ALLOWED_MOBILE_SCHEMES (backend/src/controllers/auth.controller.js)
  vexflare://reset-password            production APK
  vexflare-internal://reset-password   internal APK
  vexflare-dev://reset-password        development build
  eshop://reset-password               legacy / env var fallback
```

### Key files

```text
mobile/src/utils/appScheme.js        getAppScheme() → flavor scheme (OAuth, payment, reset);
                                     createReturnDeepLink() → env-aware link (payment);
                                     isDevServerEnvironment() → Constants.expoConfig.hostUri probe
mobile/src/screens/CheckoutPage.js   builds the payment return link via createReturnDeepLink()
mobile/src/screens/LoginScreen.js    OAuth via openAuthSessionAsync + createAppDeepLink('oauth')
mobile/src/screens/SignupPage.js     same OAuth pattern for the signup path
mobile/src/screens/ForgotPasswordPage.js  sends scheme: getAppScheme() to forgot-password endpoint
mobile/src/auth/AuthProvider.js      handles the intercepted OAuth URL + bridge-code exchange
backend/src/controllers/payment.controller.js  ALLOWED_RETURN_PROTOCOLS + success_url passthrough
backend/src/app.js                   OAuth provider callback → per-flavor deep-link bridge page
backend/src/controllers/auth.controller.js     ALLOWED_MOBILE_SCHEMES + flavor-specific reset link;
                                               POST /api/auth/oauth-signup bridge-code exchange
frontend/src/pages/MobileResetPasswordPage.jsx  bridge page: auto-opens deep link, web fallback
```

### Testing matrix

```text
Environment            Payment flow                          OAuth flow                                 Password reset
---------------------  ------------------------------------  -----------------------------------------  -------------------------------------------
Production APK         works — vexflare:// (flavor scheme)   works — vexflare:// (flavor scheme)        works — vexflare:// (flavor scheme)
Internal APK          works — vexflare-internal://          works — vexflare-internal://               works — vexflare-internal://
Prod Debug            works — vexflare-dev://               works — vexflare-dev://                    works — vexflare-dev://
Dev client + tunnel   works — generic vexflare:// (bare     works — vexflare-dev:// / Expo launcher,   falls back to MOBILE_RESET_REDIRECT_URI
                      createURL); SERVER_URL must be the    intercepted in-process by                  (dev server env does not send scheme)
                      ngrok / HTTPS URL                     openAuthSessionAsync
Expo Go               limited — exp:// may work, not        limited — openAuthSessionAsync has         limited — falls back to env var
                      officially supported                  constraints, not officially supported
```

Notes:

- **Dev client + tunnel** requires `SERVER_URL` (backend) to be the public ngrok / HTTPS origin so
  Stripe can reach it and the 303 returns to the device.
- **Multiple flavors installed:** OAuth and password reset both rely on the flavor-specific scheme
  to disambiguate; the installed-build payment link is also flavor-specific, while the dev-client
  payment link is generic.
- **Cold-start auth races** are handled by `PurchaseSuccessPage` restoring Bearer auth before
  authenticated calls.

### Password reset deep-link flow

The mobile password reset uses **flavor-specific deep links** so tapping the reset email opens only
the app that initiated the request, with no Android chooser dialog.

1. `ForgotPasswordPage` sends `POST /api/auth/forgot-password` with
   `{ email, client: 'mobile', mobile: true, scheme: getAppScheme() }`. The `scheme` field carries
   the active flavor's scheme (`vexflare`, `vexflare-internal`, or `vexflare-dev`).
2. The backend validates `scheme` against a hardcoded whitelist (`vexflare`, `vexflare-internal`,
   `vexflare-dev`, `eshop`). If valid, it constructs the deep link as
   `{scheme}://reset-password?…#token=…`. Invalid or missing schemes fall back to the
   `MOBILE_RESET_REDIRECT_URI` env var (default `eshop://reset-password`), preserving backward
   compatibility with older app versions.
3. The deep link is wrapped in a bridge URL
   (`https://vexflare.com/mobile/reset-password?…&deepLink={scheme}://…&webFallback=…`).
4. The user taps the email link → the frontend bridge page (`MobileResetPasswordPage.jsx`) loads →
   after 300 ms it calls `window.location.assign(deepLink)` to trigger the native deep link. If the
   app does not open within 1.8 s, the page falls back to the web reset form.
5. Android resolves the flavor-specific scheme (e.g. `vexflare-internal://`) and opens only the
   matching app — no chooser.

**Key files:**

```text
mobile/src/screens/ForgotPasswordPage.js     sends scheme in the request body
mobile/src/utils/appScheme.js                getAppScheme() → flavor scheme
backend/src/controllers/auth.controller.js   validateMobileScheme() + buildMobileResetDeepLink()
frontend/src/pages/MobileResetPasswordPage.jsx  bridge page (auto-opens deep link)
shared/auth/deepLinkValidator.js             extractResetParamsFromUrl() (scheme-agnostic)
```

### Other deep-linked flows

- Android intent filters in `mobile/app.config.js` accept the `vexflare://` scheme and route to the
  Expo app. Per-flavor variants (`vexflare-internal://`, `vexflare-dev://`) are resolved at runtime
  by `src/utils/appScheme.js` based on `applicationId`.
- Navigation prefixes (`AppNavigator.js`): the active flavor's `${getAppScheme()}://` plus the
  static set `['eshop://', 'vexflare://', 'exp+eshop-mobile://', 'exp+vexflare-mobile://']` and the
  dynamic `ExpoLinking.createURL('/')`, with screen config that maps `checkout-success` →
  `PurchaseSuccess`, `checkout-cancel` → `PurchaseCancel`, and `reset-password` → `ResetPassword`
  (among others).
- `extractResetParams` normalizes `token`/`code` query params and surfaces them as React Navigation
  route params.
- **Reset-password token delivery:** the reset token is delivered in the URL **fragment**
  (`#token=…`) rather than the query string. `AppNavigator` parses the fragment first with a
  query-string fallback for backward compatibility with older email clients.
- `ResetPassword` screen lives in both auth and authed stacks so the user always lands on the
  correct screen whether the app is cold-started or already in memory.
- **OAuth signup bridge-code flow:** after OAuth authentication the backend issues a short-lived,
  one-shot HMAC-signed bridge code and redirects to the mobile app. On Android, the bridge page
  emits an **Android Intent URL**
  (`intent://oauth?code=…#Intent;scheme=vexflare;package=com.ahmedmonib.eshop;end`) instead of a raw
  custom-scheme redirect, because Chrome Custom Tabs block auto-redirects to custom schemes.
  `AuthProvider` exchanges the code via a server-side API call instead of reading
  `accessToken`/`refreshToken` directly from query parameters. This prevents token leakage through
  server logs, referrer headers, and deep-link history.
  - Bridge codes are signed with `OAUTH_BRIDGE_SECRET` and expire after `OAUTH_BRIDGE_TTL_SECONDS`
    (default `300` seconds).
  - The `/auth/oauth-signup` endpoint is rate-limited to 5 requests per minute.
  - The OAuth nonce is cleared only after a successful bridge exchange. If the exchange fails
    (network error, server 5xx), the nonce survives so the user can retry without being silently
    trapped; a localized error toast is shown instead. Concurrent deep-link deliveries are deduped
    to prevent double-exchange attempts.
  - See [`docs/security/webhook-rotation.md`](docs/security/webhook-rotation.md) for
    `OAUTH_BRIDGE_SECRET` rotation instructions.
- Metro logs beginning with `🔗` confirm the linking handlers fired; re-run
  `npx uri-scheme open ...` to verify changes.

---

## Mobile app architecture & features

### Key capabilities

- **Shared domain logic:** Mobile consumes the same REST API as the web frontend through the
  `packages/api-client` workspace, reusing authentication flows, cart logic, and localization from
  the shared bundle.
- **Offline-friendly UX:** Secure tokens are cached in Expo SecureStore with background refresh
  logic so shoppers remain signed in across cold starts. On app restart, the bootstrap layer checks
  whether the stored access token is expired and silently refreshes it before fetching the user
  profile. Network errors fall back to cached user data; a `401` triggers re-authentication.
- **Proactive session heartbeat:** The app calls the refresh-token endpoint every 10 minutes and on
  each app foreground event to keep the session alive within the 30-day sliding window. The backend
  keep-alive endpoint accepts `Authorization: Bearer <refresh-token>` in addition to cookies, so the
  heartbeat works in mobile runtimes where cookies may not be propagated automatically.
- **Full shopping journey:** The app surfaces the entire catalogue, promotional sliders, cart,
  checkout, order history, profile management (including GDPR data export and account deletion), and
  transactional policy screens.
- **Return request (mobile parity):** `ReturnRequestModal` (`mobile/src/components/`) provides the
  full return form on mobile: reason picker, notes textarea, evidence photo upload
  (expo-image-picker), and COD bank fields rendered conditionally on payment method. Uses
  `useReturnRequestForm`, `useReturnReasons`, and `useCodRefundConfig` hooks. Wired into
  `OrderHistoryPage.js` via the "Request Refund" button in `OrderDetailsModal`. Posts FormData to
  the same backend endpoint as web.
- **StorefrontScreen:** `mobile/src/screens/StorefrontScreen.js` (deep-link route `stores/:slug`)
  shows the store banner, logo, name, description, rating aggregate, top-level category cards with
  breadcrumb drill-down, store-scoped search bar, a 2-column product grid (FlatList), About section,
  and a Policies modal. Registered in `AppNavigator.js`.
- **Storefront link on ProductScreen:** `ProductScreen.js` shows the store logo and name below the
  product title as a touchable that navigates to `StorefrontScreen` with the store slug.
- **Radial gradient background:** `ThemeRadialBackground` is applied as a full-screen overlay in
  `AppNavigator.js`, providing the emerald gradient that matches the web frontend across all
  screens.
- **Theme cold-start safety:** `ThemeContext` defaults to `'light'` when `useColorScheme()` returns
  `null` during cold start (common on deep-link launches) and adopts the real system scheme once
  resolved. This prevents wrong theme colors from flashing before the system preference is
  available.
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
  webhook handlers, csrf-csrf (double-submit CSRF protection), passport strategies, email helpers.
  Deployed with the root Dockerfile to Railway.
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
  - Sliding 30-day session window (`session_active_since:<userId>`) — extended on every activity.
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
- **Order status transition enforcement** — backend `ALLOWED_TRANSITIONS` map blocks invalid status
  jumps; seller/admin dropdowns surface only the next valid statuses; COD orders in early statuses
  can be cancelled without requiring payout details; friendly error toasts replace raw crashes.
- **Carrier shipping integration** — dedicated Shipment model (separate from Order), provider
  adapter pattern (disabled/manual/Aramex), seller shipment creation UI with label generation,
  visual customer tracking timeline (`OrderShipmentTimeline`), carrier webhook hub with HMAC
  verification, and admin shipment visibility panel.
- **Product & store reviews** — purchase-gated review submission (one per order line), 1–5 star
  rating with aggregate denormalization on Product and Store, automated profanity/link filter
  (shared across web, mobile, and backend), admin moderation panel (flag/reinstate/hide), seller
  review inbox, and email notification to seller on publish.
- **Dispute resolution** — post-fulfilment dispute lifecycle (7 state machine statuses), append-only
  message threads with evidence upload (Cloudinary), seller resolution offers (accept/reject), admin
  escalation with resolve/close controls, canned response templates, dispute metrics KPIs, 72-hour
  SLA enforcement cron, and Stripe chargeback webhook integration (`charge.dispute.created/closed`).
  Gated by `FEATURE_DISPUTES_V1`. Full customer, seller, and admin panels on web and mobile.
- **Transactional email system** — 11 event-driven emails for order shipped/delivered, refund
  approved/rejected (customer + seller copy), payout completed/failed, product moderation
  (approved/rejected/action required), and KYC status changes. All use fire-and-forget delivery via
  Resend; anonymised customers are silently skipped; failures never block the parent operation.
- **In-app notification centre** — `Notification` model with i18n-key-based rendering (22 types).
  Web bell with red badge, dropdown panel, 60 s polling, and click-to-open detail modal. Mobile bell
  with bottom sheet and AppState-aware polling. Gated by `FEATURE_IN_APP_NOTIFICATIONS`.
- **Admin user management** — list, suspend, unsuspend, and delete users from the admin panel.
  Suspended accounts cannot log in and have sessions immediately invalidated. Admins cannot act on
  themselves or other admins. Schema additions: `suspendedAt`, `suspendedReason`, `suspendedBy`,
  `lastLoginAt`.
- **Admin audit-log dashboard** — read-only tab exposing five audit types (seller lifecycle, seller
  profile access, COD refund access, user deletion, document access) with date-range filtering,
  pagination, CSV export, and batch-resolved human-readable identifiers. Gated by
  `FEATURE_ADMIN_AUDIT_DASHBOARD`.
- **Fully responsive Admin Dashboard** — manage orders, add products, edit variants, and run the
  store from a **phone or tablet** (mobile-friendly layouts and inputs).
- Privacy & Compliance

- GDPR-ready: user data export endpoint and account deletion/anonymization tooling.
- Observability & Security
  - Sentry client & server integration, Winston logging with daily rotation and PII redaction.
  - Rate limiting, input sanitization, NoSQL injection protection.

- Testing
  - Extensive unit/integration/E2E tests: 3,333 backend tests across 235 suites with 90.02% line
    coverage, plus 340 frontend tests across 47 suites. See [Testing & CI](#testing--ci).

---

## Per-store shipping & admin delivery regions

Checkout shipping is computed per store and per destination by
`backend/src/services/shipping/shippingRate.service.js` (`resolveStoreShipping`). It is destination-
aware: nothing is charged until the customer enters a country (see
`feat(checkout): calculate shipping after the customer enters a destination`).

### Per-store rate configuration (`store.shipping`)

Each store document carries a `shipping` sub-document the seller edits in **Store settings**:

```
mode                  'flat' | 'tiered' | 'carrier'   (default 'flat')
currency              ISO currency for the store's rates (default 'USD')
flatRate              flat per-order shipping cost (null = use marketplace default)
freeShippingThreshold subtotal at/above which shipping is free (null = off)
handlingFee           added to every computed rate (default 0)
regions[]             per-region tiers: { countries[], mode?, rate, freeOver }. Optional per-region
                      `mode` ('flat'|'tiered'|'carrier') overrides the store-level mode for that
                      region (unset = inherit). Lets one store be carrier (card-only) for some
                      regions and flat/tiered (COD-allowed) for others.
excludedCountries[]   legacy ISO-2 country opt-outs (still honoured)
excludedRegions[]     admin-region codes the seller opts out of: "EG" or "EG-ALX" / "US-CA"
carrier{}             Carrier-calculated config: provider, ship-from origin (5 fields), fallbackRate,
                      defaultParcel. Ship-from origin is required before a store can go active (see below).
```

Resolution order in `resolveStoreShipping`:

1. **Platform deliverability** — `resolvePlatformDeliverability` checks the destination against the
   admin operating regions (below). A blocked destination returns an exclusion marker
   `{ excluded, reason: 'not_operating', field, value }` (no price).
2. **Seller opt-out** — if the seller excluded the resolved country/subdivision (`excludedRegions`
   or legacy `excludedCountries`), returns `{ excluded, reason: 'seller_excluded', field, value }`.
   The block carries the exact field (`country` | `state` | `city`) and the value the customer
   typed.
3. **Effective mode** — `resolveEffectiveShippingMode(shipping, destination)` matches the
   destination to a region (`selectRegionForDestination`) and returns
   `region.mode ?? shipping.mode`. This is the mode used for rating, the `requiresPostalCode`
   carrier gate, the order snapshot (`order.shippingMode`), and the COD gate below.
4. **Rate** — a matched region's `rate` is used when the store is `tiered` (legacy) **or** the
   region sets its own `mode` (so a flat-default store with stray, mode-less regions is unaffected);
   else the store `flatRate`, else the marketplace default `SHIPPING_COST` (env, fallback **35**).
   When the effective mode is `carrier`, `resolveStoreShippingRate` performs the live Shippo lookup.
5. **Free shipping** — if `subtotal >= freeShippingThreshold` (or the matched region's `freeOver`),
   the rate is a true `0` (bypasses the handling fee and the admin clamp).
6. **Handling fee + admin clamp** — `handlingFee` is added, then the result is clamped to
   `SHIPPING_MIN_RATE` / `SHIPPING_MAX_RATE` when those env bounds are set.

**Region-aware COD:** Cash on Delivery is card-only for any destination whose effective mode is
`carrier` (the platform pays the carrier label, so there's no cash for the seller to collect). The
checkout estimate returns the effective `mode`; the web/mobile UIs disable the COD option (with a
note) for carrier destinations, and `createCODOrder` rejects such orders with
`cod_not_available_carrier`. This server-side rejection is gated by `FEATURE_REGION_SHIPPING_MODE`
(the schema/resolver/UI are additive and always on).

The marketplace default (`SHIPPING_COST`, `SHIPPING_DEFAULT_CURRENCY`) is surfaced to sellers via
`getShippingDefaults` so the Store-settings UI shows the fallback that applies when a field is
blank, and is displayed on the storefront/checkout.

### Seller shipping UI

The **Shipping** collapsible section of Store Settings (`SellerStoreSettings.jsx`) presents a
single, unified view — there are no per-mode tabs. Top to bottom:

- **Ship-from address** — always visible, independent of mode. Street autocompletes via
  `AddressAutocomplete` (backend Places proxy); city/state/postal/country fields plus a
  draggable-pin `AddressMap` fill in the rest. Required before the store can go `active` (see
  [Ship-from address requirement](#ship-from-address-requirement)).
- **Default shipping rate** — the store's flat rate and free-shipping threshold. The flat rate
  "applies when no regional tier matches, or when a tier uses 'Use store default'"; the free-over
  threshold makes orders at or above that subtotal ship free (leave empty to disable). The
  store-level `shipping.mode` itself has no toggle in this UI — it is preserved from existing data,
  and a tier's "Use store default" inherits it.
- **Regional rates** — always shown, with an **Add region** button. Each tier has:
  - **Ships to** — one or more regions picked from the admin's deliverable-region list
    (`GET /api/public/delivery-regions`), shown as removable chips with real country/subdivision
    names (e.g. "Egypt", "Egypt — Cairo"), not raw ISO codes. An empty list makes the tier the
    rest-of-world catch-all.
  - **Mode** — `Use store default` / Flat rate / Regional tiers / Carrier rates. Selecting "Use
    store default" omits the tier's `mode` from the save payload, so it inherits the store-level
    mode.
  - **Rate** and **Free over** — shown for flat/tiered tiers. Hidden for any tier whose _effective_
    mode is **Carrier rates** (its own mode is `carrier`, or it inherits a `carrier` store default),
    since Shippo computes the live rate for those destinations and a manual rate would be ignored.
    Carrier tiers are also card-only — Cash on Delivery is unavailable for them (see
    [Region-aware COD](#per-store-rate-configuration-storeshipping)).
  - **Remove** button.

  On load, stored backend tokens (country codes/names and subdivision names) are mapped back to
  picker option codes; on save, picker codes are expanded to backend tokens — a country becomes
  `[countryCode]`, a subdivision becomes `[countryCode, subdivisionName]` so
  `selectRegionForDestination` (which matches subdivisions by name) resolves the tier at checkout.
  Legacy free-text tokens that don't match a current admin region round-trip unchanged as raw chips.
  Regions are always sent on save (array-level edit).

- **Carrier rates (Shippo)** — a collapsible sub-panel containing the fallback rate and default
  parcel dimensions (weight in grams, dimensions in cm, used when a product has no weight/dimensions
  of its own). It auto-expands when the store-default mode or any tier's mode is "Carrier rates"
  (still manually collapsible otherwise). The ship-from origin lives in the section above, not here.
- **Regions I don't ship to** — the seller's opt-outs, picked from the same admin deliverable-region
  list as the regional-rate tiers.

### Admin-managed delivery regions (operating countries + subdivisions)

`DeliveryRegion` (`backend/src/models/deliveryRegion.model.js`) is the admin-managed list of where
the platform operates. One document per country:

- `countryCode` — ISO-3166 alpha-2, or the sentinel `*` for the **Rest of World** catch-all (when
  that document is enabled, any country not explicitly listed ships at country level).
- `subdivisions[]` — `{ code, name, aliases[], enabled }`. Disabling a subdivision excludes it (e.g.
  operate in EG but exclude Alexandria, or US but exclude Alaska/Hawaii).
- `deletedSubdivisions[]` — **tombstones** for subdivisions the admin removed. They still block
  delivery to that name, so deleting a major governorate doesn't silently re-open it as if it were
  an unknown small city. Re-adding the subdivision clears its tombstone.

State/city matching uses the **shared address normalizer** (see the tax section): a destination
state is a whitelist — it must resolve to an enabled subdivision (by code, name, or alias) or the
order is blocked. When the state field is empty, the **city** field is matched as a fallback (so
typing a governorate name in the city box still resolves). Matching is suffix-, article- and
Arabic-insensitive (`Alexandria Governorate`, `Al Iskandariyah`, and `الإسكندرية` all resolve to the
same subdivision).

Sellers inherit the admin set: the seller region selector only offers admin-enabled regions, and
`excludedRegions` holds the codes a seller opts out of.

### Endpoints & seed

```
GET  /api/shipping/estimate?storeId=&subtotal=&country=&state=&city=
       -> { shippingCost, currency, freeThreshold?, mode }  OR
          { shippingCost:null, requiresPostalCode:true, currency, mode }  OR
          { excluded:true, reason, field, value, currency }   (informational; checkout re-computes)
GET  /api/public/delivery-regions          enabled regions for the seller selector / dropdowns
GET  /api/admin/delivery-regions           admin list (admin only)
POST /api/admin/delivery-regions           create
PUT  /api/admin/delivery-regions/:id        update (enable/disable subdivisions, edit aliases)
DELETE /api/admin/delivery-regions/:id      delete (tombstones its subdivisions)
```

Seed the initial operating set (idempotent — preserves admin enabled flags, only adds missing
subdivisions, refreshes names/aliases) with **`npm run seed:delivery-regions`** (from `backend/`).
It seeds EG governorates (with Arabic aliases), US states, and the Rest-of-World catch-all. The
admin UI is the **Delivery Regions** tab (`AdminDeliveryRegionsPanel`).

### Ship-from address requirement

Every store must have a complete ship-from origin (street, city, state, postal code, country —
stored in `store.shipping.carrier.origin*`) before it can be set to `active`. The activation
endpoint (`PUT /api/stores/:id/status`) returns `400 ship_from_required` when any field is blank,
unless the caller is an admin (admin can activate on behalf of a seller during onboarding). Stores
that were already active before this requirement was introduced receive `shipFromComplete: false` in
the store response and show a dismissible amber warning banner in the seller dashboard. The origin
address is entered in the **Ship-from address** section of Store Settings (always visible regardless
of shipping mode) using the same `AddressAutocomplete` + draggable-pin `AddressMap` components used
at checkout. The origin is also consumed by the tax engine when Avalara is active — see
[`docs/tax-book.md §9`](docs/tax-book.md#9-ship-from-address-and-avalara-nexus).

---

## Category-based tax engine (internal rules + Avalara)

Tax is computed per order line by category and destination jurisdiction
(`backend/src/services/tax/`). The full admin guide is in [`docs/tax-book.md`](docs/tax-book.md);
this is the architecture summary.

### Internal `TaxRule` engine

A `TaxRule` (`backend/src/models/taxRule.model.js`) is
`{ jurisdiction:{ country, state }, categoryKey, rate, taxExempt, avalaraTaxCode, effectiveStartDate, effectiveEndDate, enabled }`.
`country` is ISO-2 or `*` (global default); `state` is a subdivision code or `null` (country-level).

- **Category key per line** — `resolveLineTaxCategory(product)` derives the key: an explicit,
  non-`standard` product `taxCategory` wins; otherwise the product's `subCategorySlug`, then
  `categorySlug`, then `standard`. So an admin rule keyed `dress` applies to dress products even
  when the seller never set `taxCategory` (which defaults to `standard`).
- **Precedence (`findTaxRule`)** — a single combined score `categoryExact*10 + jurisdiction` where
  jurisdiction is `state 3 > country 2 > global '*' 1`. An exact category match always beats a
  `standard` fallback within the same jurisdiction, and `standard` is the fallback when no
  category-specific rule exists. Category keys are normalised before comparison (gender-prefix strip
  `Men's`/`Women's`/`Kids'`, plural/hyphen folding) so a rule `"Men's Tshirts"` matches a product
  key `"t-shirt"`. The destination country is resolved to its ISO code (so a typed `"Egypt"` matches
  `EG`), and the state is matched through the shared normalizer against the subdivision catalog (so
  `"Alexandria Governorate"`, `"Al Iskandariyah"`, `"الإسكندرية"` all match a rule state
  `ALEXANDRIA`); an empty state falls back to the city.
- **No match** → the engine uses the env `TAX_RATE` (default `0.14`).

`resolveOrderTax` returns
`{ taxAmount, perLine:[{ productId, taxCategory, taxableBase, rate, taxAmount, exempt }], jurisdiction:{ country, state }, provider }`.

### TaxProvider interface, Avalara, and the fallback chain

`getTaxProvider()` is selected by `TAX_PROVIDER` (`internal` | `avalara`). Every provider's
`quote()` returns the same shape, so checkout/order/refund code is provider-neutral.

- **internal** — the `TaxRule` engine above (default).
- **avalara** — `services/tax/providers/avalara.provider.js` posts a non-committing `SalesOrder` to
  AvaTax `POST /transactions/create` with each line's Avalara tax code, the destination address, and
  `companyCode` (`AVALARA_COMPANY_CODE`, default `DEFAULT`). The base URL is chosen from
  `AVALARA_ENVIRONMENT` (`sandbox` → `sandbox-rest.avatax.com`, `production` → `rest.avatax.com`).

**Fallback chain: Avalara → internal `TaxRule`s → env `TAX_RATE`.** On _any_ Avalara failure (no
creds, 4xx/5xx, network/timeout, trial expired) the avalara provider catches and returns the
internal result, which itself falls back to `TAX_RATE` — so the tax provider can never break
checkout.

**Cross-border zero-rating (internal engine).** When the store's ship-from country does not match
the destination country, the internal engine returns `$0` tax — cross-border sales are zero-rated in
the internal fallback. Avalara handles cross-border jurisdiction natively when active.

**Avalara rate-limit resilience.** The `POST /api/tax/estimate` endpoint is protected by a 2 s
debounce on the frontend (fires after the last address keystroke), a 5-character street guard (skips
the call when the street field is too short), and a 60 s in-memory cache on the Avalara provider
keyed by the full address + items fingerprint — so repeated estimates for the same cart do not count
against the Avalara rate limit.

**Environment auto-correction** — these credentials must match the tier they belong to. If a request
401s against the configured environment, the provider transparently retries the _other_ environment
once; if the credentials authenticate there it uses that result and logs
`Set AVALARA_ENVIRONMENT=<env>`. The health probe does the same.

**Tax-code mapping** — `services/tax/providers/taxCodeMap.js` maps a platform category to an AvaTax
code (`standard`/`digital`/`exempt` → `P0000000`, `food` → `PF050032`, apparel → `PC040100`,
footwear → `PC040108`, unknown → `P0000000`). An admin can override per rule via
`TaxRule.avalaraTaxCode`.

**Health** — `getTaxProviderHealth` (60s TTL cache) pings `GET /utilities/ping`; "healthy" requires
a 2xx **and** `authenticated:true`. The **Tax Rules** admin panel shows a green/red provider badge.

### Order tax audit

Every order persists
`order.tax = { provider, jurisdiction:{ country, state }, breakdown:[…perLine], meta }`. `meta`
holds the provider response metadata (for Avalara: `taxSummary`, `jurisdictions`, transaction code,
timestamp; `null` for internal — it's a `Mixed` field so a provider swap needs no schema change).

### Endpoints & UI

```
POST /api/tax/estimate          { items:[{ productId, quantity, variantAttributes }],
                                  storeId,                               (optional but recommended —
                                    used to load the store's ship-from origin so Avalara receives the
                                    correct nexus address; falls back to the storeId stored on the
                                    product document)
                                  destination:{ country, state, city, street, postalCode } }
                                -> { taxAmount, perLine, jurisdiction, provider }   (informational)
GET  /api/public/tax-categories -> { data, groups:{ special[], product[] } }  (seller product form +
                                   admin rule dropdown; product categories come from the category tree)
GET  /api/admin/tax-rules        admin CRUD (admin only)
POST|PUT|DELETE /api/admin/tax-rules[/:id]
GET  /api/admin/tax/health      -> { provider, healthy, environment?, status?, reason? }
```

The cart/checkout shows tax **per product** (rate + amount under each line, plus a "Tax Total") when
products are taxed at different rates, and a single summary line when all rates match. The tax line
is deferred until a destination is entered (it shows a placeholder before that). Products carry a
`taxCategory` key (default `standard`) editable on the seller product form.

---

## Address autocomplete & draggable-pin checkout map

### Google Places autocomplete via a backend proxy

The checkout street field autocompletes through the **backend proxy** (not the browser SDK) on both
web (`AddressAutocomplete.jsx`) and mobile (`AddressAutocomplete.js`):

```
GET /api/shipping/places/autocomplete?input=   -> [{ id, text }]
GET /api/shipping/places/details?placeId=      -> { address:{ street, city, state, postalCode, country }, formattedAddress }
```

The proxy calls the **Places API (New)** REST endpoints with the server key
(`GOOGLE_PLACES_API_KEY`, falling back to `GOOGLE_MAPS_API_KEY`), so the browser avoids CORS /
HTTP-referrer-key restrictions and the server key is never exposed. Errors are surfaced verbatim
(e.g. `REQUEST_DENIED`, referrer-restriction messages) for diagnosis, and the field degrades to
plain manual entry when the proxy returns nothing.

### Draggable-pin map (web)

`frontend/src/components/AddressMap.jsx` shows a Google Map with a draggable pin below the street
field. It loads Maps JS via `frontend/src/lib/googleMaps.js` using `VITE_GOOGLE_MAPS_API_KEY`:

- With `loading=async`, nothing under `google.maps` exists until imported, so the loader
  `importLibrary`s **maps**, **geocoding**, and **marker** and only resolves once `google.maps.Map`
  & `Geocoder` exist (this fixed `google.maps.Map is not a constructor`).
- A **10 s timeout** and a `gm_authFailure` handler (referrer/auth) resolve to `null` so the UI
  never hangs; a `RefererNotAllowedMapError` is logged with the fix (allow-list the dev origin).
- The pin geocodes the entered address; dragging it reverse-geocodes the new position and updates
  the address fields, which re-runs the shipping/tax estimates. It uses `AdvancedMarkerElement` when
  a Cloud `VITE_GOOGLE_MAPS_MAP_ID` is set, otherwise the classic draggable `Marker`.
- The container is responsive (`h-80 sm:h-96`). If the map can't load it shows **"Map unavailable —
  enter your address manually."** and the fields stay editable.

### Draggable-pin map (mobile, WebView)

`mobile/src/components/AddressMapWebView.js` renders the same map inside a `react-native-webview`
(Expo-Go compatible, no native maps module). An **app-restricted** EXPO key fails inside a WebView
(Google treats the page as a foreign/null origin), so the mobile map **fetches the server key at
runtime** from `GET /api/shipping/maps/config` (`{ mapsApiKey, configured }` — the same unrestricted
server key). The page is loaded with a real `baseUrl` origin so the key may instead be HTTP-referrer
restricted. `gestureHandling: 'cooperative'` lets a one-finger drag scroll the checkout `ScrollView`
while two fingers pan the map and the marker stays draggable; the map is 280 px tall.
`gm_authFailure` / JS / script / 10 s-timeout errors post back to React Native and trigger the same
fallback message. `EXPO_PUBLIC_GOOGLE_MAPS_API_KEY` has been **removed** — the mobile app no longer
holds a Maps key (documented in `mobile/env.md`).

### Ship-from address autocomplete (seller settings)

`frontend/src/components/seller/SellerStoreSettings.jsx` reuses the same `AddressAutocomplete` and
`AddressMap` components for the seller's ship-from origin address. The street field autocompletes
via the same backend Places proxy; selecting a suggestion populates all five origin fields at once
(`originStreet/City/State/PostalCode/Country`). The draggable-pin map renders below the address grid
— dragging the pin reverse-geocodes and fills the same five fields. This section is always visible
in Store Settings regardless of the active shipping mode (flat/tiered/carrier). The values are saved
as part of the normal settings form and are read by the tax engine at checkout.

### Profile address management

Shipping and billing addresses entered at checkout are **auto-saved to the user profile** after a
successful order placement (fire-and-forget; never blocks or delays the checkout flow). On the next
checkout visit the saved address pre-fills the form.

The web **Profile page** (`frontend/src/pages/ProfilePage.jsx`) replaces the plain text inputs for
shipping and billing addresses with the same `AddressAutocomplete` + `AddressMap` components used at
checkout. When a profile address is saved, the checkout `sessionStorage` key is cleared so the fresh
address pre-fills on the next visit rather than showing stale session data. Profile addresses are
stored encrypted in the database; a missing `.lean()` call was causing encrypted fields to return as
raw objects — profile address endpoints now call `findOne()` (not `.lean()`) so decryption runs
correctly.

The **mobile** Profile screen (`mobile/src/screens/ProfileScreen.js`) uses the same
`AddressAutocomplete` component (via the backend proxy) for address editing.

---

## New environment variables (shipping, tax, address & resilience)

All optional unless noted. Backend values live in `backend/.env`; `VITE_*` build into the frontend,
`EXPO_PUBLIC_*` into the mobile bundle.

```
# ── Shipping ─────────────────────────────────────────────────────────────────
SHIPPING_COST                     Marketplace default per-order rate when a store sets no flatRate.
                                  Default 35. (Frontend mirror: VITE_SHIPPING_COST.)
SHIPPING_DEFAULT_CURRENCY         Currency shown for marketplace default. Default USD.
SHIPPING_MIN_RATE                 Optional admin floor; computed rate is clamped up to this.
SHIPPING_MAX_RATE                 Optional admin ceiling; computed rate is clamped down to this.

# ── Carrier / Shippo ────────────────────────────────────────────────
SHIPPING_PROVIDER                 Active carrier adapter: 'disabled' (dev default), 'manual',
                                  'aramex', or 'shippo'.
SHIPPO_API_KEY                    Shippo API key. Required when SHIPPING_PROVIDER=shippo. Secret.
SHIPPO_ENVIRONMENT                'sandbox' or 'production'. Default 'sandbox'.
FEATURE_SHIPPO_RATES              true → carrier-mode stores get live Shippo rates at checkout;
                                  false (default) → falls back to store.shipping.carrier.fallbackRate.
FEATURE_SHIPPO_TRACKING           true → GET /orders/:id/shipments/:shipId/tracking performs a live
                                  carrier sync; false (default) → returns persisted events only.
# Optional Shippo tuning:
# SHIPPO_API_BASE_URL=https://api.goshippo.com
# SHIPPO_TIMEOUT_MS=15000
# SHIPPO_MAX_RETRIES=3
# SHIPPO_RETRY_BASE_DELAY_MS=300
# SHIPPO_LABEL_FILE_TYPE=PDF
# SHIPPO_MAX_RATE_ATTEMPTS=6      # label purchase tries up to N carriers (skips unactivated ones)
# SHIPPO_TEST_TRACKING_NUMBER=SHIPPO_TRANSIT  # sandbox sample number for tracking demos

# ── Accounting / Liability reconciliation ────────────────────────────────────
FEATURE_SHIPPING_LABEL_LEDGER     true → post a `shipping_label_cost` ledger debit against the
                                  Shipping Liability seller when a Shippo carrier label is
                                  purchased. Off by default until backfill of historical shipments
                                  is decided. Requires SHIPPING_PROVIDER=shippo.
FEATURE_TAX_REMITTANCE            true → enable the TaxRemittance model and the admin
                                  tax-remittance API (`GET/POST
                                  /api/admin/platform-financials/tax-remittances`). Off by default.
FEATURE_LIABILITY_RECONCILIATION  true → start the monthly liability reconciliation cron job that
                                  snapshots shipping/tax collected vs. disbursed per currency and
                                  flags negative residuals. Off by default.
FEATURE_SELLER_SHIPPING_PAYOUT    true → flat/tiered (self-fulfilled) shipping is posted as a
                                  payout-eligible `shipping_revenue` credit to the seller
                                  (`shipping_revenue_refund` on refund) instead of the
                                  `shipping_fee`/Shipping Liability mirror pair. Carrier-mode
                                  shipping is unaffected — it stays a pass-through liability offset
                                  by the carrier label cost. Off by default; forward-only (applies
                                  to deliveries posted while on).
FEATURE_TAX_LIABILITY_SELLER      true → operator-side tax mirror rows (`tax`, `tax_refund`,
                                  `tax_remittance`) post against the dedicated Tax-Liability seller
                                  (TAX_SELLER_ID) instead of PLATFORM_SELLER_ID, separating the
                                  marketplace-wide tax clearing account from the platform store's
                                  own books. Off by default; forward-only (only entries posted
                                  while on use TAX_SELLER_ID). See docs/tax-book.md.
TAX_SELLER_ID                    MongoDB ObjectId of the Tax-Liability synthetic seller (slug
                                  `tax-liability`). Derived at boot by ensureAdminStore (like
                                  SHIPPING_SELLER_ID) — leave blank; do not hand-set. Only used as a
                                  posting target when FEATURE_TAX_LIABILITY_SELLER=true.
FEATURE_REGION_SHIPPING_MODE      true → checkout rejects Cash on Delivery when the customer's
                                  destination resolves to a carrier-mode region (a region whose
                                  effective `shipping.regions[].mode` is `carrier`), returning
                                  `cod_not_available_carrier`. The per-region `mode` schema,
                                  `resolveEffectiveShippingMode` resolver, and seller UI are
                                  additive and always-on regardless of this flag — only the COD
                                  rejection is gated. Off by default; flip per environment.
# Optional reconciliation job tuning:

Production

# LIABILITY_RECONCILIATION_CRON=30 3 1 * *   # keep the default — monthly accounting close, no benefit to running more often
# LIABILITY_RECONCILIATION_TIMEZONE=UTC      # keep UTC — matches SETTLEMENT_SCHEDULER_TZ and period-boundary convention
# LIABILITY_RECONCILIATION_RUN_ON_BOOT=false # multi-replica safe; only the cron fires
# LIABILITY_RECONCILIATION_PERIOD          # leave unset — defaults to previousMonthPeriod(), so each monthly run automatically reconciles "last month"

Development / local testing

# LIABILITY_RECONCILIATION_CRON=*/2 * * * *  # frequent cadence so you can see a snapshot get created/updated quickly
# LIABILITY_RECONCILIATION_TIMEZONE=UTC      # keep UTC for deterministic period math while testing
# LIABILITY_RECONCILIATION_RUN_ON_BOOT=true  # trigger an immediate run on server start for fast feedback (single-instance dev)
# LIABILITY_RECONCILIATION_PERIOD=2026-05    # pin to whatever month you've seeded ledger entries for; previousMonthPeriod() won't match seeded test data otherwise — remove this once you're done testing so it doesn't get left set


# ── Tax ──────────────────────────────────────────────────────────────────────
TAX_RATE                          Final fallback rate when no TaxRule/provider applies. Default 0.14.
                                  (Frontend mirror: VITE_TAX_RATE.)
TAX_PROVIDER                      'internal' (default) or 'avalara'.
AVALARA_ACCOUNT_ID                Avalara account id. Required when TAX_PROVIDER=avalara.
AVALARA_LICENSE_KEY               Avalara license key. Required when TAX_PROVIDER=avalara. Secret.
AVALARA_ENVIRONMENT               'sandbox' or 'production' — must match the credentials' tier
                                  (auto-retries the other env on a 401). Default 'sandbox'.
AVALARA_COMPANY_CODE              AvaTax company code. Default 'DEFAULT' (the generic value 404s on
                                  most accounts — set the account's real code).
AVALARA_TIMEOUT_MS                Avalara request timeout before falling back to internal. Default 8000.

# ── Address / maps ───────────────────────────────────────────────────────────
GOOGLE_PLACES_API_KEY             Server key for the Places proxy + the mobile map config endpoint.
                                  Falls back to GOOGLE_MAPS_API_KEY. Must NOT be HTTP-referrer
                                  restricted (server calls have no referrer); unrestricted or IP-bound.
GOOGLE_MAPS_API_KEY               Fallback server key for the above.
VITE_GOOGLE_MAPS_API_KEY          Web key for the Maps JS loader (AddressMap). HTTP-referrer restrict
                                  to your web origins; needs Maps JavaScript API + Geocoding API.
VITE_GOOGLE_MAPS_MAP_ID           Optional Cloud Map ID to enable AdvancedMarkerElement on web.
EXPO_PUBLIC_GOOGLE_MAPS_API_KEY   REMOVED — the mobile map now fetches the server key from
                                  /api/shipping/maps/config; do not set this.

# ── Analytics (frontend) ─────────────────────────────────────────────────────
VITE_GA_ID                        GA4 Measurement ID (e.g. G-XXXXXXXXXX), embedded at Vite build time
                                  (frontend/src/lib/analytics.js). When unset, analytics no-ops.
                                  Events fire only after analytics_storage consent is granted —
                                  GA4 Consent Mode v2 default-denied is pushed before gtag.js loads,
                                  and the cookie-consent decision is logged via POST
                                  /api/users/consent-log (see GDPR & user data export).

# ── COD / cache / Mongo resilience ───────────────────────────────────────────
REDIS_OP_TIMEOUT_MS               Per-op bound for cache reads/writes (stock, cart) so a slow Redis
                                  fails fast instead of hanging the request.
COD_REDIS_OP_TIMEOUT_MS           COD-flow override for the above (falls back to REDIS_OP_TIMEOUT_MS).
CACHE_REDIS_COMMAND_TIMEOUT_MS    ioredis (TCP cache driver) command timeout; offline queue disabled.
SESSION_REDIS_COMMAND_TIMEOUT_MS  Session Redis client command timeout (offline queue disabled).
UPSTASH_MAX_RETRIES               Caps Upstash REST retries so one outage costs at most one timeout.
MONGO_SERVER_SELECTION_TIMEOUT_MS Optional Mongo server-selection timeout (driver default otherwise).
MONGO_CONNECT_TIMEOUT_MS          Optional Mongo connect timeout (driver default otherwise).
```

> Note: custom MongoDB `socketTimeoutMS` / pool options were deliberately **removed** (they caused a
> recurring `MongoNetworkTimeoutError` crash); the driver defaults are used unless the two
> `MONGO_*_TIMEOUT_MS` vars above are explicitly set.

**Development profile** (relaxed bounds — local Redis/Upstash has low traffic; fail-fast matters
less)

```env
# COD / cache resilience (dev): looser timeouts for local iteration and cold-start tolerance
REDIS_OP_TIMEOUT_MS=3000              # 2× code default (1500); local Redis rarely stalls
COD_REDIS_OP_TIMEOUT_MS=1500          # relaxed to match general dev timeout
CACHE_REDIS_COMMAND_TIMEOUT_MS=3000   # ioredis TCP; tolerates dev Redis cold-starts
SESSION_REDIS_COMMAND_TIMEOUT_MS=2000 # session client; tolerates dev Redis cold-starts
UPSTASH_MAX_RETRIES=2                 # one extra retry helps Upstash sandbox cold-start flakiness
# MONGO_SERVER_SELECTION_TIMEOUT_MS — not consumed by db.js (removed in 29c2e43); driver default (30 s) applies
```

**Production profile** (tighter bounds — fail fast under real traffic)

```env
# COD / cache resilience (prod): code defaults, designed for production throughput
REDIS_OP_TIMEOUT_MS=1500              # code default; max per-op Redis latency budget
COD_REDIS_OP_TIMEOUT_MS=800           # code default; tighter COD override — fail fast at checkout
CACHE_REDIS_COMMAND_TIMEOUT_MS=1500   # code default; matches REDIS_OP_TIMEOUT_MS budget
SESSION_REDIS_COMMAND_TIMEOUT_MS=1000 # code default; stalled session store can never block a response
UPSTASH_MAX_RETRIES=1                 # code default; one retry, then fail — avoids Upstash outage hangs
# MONGO_SERVER_SELECTION_TIMEOUT_MS — not consumed by db.js (removed in 29c2e43); driver default (30 s) applies
```

**Why dev uses looser bounds:** development traffic is negligible, Redis connections are local and
fast, and Upstash sandbox can be slow to warm. Production uses the code defaults — each value was
chosen as the maximum latency budget at which a stalled cache dependency starts visibly degrading
checkout or session response times under real load.

---

## Payment & order flow (detailed)

This section explains the exact runtime flow implemented by the backend payment controller.

### Checkout (Stripe) flow — `createCheckoutSession`

1. **Request validation**
   - Backend receives `products`, `shippingAddress`, `billingAddress`, and optional `phoneNumbers`.
   - Validates addresses and that the cart is non-empty.
   - Rejects carts that span more than one store with HTTP 400. Multi-store checkout is
     intentionally blocked; customers must complete a separate checkout per store.

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

**Stripe checkout flow (createCheckoutSession)**

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Express API
    participant R as Redis
    participant S as Stripe
    participant DB as MongoDB
    C->>+API: POST /checkout/initiate
    API->>+R: cacheAcquireLock(variantId)
    R-->>-API: lock acquired
    API->>+S: createPaymentIntent(amount)
    S-->>-API: client_secret
    API-->>-C: client_secret + draft order
    C->>S: stripe.confirmPayment (Elements)
    S->>+API: POST /webhooks/stripe (HMAC signed)
    API->>API: constructEvent verify
    API->>+DB: create Order + LedgerEntry (transaction)
    DB-->>-API: committed
    API->>R: cacheReleaseLock
    API-->>-S: 200 OK
```

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

- **`account.updated`**
  - Idempotent: uses Redis key `processed_account_updated:{event.id}` with a **7-day TTL** to handle
    Stripe's retry window. Duplicate deliveries within the TTL window are discarded without
    re-processing.

- **`refund.updated`**
  - Idempotent: uses Redis key `processed_refund_updated_event:{event.id}`. Duplicate deliveries of
    the same event are discarded without re-processing.
  - On `succeeded`:
    - Marks the refund record `completed`.
    - Updates order status.
    - Posts refund ledger entries via `recordRefundLedger` (reverses sale/commission rows for the
      refunded items).
    - Restores reserved stock for full refunds.
  - On `failed`: marks the refund record `failed` and updates order status; no ledger entries are
    written.

- **`charge.dispute.created`**
  - Fired by Stripe when a cardholder raises a chargeback on a charge.
  - Looks up the order by `charge.metadata.orderId` (or by payment intent).
  - If a platform dispute already exists for that order, attaches `chargeback.provider: 'stripe'`
    and stores the Stripe dispute ID and amount.
  - If no platform dispute exists, creates one automatically with `issueType: 'other'` and status
    `open`, linking it to the order, customer, seller, and store.

- **`charge.dispute.closed`**
  - Fired by Stripe when a chargeback is resolved.
  - Maps Stripe outcome to platform resolution: `won` → `no_refund`; `lost` → `refund_full`.
  - Updates `dispute.chargeback.outcome` and transitions dispute status to `resolved` (if not
    already closed by admin).

> All webhook processing logs heavily and attempts safe recovery. Non-critical failures (e.g.,
> email) are logged but don't crash the handler.

---

**Stripe webhook processing with idempotency**

```mermaid
sequenceDiagram
    participant S as Stripe
    participant API as Express API
    participant DB as MongoDB
    participant L as Ledger
    S->>+API: POST /api/payments/webhook
    API->>API: stripe.webhooks.constructEvent
    API->>+DB: check idempotency key
    DB-->>-API: not seen
    API->>+DB: update Order status
    DB-->>-API: saved
    API->>+L: postLedgerEntry(type, amount)
    L-->>-API: LedgerEntry._id
    API->>DB: store idempotency key
    API-->>-S: 200 OK
```

**Generic webhook idempotency gate**

```mermaid
sequenceDiagram
    participant Stripe as Stripe
    participant WH as Webhook handler
    participant Cache as Redis NX
    Stripe->>WH: event (event.id)
    WH->>Cache: cacheSetNXJSON(key=event.id, ttl)
    alt key already set
        Cache-->>WH: duplicate -> res.json({received:true})
    else acquired
        WH->>WH: process (mutate state, post ledger)
    end
```

### COD flow (Cash-on-Delivery) — `createCODOrder` & `codCheckoutSuccess`

- `createCODOrder` follows almost the same path as Stripe checkout: validate payload, compute
  totals, create an `Order` (status `cod_order_placed`), reserve stock (variant-aware), and return
  order id and totals to the client.
- `codCheckoutSuccess` accepts `orderId`, verifies user ownership, clears cart and safety keys,
  sends confirmation email (if not sent), and returns order summary to the client.

**Cache-failure resilience**

COD order creation is hardened so a Redis problem can never lose or silently block an order:

- Stock-reservation writes to Redis are tolerant: a timeout, rate-limit, or disconnect is logged as
  a warning rather than thrown, so the order still commits and returns `201`. A COD order placed
  during a complete cache outage succeeds normally.
- Cache operations are bounded by `REDIS_OP_TIMEOUT_MS` and Upstash retries are capped by
  `UPSTASH_MAX_RETRIES`; once one reservation write fails, the remaining items skip Redis so a
  single outage costs at most one timeout for the whole order.
- The session Redis client uses a command timeout and disables the offline queue
  (`SESSION_REDIS_COMMAND_TIMEOUT_MS`), so a stalled session store can never block the response
  flush — previously this could leave a committed order with no HTTP response.
- If no reservation keys exist at restore time (e.g. they were never written during an outage),
  stock is restored directly from the order document, guarded by the order's `restockApplied` flag.
  The `reconcile:cod-reservations` script audits (and optionally backfills) affected orders.
- Frontend: if the request hits the axios timeout, checkout shows a calm "your order is taking
  longer than expected — please check your orders page" message instead of a failure toast, and the
  cart is **not** cleared, so the customer never loses items for an order that may have been placed.
- The COD route, the Stripe `create-checkout-session` route, and `GET /api/orders/user` are wrapped
  in the `skipSessionPersist` middleware (`middleware/sessionSkip.middleware.js`). With
  `express-session` `rolling: true`, `connect-redis` defers `res.end` until the per-request session
  touch/save returns; a slow session Redis could therefore hang the response of an already-committed
  order. These JWT-authenticated routes don't mutate the session, so skipping the rolling save/touch
  is safe — the sliding expiry just refreshes on the next mutating request.
- Per-flow override: COD reservation cache ops honour `COD_REDIS_OP_TIMEOUT_MS` (falls back to
  `REDIS_OP_TIMEOUT_MS`). The TCP/ioredis cache driver is hardened with
  `CACHE_REDIS_COMMAND_TIMEOUT_MS` + a disabled offline queue so a stalled TCP Redis fails fast
  instead of buffering commands.
- MongoDB driver defaults were restored (custom `serverSelectionTimeoutMS` / `socketTimeoutMS` /
  pool options were removed) after they were traced as the root cause of a recurring
  `MongoNetworkTimeoutError` crash; an `uncaughtException` guard now survives a transient Mongo pool
  timeout instead of exiting the process. Connection timeouts remain tunable via
  `MONGO_SERVER_SELECTION_TIMEOUT_MS` / `MONGO_CONNECT_TIMEOUT_MS` when explicitly set.

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

**Line-item partial refunds (paid + COD)**

- Refunds support selecting specific line items and quantities, not just whole orders. The customer
  picks items on the order-detail modal; the return request carries `items[]` of
  `{ productId, variantKey, quantity }` and `order.returnRequest.requestedItems[]` persists the
  selection. A line is keyed by `productId` + `buildVariantKey(variantAttributes)`.
- Approval reuses the same approve endpoints. `computeRefundPlan` prices each selected line from the
  amount actually charged (`price × (1 − deal)`) and attributes each seller **exactly** their items
  — never a proportional split. Returnable quantity per line is `ordered − already-refunded`, where
  already-refunded comes from `getRefundedQuantities` (sum of `refundRecords[].items[]` on completed
  records); `validateRefundItemsInput` re-checks it server-side.
- **Admin-initiated:** an admin can start a partial refund without a customer return request
  (`POST /api/admin/orders/:id/refund/approve` with `items[]`). **Seller scoping:** a seller may
  refund only the items they sold; a multi-seller COD return must be approved by an admin (one COD
  payout goes to the customer). Sellers cannot initiate refunds — only approve customer requests.
- **Shipping:** not refunded on partials by default. It is included only when the return reason is
  eligible (`refundShippingEligible`, e.g. defective/wrong item) or an admin passes
  `refundShipping=true`; **never for COD partials**.
- **Stripe vs COD:** Stripe issues a partial Refund for the computed amount; COD pays the customer
  via the payout workflow and debits each seller through the ledger (`cod_refund_reimbursement`).
  Concurrency/idempotency: per-order refund lock + unique `idempotencyKey` + remaining-quantity
  recheck. A partial covering all remaining items flips the order to `refunded`; otherwise the UI
  shows **Partially Refunded** — a status derived from items coverage (the stored status stays
  `refunded`/`pending_refund`; `refundRecords[].items[]` distinguishes partial from full).

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

1. Re-send original payload to `POST /api/payments/chimoney/webhook`.
2. Pass `x-chimoney-signature` header with active HMAC-SHA256 signature computed over the raw
   request body using `CHIMONEY_WEBHOOK_SECRET`.
3. Confirm response has `received: true`.
4. Verify matching via idempotency key (fallback provider refund id), and confirm order transition.

**Refund processing pipeline**

```mermaid
graph TD
    A["Refund request"] --> B["refundPlan.service\ncalculate refundable amount"]
    B --> C["acquire refundLock\nlib/refundLock.js"]
    C --> D{"Payment method?"}
    D -->|"Stripe"| E["Stripe Refund API"]
    D -->|"COD"| F["codRefundWorkflow"]
    E --> G["post LedgerEntry\ntype: refund"]
    F --> G
    G --> H["update Order status"]
    H --> I["release inventory"]
    H --> J["release refundLock"]
```

**Partial refund: webhook dedup, order lock, compensating entries**

```mermaid
sequenceDiagram
    participant Stripe as Stripe
    participant WH as payment.controller webhook
    participant Cache as Redis NX
    participant Lock as withRefundLock(orderId)
    participant Ledger as recordRefundLedger
    Stripe->>WH: charge.refunded (event.id)
    WH->>Cache: setNX processed_refund_event:{id} (24h)
    alt duplicate
        Cache-->>WH: exists -> ack & ignore
    else first delivery
        WH->>Lock: acquire order refund lock
        Lock->>WH: update refundRecord status
        WH->>Ledger: recordRefundLedger(amount, ratio)
        Ledger->>Ledger: reserve_used (if any) ->refund (−)<br/>-> commission_reversal (+) -> tax/shipping refund
        Note over WH: refundedAmountCents < total<br/>=> pending_refund (partiallyRefunded)
    end
```

**Full refund also restores reserved stock**

```mermaid
sequenceDiagram
    participant Stripe as Stripe
    participant WH as webhook
    participant Ledger as recordRefundLedger
    participant Stock as stock service
    Stripe->>WH: charge.refunded (full)
    WH->>Ledger: full reversal entries
    WH->>WH: refundedAmountCents >= total -> status refunded
    WH->>Stock: restoreStockReservationByOrderId(order)
```

**Refund record status (order.refundRecords[])**

```mermaid
stateDiagram-v2
    [*] --> pending
    pending --> in_progress
    in_progress --> completed: charge.refunded / refund.updated succeeded
    in_progress --> failed: refund.updated failed
    pending --> failed
    completed --> [*]
    failed --> [*]
```

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

When creating the webhook endpoint in Chimoney dashboard, use your backend public URL:

- Endpoint URL: `https://<your-backend-domain>/api/payments/chimoney/webhook`

The backend exposes a single authenticated route:

- `POST /api/payments/chimoney/webhook`

**Security:** Every incoming request is verified with HMAC-SHA256 over `req.rawBody` using
`CHIMONEY_WEBHOOK_SECRET`. The secret is **required** — the server throws at boot time if it is
missing in production (`NODE_ENV=production`). The URL-path secret variant
(`/webhook/:webhookSecret`) has been removed; all authentication is now signature-based. Redis SETNX
replay protection is applied per event (TTL: 24 h) so duplicated deliveries are safely ignored
without re-processing.

Subscribe to these events (the ones currently processed by backend):

- `payout.bank.initiated`
- `payout.bank.completed`
- `payout.bank.failed`

If extra events are sent, they are accepted but ignored as `unsupported_event_type`.

For local testing via tunnel (ngrok/Cloudflare Tunnel), set endpoint URL to your temporary HTTPS URL
plus `/api/payments/chimoney/webhook`, then replay a sample event and confirm `received: true`.

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

#### 3) Dispute SLA escalation cron

- **File:** `backend/src/jobs/disputeSlaEscalation.job.js`
- **Scheduler env vars**
  - `DISPUTE_SLA_ESCALATION_ENABLED` — set to `"false"` to disable (default: `"true"`)
  - `DISPUTE_SLA_ESCALATION_CRON` — cron expression (default: `"0 8 * * *"` — 08:00 UTC daily)
- **Purpose**
  - Finds all non-terminal disputes (`status` not in `resolved`, `closed`, `withdrawn`) where
    `slaDueAt <= now`.
  - Appends a system admin message to each flagged dispute thread.
  - Sends `sendDisputeSlaEscalationEmail` to the admin recipient list once per dispute (guarded by
    `slaEscalationEmailedAt` to prevent duplicate emails on re-runs).
- **SLA window:** 72 hours from dispute creation (`slaDueAt = createdAt + 72h`).
- **How to test**
  1. Create a dispute and backdate `slaDueAt` to a past timestamp.
  2. Run the job manually (or set a frequent `DISPUTE_SLA_ESCALATION_CRON` in dev).
  3. Verify a system message is appended and the escalation email is queued.
  4. Confirm re-runs do not send a second email (check `slaEscalationEmailedAt` is set).

#### 4) Liability reconciliation cron

- **File:** `backend/src/jobs/liabilityReconciliation.job.js`
- **Scheduler env vars**
  - `FEATURE_LIABILITY_RECONCILIATION` — set to `"true"` to enable (default `"false"`)
  - `LIABILITY_RECONCILIATION_CRON` — cron expression (default: `"30 3 1 * *"` — 03:30 UTC on the
    1st of each month)
  - `LIABILITY_RECONCILIATION_TIMEZONE` — default `"UTC"`
  - `LIABILITY_RECONCILIATION_RUN_ON_BOOT` — set `"true"` only on single-instance deploys
  - `LIABILITY_RECONCILIATION_PERIOD` — override the reconciliation period (e.g. `"2026-05"`);
    defaults to the previous calendar month
- **Purpose**
  - Aggregates `shipping_fee`, `shipping_refund`, `shipping_label_cost`, `shipping_revenue`,
    `shipping_revenue_refund`, `tax`, `tax_refund`, and `tax_remittance` entries for the closed
    month.
  - Produces one `ReconciliationSnapshot` per `{period, currency}` (upserted on re-run), including
    `shippingRevenueCents`, `shippingRevenueRefundCents`, and `sellerRetainedShippingCents`
    (`shippingRevenueCents + shippingRevenueRefundCents`) — reported for visibility only and never
    folded into `shippingNetLiabilityCents` or the negative-residual `status` check.
  - Sets `status: 'review'` and logs a warning when any liability net is negative (outflows exceeded
    collections), indicating a shortfall that requires operator investigation.
- **How to test**
  1. Set `FEATURE_LIABILITY_RECONCILIATION=true` and `LIABILITY_RECONCILIATION_RUN_ON_BOOT=true`.
  2. Restart the backend (or wait for the cron trigger).
  3. Query `ReconciliationSnapshot` for the previous month — verify `shippingNetLiabilityCents` and
     `taxNetLiabilityCents` match your ledger aggregates.
  4. Force a negative residual (e.g. set `LIABILITY_RECONCILIATION_PERIOD` to a month where
     `shipping_label_cost` exceeds `shipping_fee`) and confirm `status: 'review'` is set.

### Carrier shipping webhook

`POST /api/webhooks/shipping/:provider` is the generic carrier webhook hub. Each supported carrier
posts status updates here. Security:

- HMAC-SHA256 signature verification using `ARAMEX_WEBHOOK_SECRET` (or the provider-specific secret
  variable).
- Idempotent deduplication by `(trackingNumber, carrierStatusRaw, eventTimestamp)` with a 24-hour
  Redis TTL — duplicate deliveries are silently ignored.
- Automatically transitions `Order.status` to `delivered` when the carrier confirms delivery.

To rotate the Aramex webhook secret, update `ARAMEX_WEBHOOK_SECRET` and re-register the endpoint URL
in the Aramex developer console at `https://<your-backend-domain>/api/webhooks/shipping/aramex`.

### Key rotation & webhook replay procedures

For full step-by-step rotation procedures including dual-secret Stripe rotation, see
[docs/security/webhook-rotation.md](docs/security/webhook-rotation.md).

#### Rotate Stripe webhook secret safely

Stripe supports up to 5 active secrets per endpoint, enabling zero-downtime dual-secret rotation.

1. In the Stripe Dashboard, go to **Developers → Webhooks → your endpoint** and click **"Add
   secret"**.
2. Add the new secret alongside the old one in Railway:
   `STRIPE_WEBHOOK_SECRETS=whsec_new,whsec_old`.
3. Deploy and confirm both secrets verify incoming events (check logs).
4. Remove the old secret from the list and delete it from the Stripe Dashboard.
5. Update `STRIPE_WEBHOOK_SECRET` to the new value.

#### Rotate Chimoney API key safely

1. Create new API key in Chimoney console (same environment).
2. Update secret manager (`CHIMONEY_API_KEY`).
3. Deploy and validate outbound provider calls.
4. Reconcile one in-progress payout to ensure status lookup still works.
5. Revoke old key after verification window.

#### Rotate Chimoney webhook secret safely

1. Generate new webhook secret in Chimoney.
2. Update `CHIMONEY_WEBHOOK_SECRET`.
3. Replay a known webhook payload against `/api/payments/chimoney/webhook`.
4. Confirm request is accepted and matched to refund record.
5. Remove old webhook secret in provider console.

#### Replay webhook (admin/on-call detailed)

1. Capture original payload + event metadata from provider dashboard.
2. `POST /api/payments/chimoney/webhook` with the original raw payload body and the
   `x-chimoney-signature` HMAC-SHA256 header computed over the raw body bytes using
   `CHIMONEY_WEBHOOK_SECRET` — do not re-serialize the JSON.
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

- `payoutHold` is a seller-level settlement safety switch. When `true`, future scheduled settlement
  payouts are paused for that seller, but the seller storefront and normal order operations remain
  active.
- Settlement scheduler excludes sellers with `payoutHold=true` from scheduled payouts until the
  related refund is resolved.

**Admin operations: set / unset payout hold**

- API endpoint: `POST /api/admin/sellers/:id/payout-hold` (admin-only).
- Request body:

  ```json
  {
    "enabled": true,
    "reason": "Risk review pending",
    "until": "2026-06-01T00:00:00.000Z"
  }
  ```

  Notes:
  - `enabled` (boolean) is required.
  - `reason` is required when `enabled=true`.
  - `until` is optional and must be a valid date when provided.

- Example enable response (`200`): returns updated `Seller` document with hold state + history.

  ```json
  {
    "_id": "664f0cc0f9e53a001f8ab123",
    "payoutHold": true,
    "payoutHoldReason": "Risk review pending",
    "payoutHoldSetAt": "2026-05-03T12:00:00.000Z",
    "payoutHoldHistory": [
      {
        "setAt": "2026-05-03T12:00:00.000Z",
        "enabled": true,
        "reason": "Risk review pending",
        "until": "2026-06-01T00:00:00.000Z",
        "adminId": "664f0aa1f9e53a001f8ab111"
      }
    ]
  }
  ```

- Example unset request:

  ```json
  {
    "enabled": false,
    "reason": "Refund issue resolved"
  }
  ```

- Example unset response (`200`):

  ```json
  {
    "_id": "664f0cc0f9e53a001f8ab123",
    "payoutHold": false,
    "payoutHoldReason": null,
    "payoutHoldSetAt": null,
    "payoutHoldHistory": [
      {
        "setAt": "2026-05-01T09:00:00.000Z",
        "enabled": true,
        "reason": "cod_refund_pending",
        "adminId": "664f0aa1f9e53a001f8ab111"
      },
      {
        "setAt": "2026-05-03T12:10:00.000Z",
        "clearedAt": "2026-05-03T12:10:00.000Z",
        "enabled": false,
        "reason": "Refund issue resolved",
        "adminId": "664f0aa1f9e53a001f8ab111"
      }
    ]
  }
  ```

- Admin UI path: **Admin → Sellers → Seller details/actions → Payout hold**.
  - Enable hold: set `enabled=true` and provide reason (optional date window via `until`).
  - Clear hold: set `enabled=false`; current hold fields are reset and an audit row is appended.

**Scheduler behavior with held sellers**

- In settlement scheduling, the system computes positive seller payouts, then filters out sellers
  where `payoutHold=true`.
- Held sellers are ignored only for payout batch inclusion; their eligible ledger entries remain
  pending and can be picked up in later runs after hold removal.
- The hold is re-evaluated each scheduler run, so once unset, the seller is eligible again
  automatically.

**`payoutHoldHistory` structure (Seller model)**

Each history item records one hold toggle event:

- `setAt` (`Date`, default now): when the event was recorded.
- `clearedAt` (`Date|null`): populated for clear/unset events.
- `enabled` (`Boolean`, required): target state from this action.
- `until` (`Date|null`): optional operator-provided hold-until marker.
- `reason` (`String|null`): hold rationale.
- `adminId` (`ObjectId|null`): admin actor.

Validation errors (API):

- `400` invalid seller id.
- `400` `enabled` not boolean.
- `400` missing reason while enabling hold.
- `400` invalid `until` date.
- `404` seller not found.

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
  - `PLATFORM_SELLER_ID`: seller id of the platform liability ledger account (tax flows when
    `FEATURE_TAX_LIABILITY_SELLER` is off).
  - `SHIPPING_SELLER_ID`: seller id of the shipping liability ledger account (carrier-mode shipping
    flows).
  - `TAX_SELLER_ID`: seller id of the Tax-Liability ledger account (tax flows when
    `FEATURE_TAX_LIABILITY_SELLER=true`). **Operational notes**

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

#### System relationship diagram (simplified)

```mermaid
erDiagram
    SELLER ||--o{ LEDGER_ENTRY : has
    SELLER ||--o{ SETTLEMENT_PAYOUT : has
    SETTLEMENT_BATCH ||--o{ SETTLEMENT_PAYOUT : contains
    SETTLEMENT_BATCH ||--o{ LEDGER_ENTRY : includes
```

#### Settlement and payout flow

```mermaid
flowchart TD
  OrderDelivered["Order delivered"] -->|trigger| RecordLedger
  RecordLedger["recordDeliveryLedger(order)"] --> LedgerEntries{"Ledger: sale + commission entries"}
  LedgerEntries -->|pending| AwaitSettlement
  AwaitSettlement["Wait until period end"] -->|cron job| CreateBatch
  CreateBatch["scheduleNextBatch"] -->|creates| SettlementBatch
  SettlementBatch -->|creates| SettlementPayout
  SettlementPayout -->|status=scheduled| ExecutePayout
  ExecutePayout["executeSettlementBatch"] -->|stripe.transfer.create| Stripe
  Stripe -->|success| MarkPaid
  ExecutePayout -->|failure| MarkFailed
  MarkPaid["SettlementPayout: status=paid\nLedgerEntries: status=paid"]
  MarkFailed["SettlementPayout: status=failed/manual_required"]
```

#### Ledger lifecycle narrative (delivery -> payout -> reserve release)

1. Order delivery finalization calls `recordDeliveryLedger(order)` and posts immutable
   sale/commission ledger rows.
2. Refunds (partial/full) append `refund` + `commission_reversal` rows, preserving immutable
   history.
3. Scheduler closes eligible period windows and computes seller net from pending rows.
4. Batch scheduling creates `SettlementBatch` and positive-net `SettlementPayout` rows.
5. Reserve policy withholds holdback value (`reserveWithheldAmountCents`) before payout execution.
6. Stripe payout execution marks payouts `paid` or transitions to `failed` / `manual_required`.
7. Reserve release job creates `reserve_release` rows when hold windows expire.
8. Released reserves flow back into future settlement eligibility; reversals/retries keep correction
   audit trails.

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

**Settlement amount semantics**

- `grossEligibleAmountCents`: total eligible ledger value before reserve withholding.
- `reserveWithheldAmountCents`: reserve/holdback value withheld for risk policy.
- `netPayoutAmountCents`: payout-intended net after reserve withholding.
- Net equation: `netPayoutAmountCents = grossEligibleAmountCents - reserveWithheldAmountCents`.
- `inPeriodAmountCents`: portion of `grossEligibleAmountCents` from ledger entries created within
  the current settlement period.
- `carryForwardAmountCents`: portion from pending entries created in prior periods that were carried
  into this payout cycle.

When `carryForwardAmountCents > 0`, the seller financials panel displays an inline indicator —
"Includes $X.XX from prior periods" — alongside the payout total so sellers can distinguish
current-period earnings from accumulated carry-forward amounts.

#### Seller ledger exports (CSV + PDF) and filter

The seller financials ledger supports:

- Filter controls in dashboard ledger tab:
  - From date
  - To date
  - Entry type
  - Payout status
- Export options as separate actions:
  - **Download CSV** (`GET /api/seller/ledger/export`)
  - **Download PDF** (`GET /api/seller/ledger/export-pdf`)

##### Export data shape

Both export formats use the same backend filter inputs (`from`, `to`, `type`, `payoutStatus`,
`includeOperatorRows`) so exported data mirrors on-screen filtering.

Exports include order-level fields where available from ledger references/metadata:

- `orderId`
- `gross` / `grossCents`
- `commission` / `commissionCents`
- `fees` / `feesCents`
- `reserve` / `reserveCents`
- `net` / `netCents`
- `orderStatus`

##### Amount formatting fix

To avoid confusion, human-readable monetary values are now formatted from cents into currency units
(e.g. `10000` cents => `$100.00`) in:

- PDF row summaries
- CSV readable amount columns

Raw cents columns are still included in CSV for accounting/system integrations.

#### Double-entry ledger and accounting

The platform uses a **signed double-entry ledger** where `amountCents > 0` is a credit (inflow) and
`amountCents < 0` is a debit (outflow). Every financial event produces at least two rows — a
merchant-scoped row and a platform/operator-scoped mirror — so both sides of each transaction are
always on record.

**Entry type reference**

| Type                       | Settlement class | Who it touches                           | Sign convention (merchant side)                                                 |
| -------------------------- | ---------------- | ---------------------------------------- | ------------------------------------------------------------------------------- |
| `sale`                     | payout-eligible  | Seller                                   | Positive (revenue)                                                              |
| `refund`                   | payout-eligible  | Seller                                   | Negative (revenue reversal)                                                     |
| `commission`               | payout-eligible  | Seller                                   | Negative (platform fee deducted)                                                |
| `commission_reversal`      | payout-eligible  | Seller                                   | Positive (commission returned)                                                  |
| `processing_fee`           | payout-eligible  | Seller                                   | Negative (Stripe fee share)                                                     |
| `processor_fee`            | payout-eligible  | Platform                                 | Negative (full Stripe cost)                                                     |
| `processor_fee_recovery`   | payout-eligible  | Seller + Platform                        | Delta vs estimate (positive or negative)                                        |
| `platform_fee`             | payout-eligible  | Platform                                 | Positive (mirrors commission credit)                                            |
| `cod_refund_disbursement`  | payout-eligible  | Platform                                 | Negative (platform disburses refund)                                            |
| `cod_refund_reimbursement` | payout-eligible  | Seller                                   | Negative (seller reimburses platform for merchandise + tax + shipping refunded) |
| `tax`                      | pass-through     | Seller debit / Platform credit           | Negative on seller side                                                         |
| `tax_refund`               | pass-through     | Seller credit / Platform debit           | Positive on seller side                                                         |
| `tax_remittance`           | pass-through     | Platform                                 | Negative (platform pays tax authority)                                          |
| `shipping_fee`             | pass-through     | Seller debit / Shipping Liability credit | Negative on seller side                                                         |
| `shipping_refund`          | pass-through     | Seller credit / Shipping Liability debit | Positive on seller side                                                         |
| `shipping_label_cost`      | pass-through     | Shipping Liability                       | Negative (carrier label cost)                                                   |
| `shipping_revenue`         | payout-eligible  | Seller                                   | Positive (flat/tiered shipping kept as seller revenue)                          |
| `shipping_revenue_refund`  | payout-eligible  | Seller                                   | Negative (flat/tiered shipping revenue reversed on refund)                      |
| `reserve`                  | reserve          | Seller                                   | Negative (withheld at settlement)                                               |
| `reserve_release`          | reserve          | Seller                                   | Positive (hold expires)                                                         |
| `reserve_used`             | reserve          | Seller                                   | Positive (reserve applied to debt)                                              |

**Pass-through entries report `payoutStatus: 'collected'`**

The six **pass-through** types above (`tax`, `tax_refund`, `tax_remittance`, `shipping_fee`,
`shipping_refund`, `shipping_label_cost`, i.e. `PASS_THROUGH_TYPES` in
`settlementClassification.js`) are collected on the platform's behalf and are never owed to the
seller as a payout, so the pending → scheduled → paid lifecycle described above does not apply to
them. `normalizeLedgerEntry` — used by the seller and admin ledger list endpoints and the seller's
CSV/PDF exports — overrides `payoutStatus` to `'collected'` for these entries regardless of the
value stored on the row. `SellerFinancialsPanel` renders this as a dedicated **"Collected"** badge
(`seller.financials.status.collected`) instead of "Pending payout" / "Scheduled" / "Paid".

**Merchant / operator mirroring pattern**

Each delivery event calls `buildMerchantLedgerDimensions` and `buildOperatorLedgerDimensions` to
produce both scopes. The `accountingScope` field (`merchant` or `platform`) and `entrySide` field
(`merchant` or `operator`) distinguish the two rows. Seller dashboards default to merchant-scoped
rows; `includeOperatorRows=true` is available for admin audit queries.

**Shipping Liability synthetic seller**

`ensureAdminStore.js` creates a synthetic seller with slug `shipping-liability` on first startup and
exports its `_id` as `process.env.SHIPPING_SELLER_ID`. This account acts as the platform's shipping
liability pool:

- **Credits** (`shipping_fee`, positive): collected from customers on every delivered order.
- **Debits** (`shipping_refund`, negative): returned on shipping refunds.
- **Debits** (`shipping_label_cost`, negative): carrier label costs debited when a Shippo label is
  purchased (gated by `FEATURE_SHIPPING_LABEL_LEDGER`). This closes the accounting loop — carrier
  spend is now tracked against the shipping fees collected.

**Shipping mode and seller-retained shipping revenue**

Each order snapshots the **effective, region-specific** shipping mode at checkout time as
`order.shippingMode` (`'flat' | 'tiered' | 'carrier'`, defaulting to `'flat'` for orders placed
before this field existed). `resolveEffectiveShippingMode(shipping, destination)` resolves a
per-region `shipping.regions[].mode` override for the customer's destination, falling back to the
store-level `shipping.mode` when the matched region has none — so two orders from the same store can
carry different `shippingMode` values depending on the destination. On delivery,
`resolveShippingMode(order)` reads this per-order snapshot to decide how the collected shipping
amount is posted:

- **`carrier`** (or `FEATURE_SELLER_SHIPPING_PAYOUT=false`): unchanged pass-through behavior — the
  seller is debited `shipping_fee` and the Shipping Liability pool (`SHIPPING_SELLER_ID`) is
  credited the same amount, later offset by `shipping_label_cost` when a label is purchased.
- **`flat` / `tiered`** (with `FEATURE_SELLER_SHIPPING_PAYOUT=true`): the seller fulfils shipping
  themselves, so the collected amount is their revenue. A single payout-eligible `shipping_revenue`
  credit is posted to the seller — there is no Shipping Liability mirror, since there is no platform
  payable to offset. On refund, `shipping_revenue_refund` (negative) reverses it symmetrically.

Both entry types carry `metadata: { shippingMode }` for audit/debugging. The flag is **off by
default** and **forward-only** — it changes how new deliveries are posted while it is on; existing
`shipping_fee`/`shipping_refund` rows from before the cutover are left untouched (no backfill).
`marketplaceMetrics.sellerRetainedShippingCents` (`shipping_revenue + shipping_revenue_refund`) and
the seller financial summary's `sellerRetainedShippingCents` report this figure separately from the
carrier `shippingLiabilityCents` — the two are never blended.

**Tax Liability synthetic seller**

`ensureAdminStore.js` also creates a synthetic seller with slug `tax-liability` on first startup and
exports its `_id` as `process.env.TAX_SELLER_ID`. When `FEATURE_TAX_LIABILITY_SELLER=true`,
`resolveTaxLiabilityLedgerSellerId()` routes the operator-side mirror of every `tax`, `tax_refund`,
and `tax_remittance` entry to this account instead of `PLATFORM_SELLER_ID` — separating the
marketplace-wide tax clearing account from the platform store's own books, mirroring how
`SHIPPING_SELLER_ID` isolates the carrier pass-through pool. The flag is **off by default** and
**forward-only**: historical tax rows already posted against `PLATFORM_SELLER_ID` are left
untouched; only entries posted while the flag is on use `TAX_SELLER_ID`. See `docs/tax-book.md` for
the cutover and reconciliation impact.

When `FEATURE_TAX_LIABILITY_SELLER=false` (default), `PLATFORM_SELLER_ID` holds the tax liability
pool.

**Shippo label cost tracking**

When a seller purchases a carrier label via `POST /seller/orders/:id/shipments` (source: `api`), the
Shippo adapter extracts the label cost from the transaction response and stores it as
`labelCostCents` / `labelCostCurrency` / `labelCostSource` on the `Shipment` document. The
`recordShippingLabelExpense` function then posts a `shipping_label_cost` debit:

- Idempotency key: `shipping_label_cost:<shipmentId>` — safe to retry.
- Skipped automatically when `labelCostSource === 'sandbox'` (Shippo sandbox labels are $0 and must
  not pollute the production ledger).
- Skipped when `labelCostCents <= 0`.
- Enable with `FEATURE_SHIPPING_LABEL_LEDGER=true` (default `false`).

**Tax remittance tracking**

Admins record tax payments made to a tax authority from **Admin → Marketplace Financials → Tax
remittances**, a panel that only renders when `FEATURE_TAX_REMITTANCE=true`. Its **Record tax
remittance** button opens a modal with **Period** (`YYYY-MM`), **Jurisdiction**, **Amount**,
**Currency**, **Date filed**, and **Notes** fields; submitting calls
`POST /api/admin/platform-financials/tax-remittances`. The service creates a `TaxRemittance`
document and calls `recordTaxRemittance`, which posts a `tax_remittance` debit against
`resolveTaxLiabilityLedgerSellerId(PLATFORM_SELLER_ID)` — the Tax-Liability seller (`TAX_SELLER_ID`)
when `FEATURE_TAX_LIABILITY_SELLER=true`, otherwise `PLATFORM_SELLER_ID`. The document starts as
`'draft'`, the ledger entry is posted, and then it is promoted to `'filed'`. If the ledger call
fails the document stays as `'draft'` for operator review rather than disappearing. On success the
modal closes, a toast confirms the result, and the remittances table (Period / Jurisdiction / Amount
/ Status / Filed at) and liability totals refresh.

**Seller financials**

`sellerFinancialSummary.service.js` aggregates the seller's ledger entries using the same signed-sum
convention as the platform side: merchandise refunds (`type: 'refund'`, negative) are summed with
sales (`merchandiseSubtotalCents = sale + refund`), and tax/shipping refunds are _subtracted_ from
their collected totals (`passThroughTaxCollectedCents = |tax| - |tax_refund|`, and similarly for
shipping). The "Customer paid total" card therefore decreases correctly after a refund, aligning the
seller dashboard with the admin platform liability view. Seller-retained shipping revenue
(`shipping_revenue` / `shipping_revenue_refund`, `sellerRetainedShippingCents`) is reported as its
own field, separate from the carrier shipping liability (`passThroughShippingCollectedCents`) — see
"Shipping mode and seller-retained shipping revenue" above.

**COD refund accounting**

When a COD order is refunded, the platform disburses the full refund (merchandise + tax + shipping)
to the customer via Chimoney. `cod_refund_reimbursement` claws back
`refundAmountCents + taxRefundAmountCents + shippingRefundAmountCents` — the full reimbursed amount
— from the seller's merchant-side balance. In the same event the seller is credited `tax_refund`
(+`taxRefundAmountCents`) and `shipping_refund` (+`shippingRefundAmountCents`), which exactly offset
the tax/shipping portion of the reimbursement debit, leaving the seller's net merchant-side movement
at `-refundAmountCents` — the merchandise portion only. This matches the operator-side
`cod_refund_disbursement` debit (also `-refundAmountCents`), so the order's refund is fully
accounted for on both sides with no residual liability.

**Batch statuses (`SettlementBatch.status`)**

- `calculated`: reserved for pre-scheduled calculation states.
- `scheduled`: payouts are queued and awaiting execution.
- `paid`: all payouts in the batch are terminally paid.
- `partial`: mixed outcome where at least one payout is paid and one remains unresolved.
- `failed`: unresolved failures/manual-required outcomes need operator intervention.
- `skipped_no_positive_payouts`: scheduling pass found no positive seller net payouts.

**Why batch `retryable` is not the primary status**

`retryable` is a payout-level recovery state, not a batch-level lifecycle summary. A single batch
can contain `paid`, `failed`, `manual_required`, and `retryable` payouts simultaneously. The batch
status remains the primary operational summary (`scheduled`/`paid`/`partial`/`failed`/`skipped...`).
Use `hasRetryablePayouts` as the explicit signal for retry actionability.

**`hasRetryablePayouts` indicator**

- `hasRetryablePayouts=true` means one or more payouts in the batch are currently retryable.
- It unlocks retry-focused actions without changing the primary batch status semantics. **Payout
  statuses (`SettlementPayout.status`)**

- `scheduled`: ready to execute.
- `paid`: transfer completed (`externalTransferId` recorded).
- `failed`: Stripe transfer attempt failed.
- `manual_required`: cannot auto-pay (for example, missing Stripe account); export CSV and process
  manually. **Manual payout approval + CSV export workflow**

1. Execute/sweep payouts and isolate `manual_required` records.
2. Export manual payout CSV for treasury/off-platform transfer operations.
3. Perform payout outside automated rails.
4. Approve manually with transaction reference + notes.
5. Recompute batch status; batches return to `paid` when all unresolved payouts are settled.

re-execution using a regenerated idempotency key.

**Reversal scopes**

- Per-seller reverse: reverse one paid payout.
- Selected bulk reverse: reverse a selected set of payout IDs.
- Full batch reverse: reverse all eligible paid payouts in a batch. -

**Reversal + retry semantics**

- Reversal endpoint transitions a paid payout to `retryable` and persists reversal metadata
  (`reversalId`, `reversedAt`, actor) for auditability.
- Reversal and retry flows regenerate payout idempotency keys (`...:retry:<n>:<timestamp>`) so each
  retry execution has a distinct transfer domain while retaining linkage through
  `previousPayoutId`/`nextPayoutId`.
- Repeating reversal requests is idempotent: no duplicate status transition and no duplicate
  `payout_reversal` ledger rows.

**Retry ledger entry (`payout_execution`) and reconciliation math**

- A successful re-execution of a retryable payout creates a ledger entry type `payout_execution`.
- Reconciliation for a reversed + re-executed payout should satisfy:
  `payout_reversal (-P) + payout_execution (+P) = 0` net delta relative to baseline paid settlement,
  while preserving the full correction history.
- See the dedicated operator guide for fallback payout + CSV procedures:
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
- Period computation example (UTC): with `SETTLEMENT_PERIOD_DAYS=15`, windows are typically
  `1st–15th` and `16th–end-of-month`; `SETTLEMENT_HOLD_DAYS` is then added after `periodEnd` before
  payout eligibility.
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

**Prepaid order: processor fee at payment, full ledger at delivery**

```mermaid
graph LR
    A["order_placed<br/>(no ledger)"] --> B["payment resolved<br/>processor_fee (operator)<br/>+ processing_fee (per seller)"]
    B --> C["dispatched<br/>(no ledger)"]
    C --> D["delivered → recordDeliveryLedger():<br/>sale (+), commission (−)/platform_fee (+),<br/>tax pair, shipping entries"]
    D --> E["settlement batch<br/>reserve withheld"]
    E --> F["Stripe transfer<br/>payout paid"]
```

**COD order: commission only at delivery; disbursement + reimbursement on refund**

```mermaid
graph LR
    A["cod_order_placed<br/>(no ledger, no Stripe)"] --> B["dispatched"]
    B --> C["delivered → recordDeliveryLedger():<br/>commission (−)/platform_fee (+)<br/>NO sale entry"]
    C --> D{"refund?"}
    D -->|yes| E["cod_refund_disbursement (−) platform→customer<br/>cod_refund_reimbursement (−) seller→platform"]
    D -->|no| F["settlement batch → payout"]
```

**Shipping ledger branch by shippingMode**

```mermaid
graph TD
    A["delivered: shipping component"] --> B{"shippingMode"}
    B -->|flat / tiered| C["shipping_revenue (+) to seller<br/>seller keeps shipping"]
    B -->|carrier| D["shipping_fee (−) on seller<br/>shipping_fee (+) on Shipping seller"]
    D --> E["shipping_label_cost (−)<br/>posted when label purchased"]
```

**Merchant rows are mirrored by operator rows for two-sided reconciliation**

```mermaid
graph LR
    subgraph Merchant ["accountingScope=merchant / entrySide=merchant"]
      M1["commission (−) seller"]
      M2["tax (−) seller"]
      M3["shipping_fee (−) seller (carrier)"]
    end
    subgraph Operator ["accountingScope=platform / entrySide=operator"]
      O1["platform_fee (+) platform seller"]
      O2["tax (+) tax-liability seller"]
      O3["shipping_fee (+) shipping seller"]
    end
    M1 -. mirrors .-> O1
    M2 -. mirrors .-> O2
    M3 -. mirrors .-> O3
```

**Balances are computed from ledger rows, never stored**

```mermaid
graph LR
    A["LedgerEntry rows<br/>(append-only)"] --> B["filter:<br/>accountingScope = merchant,<br/>payoutStatus ≠ paid,<br/>PAYOUT_ELIGIBLE_TYPES"]
    B --> C["$group sum(amountCents)"]
    C --> D["outstanding seller balance<br/>(derived)"]
```

**Settlement batch build (settlementScheduler.service.js)**

```mermaid
sequenceDiagram
    participant Cron as settlementScheduler job
    participant Lock as Redis lock
    participant Sched as settlementScheduler.service
    participant DB as MongoDB
    Cron->>Lock: acquire settlement:scheduler:lock
    alt not acquired
        Lock-->>Cron: skip cycle
    else acquired
        Cron->>Sched: scheduleNextBatch(period)
        Sched->>DB: find LedgerEntry {payoutStatus: pending,<br/>eligible types, createdAt <= periodEnd}
        Sched->>Sched: group by seller, drop non-positive,<br/>exclude platform/shipping/tax sellers
        Sched->>Sched: buildReserveWithholding per seller
        Sched->>DB: create SettlementBatch (scheduled)
        Sched->>DB: insertMany SettlementPayout (scheduled)
        Sched->>DB: ledger rows -> payoutStatus scheduled + settlementBatchId
        Sched->>DB: reserve rows -> type reserve, payoutStatus reserved
        Cron->>Lock: release
    end
```

**Stripe Connect transfer with preflight, balance check, and idempotency**

```mermaid
sequenceDiagram
    participant Exec as settlementExecution.service
    participant Stripe as Stripe Connect
    participant DB as MongoDB
    Exec->>Exec: check bank linkage + amount > 0
    Exec->>Stripe: accounts.retrieve(connectId)
    alt requirements due / transfers inactive / payouts disabled
        Exec->>DB: payout -> manual_required
    else account ready
        Exec->>Stripe: balance.retrieve()
        alt available < amount
            Exec->>DB: payout -> failed (insufficient balance)
        else funded
            Exec->>Stripe: transfers.create(amount, dest,<br/>idempotencyKey settlement:batch:seller:ccy)
            Note over Exec,Stripe: retry w/ backoff on retryable errors
            Stripe-->>Exec: transfer id
            Exec->>DB: payout -> paid, externalTransferId set
        end
    end
```

**Service interaction map — solid = synchronous write, dashed = async / detection**

```mermaid
graph TD
    O["Order Service"] -->|status-change hook| L["Ledger Service"]
    P["Stripe"] -->|webhook (async)| W["Webhook Handler"]
    W --> L
    L -->|append entries| DB[("LedgerEntry")]
    S["Settlement Scheduler (cron)"] --> DB
    S --> SB[("SettlementBatch / Payout")]
    EX["Settlement Execution"] -->|transfers.create| P
    R["Reconciliation (cron)"] -.->|detect drift| DB
    R -.->|cross-check| P
    R -.->|quarantine| SB
```

**SettlementBatch status (settlementBatch.model.js)**

```mermaid
stateDiagram-v2
    [*] --> calculated
    calculated --> scheduled
    calculated --> skipped_no_positive_payouts
    scheduled --> paid: all payouts paid
    scheduled --> failed: all payouts failed
    scheduled --> partial: mixed outcomes
    scheduled --> retryable: has retryable payouts
    partial --> paid
    retryable --> paid
    paid --> [*]
    skipped_no_positive_payouts --> [*]
```

**SettlementPayout status (settlementPayout.model.js)**

```mermaid
stateDiagram-v2
    [*] --> scheduled
    scheduled --> processing
    processing --> paid
    processing --> failed
    failed --> processing: payoutRetry claim
    failed --> manual_required: retries exhausted
    paid --> retryable: reversal requested
    retryable --> processing
    scheduled --> under_review: reconciliation flag
    paid --> under_review: reconciliation flag
    paid --> [*]
    manual_required --> [*]
```

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
  - `shippingLiabilityCents`: carrier-mode-only net
    (`shipping_fee + shipping_refund + shipping_label_cost`) — the Shipping Liability pool balance.
  - `sellerRetainedShippingCents`: flat/tiered shipping kept as seller revenue
    (`shipping_revenue + shipping_revenue_refund`), reported separately and never blended into
    `shippingLiabilityCents`.
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
(`tax/tax_refund/tax_remittance/shipping_fee/shipping_refund/shipping_label_cost/reserve/reserve_used/cod_refund_*`).

**Response shape**

- `data.summary`
  - `totalLiabilityAmountCents`
  - `outstandingLiabilityAmountCents`
  - `entryCount`
- `data.items[]`
  - grouped by `type + payoutStatus`
  - `type, payoutStatus, totalAmountCents, entryCount, lastCreatedAt`
- `meta` + `filters`

#### `GET /api/admin/platform-financials/tax-remittances`

Returns a paginated list of tax remittances recorded by the platform. Returns `404` when
`FEATURE_TAX_REMITTANCE=false`.

**Optional query params:** `page`, `limit`, `currency`

**Response shape:** `{ data: { items: [...], total, page, pages }, meta }`

#### `POST /api/admin/platform-financials/tax-remittances`

Records a tax payment made to a tax authority. Creates a `TaxRemittance` document and posts a
`tax_remittance` debit ledger entry against the platform tax-liability pool. Returns `404` when
`FEATURE_TAX_REMITTANCE=false`. Returns `422` on validation error (invalid period format or
non-positive amount).

**Request body:**

| Field          | Type   | Required | Notes                                                 |
| -------------- | ------ | -------- | ----------------------------------------------------- |
| `period`       | String | Yes      | `YYYY-MM` — the tax period being remitted             |
| `amountCents`  | Number | Yes      | Positive integer (cents)                              |
| `currency`     | String | Yes      | ISO 4217 code (e.g. `USD`)                            |
| `jurisdiction` | String | No       | Identifies the taxing authority; default `'default'`  |
| `filedAt`      | String | No       | ISO date — backdates the filing date; defaults to now |
| `notes`        | String | No       | Free-text filing notes                                |

**Response:** the created `TaxRemittance` document with `status: 'filed'` and `ledgerEntryId`.

### Seller financial summary behavior updates

Endpoint: `GET /api/seller/financials/summary`

> Quick interpretation: keep **settlement payout balances** and **subscription dues** separate in
> seller/admin UI logic. Settlement readiness should be based on settlement fields, not subscription
> dues.

- For normal sellers, summary reflects the seller's rows in requested/default scope.
- For platform-store seller context (admin acting as platform seller), default scope is
  merchant-only and summary is restricted to platform-owned order IDs.

**Response DTO (summary `data` object)**

- `currency`: currency code used for all cents fields (for example `USD`).
- `outstandingBalanceCents`: **legacy compatibility field** backward-compatible alias for unsettled
  settlement balance.
- `merchantOutstandingBalanceCents`: unsettled payout balance for seller-facing consumers.
- `settlementOutstandingBalanceCents`: unsettled settlement balance alias; same cents value as
  `outstandingBalanceCents` today for compatibility.
- `subscriptionOutstandingDuesCents`: unpaid subscription dues included in liabilities.
- `merchantGrossCents`: merchandise-sale gross amount from merchant settlement mix.
- `merchantTaxCents`: tax component in merchant settlement mix.
- `merchantShippingCents`: shipping component in merchant settlement mix.
- `merchantCommissionCents`: commission effect (typically negative) in merchant settlement mix.
- `merchantNetCents`: net before processor fees (`gross + tax + shipping + commission`).
- `customerPaidTotalCents`: net total paid by customer after refunds
  (`merchandiseSubtotalCents + passThroughTaxCollectedCents + passThroughShippingCollectedCents + sellerRetainedShippingCents`).
  Customer-paid shipping is mode-independent — it includes carrier pass-through shipping and
  seller-retained (flat/tiered) shipping, whichever applies to the order.
- `merchandiseSubtotalCents`: merchandise net — `sale` entries (positive) plus `refund` entries
  (negative), so partial/full refunds reduce this value.
- `passThroughTaxCollectedCents`: tax pass-through net (`abs(tax) − abs(tax_refund)`). Tax refunds
  are subtracted so the total decreases after a refund, matching the platform-side signed-sum
  convention.
- `passThroughShippingCollectedCents`: carrier-mode shipping pass-through net
  (`abs(shipping_fee) − abs(shipping_refund)`). Shipping refunds are subtracted so the total
  decreases after a refund.
- `sellerRetainedShippingCents`: flat/tiered (self-fulfilled) shipping kept as seller revenue
  (`shipping_revenue + shipping_revenue_refund`, signed sum). Zero unless
  `FEATURE_SELLER_SHIPPING_PAYOUT=true` and the order's `shippingMode` is `flat`/`tiered`.
- `settlementPayableBeforeReserveCents`: net seller-settlement amount before reserve withholding
  (this is the reserve-calculation base, not pass-through customer collections).
- `reserveWithheldCents`: reserve amount currently withheld.
- `availablePayoutNowCents`: payable now after reserve withholding.
- `liabilitiesSummary`: grouped liabilities totals returned by liabilities summarizer.
- `deductionTotalsCents`: grouped deduction totals keyed by deduction type.
- `reserve.byCurrency[]`: reserve amounts grouped by currency.
- `reserve.percentage`: configured reserve percentage for seller.
- `reserve.tier`: reserve tier label (for example `new`, `standard`).
- `codPolicy.negativeThresholdCents`: COD block threshold.
- `codPolicy.codAllowed`: whether COD is currently allowed.
- `codPolicy.blockedReason`: non-null only when COD is blocked.

These are derived from unpaid merchant-side settlement types and are intended as seller-operator
readable business totals.

**Exact formulas (cents, aligned to current backend implementation)**

- Settlement balance = `merchantOutstandingBalanceCents` (same value currently returned as
  `outstandingBalanceCents` in summary response; legacy semantics are intentionally preserved).
- `settlementPayableBeforeReserveCents` = payout-eligible net base used to calculate reserve
  withholding.
- `availablePayoutNowCents = settlementPayableBeforeReserveCents - reserveWithheldCents`.
- Net proceeds before processor fees = `merchantNetCents` where
  `merchantNetCents = merchantGrossCents + merchantTaxCents + merchantShippingCents + merchantCommissionCents`.
- Deductions total = sum of values in `deductionTotalsCents` (currently seeded with `commission`,
  `processing_fee`, `cod_refund_reimbursement`, plus any additional deduction keys returned by API).
- Sign convention: positive amounts increase seller receivable; negative amounts reduce it.

**Outstanding balance interpretation**

- `outstandingBalanceCents` / `merchantOutstandingBalanceCents` /
  `settlementOutstandingBalanceCents` are unsettled ledger totals for included types and scope, not
  "cash available now".
- Positive values generally indicate net receivable; negative values indicate net owed/offset state.
- COD policy consumes this scoped value against `negativeThresholdCents`.
- Backward compatibility: legacy consumers of `outstandingBalanceCents`,
  `merchantOutstandingBalanceCents`, and `settlementOutstandingBalanceCents` continue to receive the
  same unsettled payout-balance meaning; reserve-aware cash availability should use
  `availablePayoutNowCents`.

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
- When `sellerRetainedShippingCents` is non-zero (`FEATURE_SELLER_SHIPPING_PAYOUT=true` and the
  seller's orders use `flat`/`tiered` shipping), `SellerFinancialsPanel` renders an additional
  "Shipping revenue (you keep)" card (`seller.financials.summary.sellerRetainedShipping`) showing
  this amount alongside the existing summary cards.

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
  - `GET /tax-remittances`
  - `POST /tax-remittances`

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
RESERVE_RELEASE_BATCH_LIMIT=500   # max rows processed per release cycle (default 500)


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
RESERVE_RELEASE_BATCH_LIMIT=500   # max rows processed per release cycle (default 500)

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

Any reserve with an elapsed `releaseScheduledAt` is eligible regardless of which calendar month it
originated. The release is not gated on the calendar month of the reserve — same-month releases are
included when their hold period has elapsed.

Processing behavior:

- Processes at most `RESERVE_RELEASE_BATCH_LIMIT` eligible rows per cycle (default `500`). Set this
  env var to control per-cycle DB load.
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

**Reconciliation response types**

- `none`: no discrepancy detected.
- `reserve_discrepancy`: gross mismatch explained by reserve withholding math while net payout
  remains consistent.
- `hard_mismatch`: material state/amount mismatch requiring operator investigation. and Stripe.

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
- Reconciliation/financial summary emails are emitted on the same cron cadence to configured
  recipients/role.
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

**Reconciliation: batch + payout cross-check, quarantine on hard mismatch**

```mermaid
sequenceDiagram
    participant Cron as settlementReconciliation job
    participant Recon as reconciliation.service
    participant DB as MongoDB
    participant Stripe as Stripe
    Cron->>Recon: checkBatchConsistency(batchId)
    Recon->>DB: batch + payouts + ledger aggregate
    Recon->>Recon: net ledger vs batch vs payouts
    Cron->>Recon: checkPayoutWithStripe(payoutId)
    alt externalTransferId present
        Recon->>Stripe: transfers.retrieve(id)
    else
        Recon->>Stripe: transfers.list -> match metadata
    end
    alt transfer missing / status mismatch
        Recon->>DB: payout -> under_review (quarantine)
    end
    Recon-->>Cron: discrepancy report (emailed)
```

### Settlement Payout Retry & Cron Operations

**Operator action endpoints (quick map):**

- Retry failed payout: `POST /api/admin/settlements/payouts/:payoutId/retry`
- Reverse paid payout (Stripe transfer reversal):
  `POST /api/admin/settlements/payouts/:payoutId/reverse`
- Manually complete payout after off-platform transfer:
  `POST /api/admin/settlements/payouts/:payoutId/manual-complete`

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

##### Distributed Redis locks (non-scheduler cron jobs)

Five non-scheduler cron jobs — settlement reconciliation, payout retry, COD payout reconciliation,
subscription renewal, and the seller closure finalizer — now each acquire a Redis distributed lock
before executing. This prevents duplicate runs across multiple app replicas. Lock keys and TTLs are
documented in [`docs/settlements-runbook.md`](docs/settlements-runbook.md).

Each job wraps its callback with `wrapCronCallback`, which catches and logs unhandled rejections
from within the cron body so a single-job failure cannot crash the Node process.

**Boot-kickoff behavior:** all four jobs default to `*_RUN_ON_BOOT=false`, meaning they only trigger
on their configured cron schedule. This is the safe default for multi-replica deployments where
process startup is staggered. See Environment Variables below for the full list of `RUN_ON_BOOT`
vars.

##### Job-health monitoring

- **Stale scheduler detection:** job-health raises alerts if `settlement_scheduler` has not
  completed inside `JOB_HEALTH_SCHEDULER_STALE_HOURS`.
- **Payout retry failure-rate alerts:** payout retry runs are windowed and failure-rate alerts are
  triggered when configured minimum-run and threshold conditions are met.
- **Alert fan-out:** alerts are sent via email to the configured admin recipient list and optionally
  to a Slack channel via `OPS_SLACK_WEBHOOK`.
- **Admin dashboard health indicator:** the Admin page surfaces a compact scheduler-health pill
  (`healthy` / `stale` / `unknown`) based on `/api/admin/jobs/health` for quick operator triage.
- **Prometheus + Datadog hooks:** settlement scheduler and payout retry jobs emit counters that can
  be scraped/ingested (`/api/monitoring/metrics` for Prometheus; `datadog.metric` structured logs
  for Datadog log pipelines).

#### 2) Environment variables reference

| Variable                                          | Purpose                                                                          | Accepted format / range                                                                   | Operational effect                                                                                                           |
| ------------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `SETTLEMENT_PAYOUT_RETRY_ENABLED`                 | Enables/disables payout retry cron scheduling.                                   | Boolean-like (`true/false`, `1/0`, `yes/no`, `on/off`).                                   | `false` disables automatic recovery, so failed payouts remain failed/manual until operators act.                             |
| `SETTLEMENT_PAYOUT_RETRY_CRON`                    | Cron cadence for retry cycles.                                                   | Valid cron expression (node-cron syntax).                                                 | Faster cadence reduces time-to-recovery but increases Stripe/API pressure and log volume.                                    |
| `SETTLEMENT_PAYOUT_RETRY_MAX_RETRIES`             | Max failed retry attempts before escalation.                                     | Positive integer (`>=1`).                                                                 | Lower values escalate to `manual_required` faster; higher values increase auto-retry persistence before manual intervention. |
| `SETTLEMENT_SCHEDULER_TZ`                         | Timezone used by settlement-related cron jobs.                                   | Valid IANA timezone (e.g., `UTC`, `America/New_York`). Invalid values fall back to `UTC`. | Controls when cron expressions fire in wall-clock terms; mismatches can cause off-hours execution.                           |
| `JOB_HEALTH_SCHEDULER_STALE_HOURS`                | Staleness threshold for scheduler-health alerting.                               | Positive integer hours.                                                                   | Lower values alert sooner for stuck schedulers; overly low values can create noisy false positives.                          |
| `JOB_HEALTH_PAYOUT_FAILURE_RATE_THRESHOLD`        | Failure-rate alert threshold for payout retry job.                               | Decimal between `0` and `1` inclusive.                                                    | Lower thresholds make alerting more sensitive to degradation; higher thresholds reduce alert frequency.                      |
| `JOB_HEALTH_PAYOUT_FAILURE_WINDOW_RUNS`           | Number of latest runs evaluated for payout failure rate.                         | Positive integer.                                                                         | Smaller windows react quickly to spikes; larger windows smooth volatility and react slower.                                  |
| `JOB_HEALTH_PAYOUT_FAILURE_MIN_RUNS`              | Minimum runs required before failure-rate alerts can trigger.                    | Positive integer.                                                                         | Prevents early noisy alerts on low sample sizes; too high can delay real incident detection.                                 |
| `SETTLEMENT_RECONCILIATION_REPORT_EMAILS`         | Explicit recipient list for reconciliation/retry/reversal failure notifications. | Comma/semicolon separated email list.                                                     | Empty value suppresses direct email notifications; configured value provides immediate operator visibility.                  |
| `SETTLEMENT_RECONCILIATION_MAX_REPORT_RECIPIENTS` | Cap on the number of email recipients for reconciliation reports.                | Positive integer (default `5`).                                                           | Prevents accidental wide distribution of financial alert emails.                                                             |
| `OPS_SLACK_WEBHOOK`                               | Optional Slack incoming webhook URL for job-health alert fan-out.                | Full Slack webhook URL.                                                                   | When set, job-health alerts are posted to the configured Slack channel in addition to email recipients.                      |
| `SETTLEMENT_RECONCILIATION_RUN_ON_BOOT`           | Run settlement reconciliation job once immediately on server start.              | Boolean-like (`true/false`). Default `false`.                                             | Set `true` only on single-instance deployments where an immediate first run is required.                                     |
| `PAYOUT_RETRY_RUN_ON_BOOT`                        | Run payout retry job once immediately on server start.                           | Boolean-like (`true/false`). Default `false`.                                             | See `SETTLEMENT_RECONCILIATION_RUN_ON_BOOT` notes.                                                                           |
| `COD_PAYOUT_RECONCILIATION_RUN_ON_BOOT`           | Run COD payout reconciliation job once immediately on server start.              | Boolean-like (`true/false`). Default `false`.                                             | See `SETTLEMENT_RECONCILIATION_RUN_ON_BOOT` notes.                                                                           |
| `SUBSCRIPTION_RENEWAL_RUN_ON_BOOT`                | Run subscription renewal job once immediately on server start.                   | Boolean-like (`true/false`). Default `false`.                                             | See `SETTLEMENT_RECONCILIATION_RUN_ON_BOOT` notes.                                                                           |
| `SELLER_CLOSURE_FINALIZER_RUN_ON_BOOT`            | Run seller closure finalizer job once immediately on server start.               | Boolean-like (`true/false`). Default `false`.                                             | See `SETTLEMENT_RECONCILIATION_RUN_ON_BOOT` notes.                                                                           |

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

**Failure-mode to recovery mapping**

```mermaid
graph TD
    A{"Failure mode"}
    A -->|Stripe outage / timeout| B["retryable error -> backoff retry<br/>(settlementExecution)"]
    A -->|duplicate webhook| C["Redis NX dedup -> ack & ignore"]
    A -->|failed transfer| D["payout failed -> payoutRetry.job"]
    D --> E{"retryCount < max?"}
    E -->|yes| F["claim & retry<br/>(new idempotency key)"]
    E -->|no| G["manual_required (dead-letter)<br/>+ alert email"]
    A -->|COD payout stuck| H["codPayoutReconciliation poll<br/>-> complete / manual review"]
    A -->|ledger/payout drift| I["reconciliation -> under_review"]
```

**Failed payout recovery (payoutRetry.job)**

```mermaid
sequenceDiagram
    participant Cron as payoutRetry.job
    participant Lock as Redis lock
    participant DB as MongoDB
    participant Stripe as Stripe
    Cron->>Lock: acquire settlement:payout-retry:lock
    Cron->>DB: find payouts {status: failed, retryCount < max}
    loop each payout
        Cron->>DB: findOneAndUpdate claim<br/>(failed -> processing, guard retryCount)
        alt not claimed
            DB-->>Cron: skip (already taken)
        else claimed
            Cron->>Stripe: executeStripePayout (new idempotency key)
            alt paid
                Cron->>DB: status paid, retryCount 0
            else failed & retryCount >= max
                Cron->>DB: status manual_required + alert email
            else failed
                Cron->>DB: status failed, retryCount++
            end
        end
    end
```

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

### Size-chart visibility and category slug naming (Product Page)

On `frontend/src/components/ProductPage.jsx`, size-chart visibility depends on **both**:

1. A variant attribute that looks like size (`size`, `sizes`, `Size`, etc.).
2. A resolvable size-chart key from product category/subcategory metadata.

Recommended conventions for reliable matching:

- Always store the **leaf subcategory slug** on products (`subCategorySlug`) when applicable.
  - Example tree: `fashion -> women-fashion -> dress`
  - Product should carry `subCategorySlug: "dress"` (not only `categorySlug: "fashion"`).
- Use lowercase, URL-safe slugs with hyphens (no spaces/underscores), e.g.:
  - `dress`, `jackets`, `jeans`, `shoes`, `suits`, `hats`, `glasses`, `bags`, `wallets`.
- For t-shirts, use one of the supported forms:
  - `t-shirt`, `tshirts`, `t-shirts`, `tee`, `t shirts`.
  - The resolver aliases these to the existing t-shirt chart key.
- For dresses, `dress` is supported and is aliased to `dresses`.

Notes:

- Category **name** fields are used as fallback hints, but slug-based matching is more reliable.
- If a product has no size-like variant attribute, the size-chart link is intentionally hidden even
  when category resolution succeeds.

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
  - A `session_active_since:<userId>` key is set in Redis to track the current active session window
    (default 30 days). Activity resets the expiry, implementing a **sliding window** rather than a
    hard cap.

- Token rotation:
  - `/api/auth/refresh-token` validates the cookie against Redis and atomically rotates to a new
    `jti` to prevent reuse of stolen tokens.
  - A dedicated rate limiter on the refresh-token route is enforced at the router level
    (`auth.route.js`) to limit brute-force refresh attempts. The cap is configurable via
    `REFRESH_RATE_LIMIT` (default 30 per IP per 60 s) to leave headroom for legitimate background
    traffic: page boots, per-tab heartbeats, and mobile apps behind shared CGNAT IPs.

- Sliding window:
  - Each successful refresh or keep-alive `EXPIRE`s the `session_active_since:<userId>` key for
    another 30 days, extending the session as long as the user stays active.
  - If a user is inactive for 30 days (no refresh / keep-alive), `session_active_since` expires —
    server rejects further refreshes and requires re-login.

- Session invalidation:
  - **Password reset**, **`unlinkSocial`**, and **profile password change** all invalidate every
    existing refresh token for the user (all `refresh_token:<userId>:*` keys deleted) and clear both
    the access-token and refresh-token cookies. The `session_active_since:<userId>` heartbeat key is
    also removed before issuing new credentials. This ensures a compromised token cannot survive a
    password change.
  - `keepAlive` verifies the `session_active_since:<userId>` Redis key exists before minting a new
    access token; if the key has expired (inactive session), the request is rejected and the client
    must re-authenticate.
  - On logout, after revoking the current device's refresh token, the server checks whether any
    other devices still hold active refresh tokens for this user. If none remain,
    `session_active_since:<userId>` is also deleted.

- OAuth social-login account linking:
  - When a user signs in with Google or Facebook and a local account with the same verified email
    already exists, the provider identity is automatically linked onto the existing account rather
    than returning a 409 conflict or creating a duplicate. This applies symmetrically to both
    providers.
  - `email_verified` from Google is coerced to boolean (`true`, `"true"`, and `1` are all accepted)
    to handle workspace and edge accounts that return non-boolean values.
  - A controller-level fallback catches E11000 duplicate-key errors: if the email already exists and
    the provider has verified it, the account is auto-linked and the user is logged in.
  - Provider-specific verification rules gate auto-link to prevent account takeover: Google requires
    `email_verified`; Facebook requires the email to be the account's primary, confirmed address.
  - If the matched local account was **unverified** (email/password signup whose code was never
    entered), the auto-link sets `verified = true`. This is intentional and safe: the provider has
    already proven the user controls that mailbox, which is exactly what the email-verification code
    proves. The original password remains valid, so after linking the account can sign in with
    either email/password **or** the OAuth provider. Local password login stays blocked only while
    the account is still unverified and no OAuth link has occurred.

- Web checkAuth single-flight lock:
  - `checkAuth()` uses a single-flight lock that ensures only one refresh-token rotation runs at a
    time. Concurrent callers (React StrictMode double-mount, multi-tab visibility changes) wait for
    the in-flight rotation to complete and share its result instead of racing.

- Email enumeration prevention:
  - `POST /api/auth/login` returns a generic `401 Unauthorized` for both unknown email addresses and
    incorrect passwords — the response body does not distinguish between the two cases.
  - `POST /api/auth/forgot-password` adds a 250 ms artificial delay for requests where the email
    address does not match any account, equalizing response timing with the code-path that does find
    a user.

- Email-verification token security:
  - The raw token sent to the user's email is **SHA-256 hashed** before being stored in the
    database. Comparison at verification time also uses the hash, so a database dump cannot be used
    to verify arbitrary addresses without the original token.

- Keep-alive authentication:
  - `GET /auth/keep-alive` does **not** require a pre-issued access token. The handler
    self-authenticates by cryptographically verifying the refresh JWT and issuing a fresh access
    token. Only a valid refresh token (httpOnly cookie or `Authorization: Bearer <token>` header) is
    needed.

- Redis resilience in keep-alive and refresh-token:
  - If Redis returns a network error during a `keepAlive` or `refreshToken` call, the server
    responds with `503 Service Unavailable` so clients can retry rather than logging out.
  - If the refresh-token key is absent from Redis (cache miss) but the JWT signature is valid, both
    handlers re-store the token in Redis (cache-heal) instead of issuing a `401`. This prevents
    spurious logouts caused by Redis eviction or brief unavailability. The refresh handler skips the
    heal when a rotation grace marker exists for the jti, so a token that was genuinely rotated away
    moments earlier cannot be resurrected by a replay.
  - Clients treat **only a `401`** from the refresh/keep-alive endpoints as a dead session.
    Transient failures (network errors, `429` rate limits, `5xx`, `503`) never clear stored
    credentials: the web `checkAuth` falls back to `GET /auth/keep-alive` (which self-authenticates
    via the refresh cookie, no CSRF token needed) and retries the profile fetch once; the mobile
    bootstrap keeps SecureStore tokens and shows the cached profile until a definitive `401`.

- Mobile friendliness:
  - The mobile app runs a proactive session heartbeat: every 10 minutes and on app foreground, it
    calls `GET /auth/keep-alive` with the stored refresh token as a Bearer header. Keep-alive slides
    the Redis TTLs **without rotating** the refresh token, so an app kill mid-heartbeat can never
    orphan the token persisted in SecureStore (rotation remains reserved for `/auth/refresh-token`,
    used at bootstrap and on `401`s).
  - For mobile clients (`X-Mobile-Client: true`), keep-alive also returns the freshly minted access
    token in the response body — React Native cannot read httpOnly cookies.
  - `GET /auth/keep-alive` accepts both httpOnly cookie refresh tokens and
    `Authorization: Bearer <refresh-token>` headers, so mobile clients can call it without relying
    on cookies.
  - On app restart, mobile bootstrap checks whether the stored access token is expired and silently
    refreshes it before fetching the user profile. Network errors fall back to cached user data; a
    `401` response triggers re-authentication rather than falling back to the cache.
  - Web frontend also implements mobile-aware keep-alive intervals and additional activity listeners
    (touchstart, focus) to counteract background throttling on mobile browsers.

**UX rule implemented:** sessions persist for the full 30-day sliding window — any activity
(refresh, keep-alive) resets the clock. Redis resilience ensures transient cache issues never cause
an unnecessary logout.

---

**Login & authenticated request (JWT + Redis session)**

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Express API
    participant DB as MongoDB
    participant R as Redis
    C->>+API: POST /auth/login
    API->>+DB: find user + bcrypt.compare
    DB-->>-API: user record
    API->>R: SET userId:jti (TTL 30d)
    API-->>-C: Set-Cookie: jwt(15m) + refresh-token(30d)
    Note over C,API: Authenticated request
    C->>+API: GET /api/protected (jwt cookie)
    API->>API: verify JWT signature + expiry
    API-->>-C: 200 OK
```

**Refresh-token rotation**

```mermaid
sequenceDiagram
    participant C as Client
    participant API as Express API
    participant R as Redis
    C->>+API: POST /auth/refresh (refresh-token cookie)
    API->>API: verify HMAC signature + expiry
    API->>+R: GET userId:jti
    R-->>-API: jti valid
    API->>R: DEL old jti
    API->>R: SET new jti (TTL 30d)
    API-->>-C: Set-Cookie: jwt(new 15m) + refresh-token(rotated)
    Note over C,API: On revoked or expired token
    API-->>C: 401 — clear cookies
```

## Order fulfilment, PDF export & canceled order labeling

- Admin UI supports exporting order details to PDF for fulfilment and label printing. Export
  includes order lines, shipping address, and order metadata.
- Canceled orders are explicitly labeled in the admin (status `cancelled` / `payment_failed` /
  `refunded`) and included in exports as such for tracking.
- PDF export utility is implemented server-side (PDF generation libraries) and exposed to the admin
  for download/printing.

### Order status transition enforcement

The backend enforces a strict `ALLOWED_TRANSITIONS` map that defines which status moves are legal
for each order state:

- Sellers must advance orders one step at a time along the fulfilment chain
  `placed → processing → dispatched → delivered`; statuses cannot be skipped.
- **Cancellation** is allowed from any non-terminal status. Terminal states (`delivered`,
  `cancelled`, `refunded`) accept no further transitions.
- Attempts to jump directly from `order_placed` to `delivered` (or any other invalid hop) are
  rejected with a `400` carrying a human-readable message, before any database write occurs.

**Frontend behaviour:**

- The `OrderRow` status dropdown in the seller and admin panels renders only the statuses that are
  valid next steps for the current order state, preventing operators from attempting invalid
  transitions in the first place.
- When a backend validation error is returned, a friendly toast message is shown instead of the raw
  error object that previously crashed the UI.

**COD cancellation fix:**

COD orders in early statuses (`order_placed`, `processing`) can now be cancelled without requiring
the seller to have completed COD payout bank details. The eligibility check for COD-specific payout
fields is skipped until the order has reached a stage where a refund disbursement is actually
needed.

**Order status state machine (order.model.js)**

```mermaid
stateDiagram-v2
    [*] --> order_placed: prepaid checkout completed
    [*] --> cod_order_placed: COD checkout
    [*] --> payment_failed: payment_intent.payment_failed
    payment_failed --> order_placed: checkout.session.completed (recovered)
    payment_failed --> expired: session expires
    order_placed --> processing
    cod_order_placed --> processing
    processing --> dispatched
    dispatched --> delivered: recordDeliveryLedger() posts entries
    order_placed --> pending_refund: partial refund
    delivered --> pending_refund: partial refund
    pending_refund --> refunded: full refund completed
    order_placed --> refunded: full pre-delivery refund
    delivered --> refunded
    pending_refund --> refund_rejected
    order_placed --> cancelled
    delivered --> externally_refunded_needs_review: refund outside platform
    refunded --> [*]
    cancelled --> [*]
    expired --> [*]
```

### Shipment model

The dedicated `Shipment` model (`backend/src/models/shipment.model.js`) is stored separately from
the `Order` document, supporting multi-store orders where each store ships independently.

```
Fields
  orderId          — reference to the parent Order
  storeId          — the store fulfilling this shipment
  sellerId         — owning seller
  itemIndexes      — which order line items are covered
  carrier          — carrier name (e.g. "aramex", "manual")
  carrierService   — service level (e.g. "express", "ground")
  trackingNumber   — carrier-issued tracking number
  trackingUrl      — public carrier tracking page URL
  labelUrl         — pre-signed label download URL (server-side only; stripped from customer API)
  labelExpiresAt   — label link TTL
  shipmentStatus   — enum: created → label_printed → picked_up → in_transit →
                     out_for_delivery → delivered → returned → exception → cancelled
  carrierStatusRaw — raw carrier status string for debugging
  source           — "manual" (seller-entered) or "api" (carrier-generated)
  labelCostCents   — carrier label cost in cents (api-source labels only; absent for manual shipments)
  labelCostCurrency — ISO 4217 currency code for labelCostCents
  labelCostSource  — "api" (live Shippo label) or "sandbox" (Shippo sandbox; $0, excluded from ledger)
  shippedAt        — timestamp of first shipment (also stamped on Order.shippedAt)
  deliveredAt      — confirmed delivery timestamp
  lastSyncedAt     — last carrier webhook sync
  providerMetadata — carrier-specific response data (arbitrary JSON)
  events[]         — append-only timeline of carrier status events
  createdBy        — user who created the shipment record
```

**`order.shippingMode`** (`'flat' | 'tiered' | 'carrier' | null`) is a separate field on the `Order`
document (not `Shipment`) — a snapshot of the **effective**, region-specific shipping mode for the
order's destination at checkout time (`resolveEffectiveShippingMode`: a per-region
`shipping.regions[].mode` override, falling back to the store-level `shipping.mode`).
`recordDeliveryLedger`/`recordRefundLedger` read it via `resolveShippingMode(order)` to decide
whether collected shipping is posted as a carrier pass-through liability (`shipping_fee`) or, when
`FEATURE_SELLER_SHIPPING_PAYOUT=true` and the mode is `flat`/`tiered`, payout-eligible seller
revenue (`shipping_revenue`). Orders placed before this field existed have `shippingMode: null` and
fall back to `'flat'`.

Creating the first shipment on an order transitions the order from `processing` → `dispatched` and
stamps `Order.shippedAt`. If the order is already `dispatched` when the first shipment is added
(e.g. set by an admin), the status is unchanged but `onOrderStatusChanged` still fires so the
customer receives the shipped email and notification.

**Dispatch gate:** The seller cannot manually transition an order to `dispatched` until at least one
shipment carries a non-empty `trackingNumber`. The `updateSellerOrderStatus` endpoint returns
`400 tracking_required` when this condition is not met. The dispatched `<option>` in the seller
order-status dropdown is also disabled in the UI until the condition is met.

### Carrier service abstraction

The shipping layer follows the same adapter-map pattern as COD payouts
(`backend/src/services/shipping/`):

- `index.js` — dispatcher: resolves the active provider from `SHIPPING_PROVIDER` and delegates to
  the matching adapter.
- `providers/index.js` — adapter map keyed by provider name.
- `providers/manual.adapter.js` — accepts caller-supplied tracking number; no external API call.
- `providers/aramex.adapter.js` — Aramex Shipping API integration (sandbox: create shipment, track
  shipment, status mapping).
- `providers/shippo.adapter.js` — Shippo integration (Phase 3): live multi-carrier rates
  (`getRates`), label purchase (create), and tracking (`track`). Single platform-wide account via
  `SHIPPO_API_KEY`. When a label is purchased the adapter extracts the carrier cost from the Shippo
  transaction response and surfaces `labelCostCents`, `labelCostCurrency`, and `labelCostSource`
  (`'api'` for live labels, `'sandbox'` for Shippo test-mode labels). The controller persists these
  fields on the `Shipment` document and then calls `recordShippingLabelExpense`, which posts a
  `shipping_label_cost` debit against the Shipping Liability seller — closing the accounting loop
  between shipping fees collected from customers and actual carrier spend. Sandbox labels are tagged
  and skipped so test data never enters the production ledger. Gated by
  `FEATURE_SHIPPING_LABEL_LEDGER`.
- `providers/disabled.adapter.js` — returns a safe `{ ok: false }` no-op without throwing; default
  for local development.

Adapters share a normalized result shape, idempotency keys, and retry-with-backoff. Set
`SHIPPING_PROVIDER=disabled` for local development, `manual` for operator-entered tracking, `aramex`
for the Aramex carrier API, or `shippo` for Shippo label/tracking.

**Carrier-calculated rates (`shipping.mode='carrier'`).** A seller can opt a store into live carrier
rates. At checkout, `resolveStoreShippingRate` (async wrapper around the pure
`resolveStoreShipping`) calls Shippo `getRates` for the origin → destination → parcel and charges
the cheapest rate (+ handling, clamped). The parcel weight and dimensions are summed from
per-product `weight`, `dimensionLength`, `dimensionWidth`, and `dimensionHeight` fields, falling
back to the store's `shipping.carrier.defaultParcel`, then a global default (0.5 kg / 10 cm cube).
The shipping/ tax estimate endpoint uses the same product weight and dimensions so the quoted rate
during browsing matches what is charged at checkout. Gated by `FEATURE_SHIPPO_RATES`; when off (or
on any Shippo error/timeout) it falls back to `shipping.carrier.fallbackRate`, so checkout never
blocks on the carrier. Flat/tiered modes stay fully synchronous.

The rate picker shows the cheapest available rate **pre-selected** on load. The checkout uses the
`servicelevelToken` (not the carrier name string) to match the customer-chosen rate when purchasing
the Shippo label — this guarantees the correct service level is purchased even when two carriers
share a similar service name. When Shippo returns no rates the UI surfaces the reason from Shippo
(e.g. `rate_not_available`, `outside_service_area`) alongside freight-carrier guidance for sellers
whose parcels exceed standard carrier size or weight limits.

**Customer tracking.** `GET /orders/:orderId/shipments/:shipmentId/tracking` (owner-only) performs a
throttled live carrier sync (gated by `FEATURE_SHIPPO_TRACKING`) and persists the latest events; the
order-details "Track My Shipment" action (web + mobile) drives it.

### Seller shipment management

- `POST /seller/orders/:id/shipments` — create a shipment. Sellers choose one of two paths:
  - **Manual entry** — enter a tracking number + carrier directly; no external API call.
  - **Generate Label** (`source:'api'`) — the backend calls Shippo to purchase a label using the
    carrier rate that was stored on the order at checkout. If that rate has expired Shippo re-quotes
    automatically before purchase. The label URL (`labelUrl`) is signed and stripped from the
    customer-facing shipment response.
- `GET /seller/orders/:id/shipments` — list all shipments on an order.
- Zustand store actions: `fetchShipments`, `createShipmentManual`, `createShipmentApi` (with
  optimistic update and rollback on error).
- The `ShipmentForm` component (`frontend/src/components/SellerOrdersPanel.jsx`) exposes both paths
  in a toggle; the Generate Label button is only shown when `FEATURE_SHIPPO_RATES=true` and the
  store's shipping mode is `carrier`.
  - **Manual entry** supports a custom carrier name: selecting "Other" in the carrier dropdown
    reveals a free-text input so sellers can specify a carrier not on the preset list.
  - An optional **Tracking URL** field lets the seller paste the carrier's public tracking page URL;
    when provided it is stored on the shipment, used as the notification `link`, and included in the
    shipped email as a clickable link.

### Customer order tracking timeline

- `GET /api/orders/:id/shipments` — owner-only; returns shipment data without `labelUrl`.
- `OrderShipmentTimeline` component (`frontend/src/components/OrderShipmentTimeline.jsx`) replaces
  the text-only order status with a visual horizontal stepper:
  `processing → shipped → in_transit → out_for_delivery → delivered`.
- Supports multiple shipments per order (multi-store orders).
- Shows "Track on carrier site" link when `trackingUrl` is present.
- i18n keys: `orderModal.shipment.*` across all 4 locales.

### Carrier webhook endpoint

- `POST /api/webhooks/shipping/:provider` — generic webhook hub for incoming carrier status updates.
- HMAC signature verification mirrors the Chimoney webhook pattern.
- Idempotent deduplication by `(trackingNumber, carrierStatusRaw, eventTimestamp)`.
- Auto-transitions `Order.status` to `delivered` when the carrier confirms delivery.

### Admin shipment visibility

- `GET /api/admin/shipments` — paginated list with filters (status, storeId, carrier, date range).
- `OrderTab.jsx` (`frontend/src/pages/AdminPage.jsx`) shows a shipment badge and an expander row per
  order line.

---

## Product Reviews & Ratings

### Review model

The `Review` model (`backend/src/models/review.model.js`) stores customer feedback with the
following fields:

```
productId    — reviewed product
storeId      — store that sold the product
sellerId     — seller being reviewed
customerId   — reviewer
orderId      — purchase that gates eligibility
rating       — 1–5 integer
comment      — up to 1 000 characters
status       — published | flagged | hidden
flagReason   — reason set by automated filter or admin
hiddenBy     — admin who hid the review
hiddenAt     — hide timestamp
editedAt     — last edit timestamp
```

Compound unique index `{ customerId, productId, orderId }` enforces one review per purchase. Repeat
purchases earn separate reviews.

A denormalized `ratingAggregate` object (average, count, distribution 1–5) is stored on the
`Product` document and recomputed whenever a review's status changes.

### Profanity filter

`shared/profanity/profanityFilter.js` is a shared utility consumed by both the backend and the
mobile/web frontends.

- Detects profanity via `bad-words` with a custom multi-language blocklist covering Arabic, Arabic
  romanisation, Spanish, and French, plus ethnic slurs.
- Also rejects links, email addresses, and phone numbers embedded in review text.
- **Client side:** blocks form submission with a toast error before the request is sent.
- **Server side:** publishes the review immediately if clean; sets `status=flagged` if dirty (no
  manual approval queue for every review — only flagged ones require admin attention).

### Customer review endpoints

```
POST /api/reviews                       — create a review; verifies purchase, runs profanity
                                          check, publishes or flags.
GET  /api/products/:id/reviews          — public; published reviews only; paginated; sortable
                                          (newest / highest / lowest).
GET  /api/products/:id/rating-aggregate — public; average, count, and 1–5 distribution.
GET  /api/reviews/eligibility           — authenticated; returns whether the user may review a
                                          product (has a qualifying delivered order).
```

### Seller review endpoint

```
GET /api/seller/reviews   — seller-scoped; all statuses; paginated.
                            Sellers see all reviews on their products regardless of status.
```

### Admin review moderation

```
GET /api/admin/reviews?status=flagged    — list flagged reviews.
PUT /api/admin/reviews/:id               — reinstate (flagged → published) or hide
                                           (published → hidden); triggers rating aggregate
                                           recomputation.
```

The `AdminReviewsPanel` component (`frontend/src/pages/AdminPage.jsx`) provides a filter bar, table
with inline reinstate/hide actions, and pagination. A "Reviews" tab has been added to the Admin
Dashboard.

i18n keys: `admin.reviews.*` across all 4 locales.

### Email notification on new review

`sendNewReviewToSellerEmail()` in `backend/src/lib/email.js` fires when a review is published (not
when it is flagged). The email includes the product name, rating, comment excerpt, and a direct link
to the seller dashboard reviews view.

### Web & mobile review UI

- **Web:** `RatingAggregate`, `ReviewList`, and `ReviewForm` components are embedded in
  `ProductPage.jsx`. The submission form is gated by eligibility and runs the profanity check before
  sending the request.
- **Mobile:** equivalent inline review UI on `ProductScreen.js`.
- i18n keys: `reviews.*` across all 4 locales.

---

## Store Reviews

The `Review` model uses a `target` discriminator (`product` | `store`) to support store-level
reviews alongside product reviews.

### Store review routes

```
GET /api/stores/:slug/reviews           — public; published store reviews; paginated.
GET /api/stores/:slug/rating-aggregate  — public; average, count, and 1–5 distribution.
```

**Eligibility:** a customer must have at least one `delivered` order from the store (no `orderId` in
the unique key — one review per customer per store).

A denormalized `ratingAggregate` is also stored on the `Store` model and updated on status changes.

The customer-facing review UI is embedded in `StorefrontPage.jsx`.

---

## Dispute Resolution

All dispute functionality is gated by `FEATURE_DISPUTES_V1=true`.

### Dispute model

`backend/src/models/dispute.model.js` is the canonical document for every buyer–seller dispute.

```
Fields
  orderId             — parent order
  productId           — disputed product (optional; for item-level disputes)
  storeId / sellerId  — owning store and seller
  customerId          — buyer who opened the dispute
  issueType           — enum: item_not_received | item_not_as_described | item_damaged |
                         wrong_item | refund_not_received | other
  description         — up to 2 000 characters
  evidence[]          — array of { url, mimeType } uploaded at creation
  status              — 7-value state machine:
                         open → under_review → awaiting_seller_reply →
                         awaiting_buyer_reply → resolved | closed | withdrawn
  messages[]          — append-only thread; each entry has:
                         authorRole (customer | seller | admin | system),
                         authorUserId, body (up to 2 000 chars),
                         attachments[] { url, mimeType }
  resolution          — { decision, notes, refundOrderId, resolvedBy, resolvedAt }
  resolutionOffer     — seller/admin proposal: { offerType, amount, notes, status,
                         offeredBy, offeredAt, respondedAt }
                         offerType: refund_full | refund_partial | no_refund |
                                    replacement | other
                         status:    pending | accepted | rejected
  chargeback          — { provider: 'stripe', stripeDisputeId, amount, outcome }
  slaDueAt            — createdAt + 72 hours; used by the SLA escalation cron
  slaEscalationEmailedAt — guards against duplicate escalation emails
```

Compound indexes: `{ status, slaDueAt }`, `{ customerId }`, `{ sellerId }`.

`DisputeTemplate` model (`backend/src/models/disputeTemplate.model.js`) stores admin canned
responses with `title`, `body`, and `category` fields, managed via the template CRUD API.

### Customer dispute flow

**Eligibility:** a dispute may be opened on orders with status `delivered`, `dispatched`,
`refund_rejected`, or `refunded`. The endpoint rejects duplicate open disputes on the same order.

**Customer endpoints:**

```
POST /api/disputes                    — create a dispute; ownership check, duplicate prevention,
                                        optional evidence upload (multer + Cloudinary).
GET  /api/disputes                    — list own disputes; paginated.
GET  /api/disputes/:id                — view own dispute including full message thread.
POST /api/disputes/:id/messages       — append a message with optional evidence upload.
POST /api/disputes/:id/respond-offer  — accept or reject a seller resolution offer.
```

**Web UI:**

- `OrderDetailsModal` shows an **Open dispute** button only on eligible order statuses and hides it
  when an active dispute already exists.
- Dispute creation form, dispute list page (`/profile/disputes`), and dispute detail page with the
  full message thread, reply form with file upload, and a resolution offer accept/reject card.

**Mobile UI:**

- `OrderHistoryPage` shows an equivalent **Open dispute** CTA on eligible orders.
- `DisputeForm`, `DisputeListScreen`, `DisputeDetailScreen` with evidence upload and a full-screen
  image viewer for attachments.

**i18n keys:** `disputes.form.*`, `disputes.detail.*`, `disputes.list.*`, `disputes.issueTypes.*`,
`disputes.statuses.*`, `disputes.offer.*`, `orderModal.openDispute`, `profilePage.myDisputes` across
all 4 locales.

### Seller dispute endpoints

```
GET  /api/seller/disputes             — list disputes against own store; paginated.
GET  /api/seller/disputes/:id         — view single dispute (ownership enforced).
POST /api/seller/disputes/:id/messages — reply with optional evidence upload.
POST /api/seller/disputes/:id/offer   — propose a resolution offer (offerType + optional amount
                                         and notes). One pending offer allowed at a time.
```

**Web UI:** `SellerDisputesPanel` in `SellerDashboardPage` — a **Disputes** tab with expandable rows
showing the message thread, a reply form with file upload, and an **Offer resolution** button that
opens a modal to submit an offer.

**i18n keys:** `seller.disputes.*`, `seller.tabs.disputes` across all 4 locales.

### Admin dispute management

```
GET    /api/admin/disputes                    — list all disputes; filterable by status, sellerId,
                                               issueType; sorted by slaDueAt ascending.
GET    /api/admin/disputes/:id                — view any dispute.
PATCH  /api/admin/disputes/:id/status         — transition status through the state machine.
POST   /api/admin/disputes/:id/messages       — add a message with optional evidence upload.
POST   /api/admin/disputes/:id/resolve        — resolve: set decision, notes, and optional refund
                                               trigger (creates a refund order).
POST   /api/admin/disputes/:id/close          — force-close with notes.
GET    /api/admin/disputes/metrics            — aggregate metrics: volume, resolution rate, average
                                               resolution time; breakdowns by store, seller,
                                               issueType, and month.
GET    /api/admin/disputes/templates          — list canned response templates.
POST   /api/admin/disputes/templates          — create a template.
PUT    /api/admin/disputes/templates/:id      — update a template.
DELETE /api/admin/disputes/templates/:id      — delete a template.
```

**Web UI:** `AdminDisputesPanel` in `AdminPage` — a **Disputes** tab containing:

- Metrics KPI cards (volume, resolution rate, average resolution time).
- Filter bar (status, issueType, seller).
- Dispute table sorted by SLA due date; expandable rows with full message thread.
- Status transition buttons (move dispute through the state machine).
- **Resolve** modal with decision selector and refund toggle.
- **Close** modal with notes field.
- Reply form with file upload and a template selector (inserts a canned response body).
- **Manage templates** modal for CRUD on `DisputeTemplate` records.

**i18n keys:** `admin.disputes.*`, `admin.disputes.metrics.*`, `admin.disputes.templates.*`,
`adminPage.tabs.disputes` across all 4 locales.

### SLA enforcement

A daily cron (`disputeSlaEscalation.job.js`, default 08:00 UTC) finds non-terminal disputes where
`slaDueAt <= now`, appends a system message, and emails the admin list once per dispute (guarded by
`slaEscalationEmailedAt`). See
[Cron Jobs — Dispute SLA escalation cron](#3-dispute-sla-escalation-cron) for scheduler env vars and
test instructions.

### Stripe chargeback integration

Handled inside the existing `stripeWebhook` controller. See
[Stripe webhooks — chargeback events](#stripe-webhooks--async-handling--stripewebhook) for the full
event logic for `charge.dispute.created` and `charge.dispute.closed`.

### Evidence upload

`disputeUpload.middleware.js` wraps multer and is applied to:

- `POST /api/disputes` (creation evidence)
- `POST /api/disputes/:id/messages` (customer message attachments)
- `POST /api/seller/disputes/:id/messages` (seller message attachments)
- `POST /api/admin/disputes/:id/messages` (admin message attachments)

Files are processed and uploaded to Cloudinary. The resulting URLs and MIME types are stored in
`dispute.evidence[]` (creation) or `message.attachments[]` (message replies).

### Email notifications (disputes)

All dispute email helpers live in `backend/src/lib/email.js`:

| Function                        | Trigger                                      | Recipient            |
| ------------------------------- | -------------------------------------------- | -------------------- |
| `sendDisputeOpenedEmail`        | Dispute created                              | Seller + Admin       |
| `sendDisputeMessageEmail`       | New message added to thread                  | Opposing party       |
| `sendDisputeAwaitingReplyEmail` | Status transitions to `awaiting_*_reply`     | Relevant party       |
| `sendDisputeResolvedEmail`      | Dispute resolved or closed                   | Customer + Seller    |
| `sendDisputeSlaEscalationEmail` | SLA cron: `slaDueAt` passed, not yet emailed | Admin recipient list |

### i18n keys (disputes)

New translation keys added across all 4 locales (`en`, `es`, `fr`, `ar`) in `shared/locales/`:

```
disputes.form.*            — creation form labels, placeholders, validations
disputes.detail.*          — detail page labels, status badges, timeline
disputes.list.*            — list page headings, empty state, pagination
disputes.issueTypes.*      — human-readable labels for the 6 issue type enum values
disputes.statuses.*        — human-readable labels for the 7 status enum values
disputes.offer.*           — resolution offer card, accept/reject buttons, offer types
orderModal.openDispute     — "Open dispute" CTA in order details modal
profilePage.myDisputes     — "My disputes" link in the profile navigation
seller.disputes.*          — seller disputes panel labels and actions
seller.tabs.disputes       — "Disputes" tab label in the seller dashboard
admin.disputes.*           — admin disputes panel labels, filter bar, modals
admin.disputes.metrics.*   — KPI card labels and breakdowns
admin.disputes.templates.* — canned response template management labels
adminPage.tabs.disputes    — "Disputes" tab label in the admin dashboard
```

---

## Transactional email system

All transactional emails are implemented in `backend/src/lib/email.js` via the private
`sendTransactionalEmail()` helper. Resend is the active provider (`RESEND_API_KEY`); SendGrid is
available as a drop-in fallback by swapping the import. Translations are resolved with
`getNestedTranslation` from `@eshop/locales` so email copy stays in sync with the UI locale files.

**Delivery behaviour:**

- All calls are fire-and-forget — email failures are logged but never propagate to the caller.
- Anonymised customers (soft-deleted users with no email) are silently skipped.
- Failures never block the parent operation (order update, refund, payout execution, moderation
  decision, etc.).

**Sender domains & verification:**

- Two From identities: order/marketing mail uses `RESEND_FROM_EMAIL` (`ORDERS_FROM`); account mail
  (verification code, password reset) uses `SUPPORT_EMAIL` (`SUPPORT_FROM`), which falls back to
  `RESEND_FROM_EMAIL` when unset so a missing value never yields a malformed `Name <undefined>`.
- The From domain must **exactly match a domain verified in Resend**. The verified domain is the
  `email.vexflare.com` subdomain — **not** the apex `vexflare.com`. Sending from the apex makes
  Resend reject every message (`{ error }` with "Domain not verified"); the SDK returns that on a
  4xx rather than throwing, so the senders log it explicitly instead of silently returning `false`.
- `verifyResendSenderDomains()` runs at startup (`server.js`, non-blocking) and logs a warning if a
  configured sender domain isn't among the verified Resend domains (checked live via
  `domains.list()`, falling back to the `RESEND_VERIFIED_DOMAINS` allowlist), catching this
  misconfiguration at boot rather than at first send.

**Event catalogue (in addition to earlier order-confirmation and dispute emails):**

| Function                          | Trigger                             | Recipient |
| --------------------------------- | ----------------------------------- | --------- |
| `sendOrderShippedEmail`           | Order status → `dispatched`         | Customer  |
| `sendOrderDeliveredEmail`         | Order status → `delivered`          | Customer  |
| `sendRefundApprovedCustomerEmail` | Refund approved                     | Customer  |
| `sendRefundRejectedCustomerEmail` | Refund rejected                     | Customer  |
| `sendRefundApprovedSellerEmail`   | Refund approved                     | Seller    |
| `sendRefundRejectedSellerEmail`   | Refund rejected                     | Seller    |
| `sendPayoutCompletedEmail`        | Settlement payout marked `paid`     | Seller    |
| `sendPayoutFailedEmail`           | Settlement payout marked `failed`   | Seller    |
| `sendProductApprovedEmail`        | Admin approves a pending product    | Seller    |
| `sendProductRejectedEmail`        | Admin rejects a pending product     | Seller    |
| `sendProductActionRequiredEmail`  | Admin flags product needing changes | Seller    |

**i18n keys:** all new email copy lives under `email.transactional.*` in all 4 locale files (`en`,
`ar`, `es`, `fr`) in `shared/locales/`.

**Wiring locations:**

- Order status hook (`orderStatusChanged`) — fires shipped and delivered emails.
- Refund controllers — fire refund approved/rejected emails for customer and seller.
- Settlement execution service — fires payout completed/failed emails.
- Manual payout service — fires payout completed/failed emails.
- Product moderation controller — fires product approved/rejected/action-required emails.
- KYC review controller — fires KYC approval/rejection emails (pre-existing wiring extended).

For the full notification counterpart that fires alongside every email, see
[In-app notification centre](#in-app-notification-centre).

---

## In-app notification centre

Gated by `FEATURE_IN_APP_NOTIFICATIONS=true`.

### Notification model

`backend/src/models/notification.model.js` stores per-user notification documents.

```
Fields
  userId      — recipient (indexed)
  type        — one of 22 enum values (order_shipped, order_delivered, refund_approved,
                 refund_rejected, payout_completed, payout_failed, product_approved,
                 product_rejected, product_action_required, kyc_approved, kyc_rejected,
                 store_closed, store_closure_cancelled, store_closure_completed,
                 new_review, dispute_opened, dispute_message, dispute_resolved,
                 dispute_escalated, order_cancelled, new_order, new_dispute)
  titleKey    — i18n key resolved client-side (e.g. "notifications.order_shipped.title")
  bodyKey     — i18n key resolved client-side
  variables   — interpolation map (orderId, amount, productName, carrier, trackingUrl, etc.)
  read        — boolean; false by default
  createdAt   — timestamp
```

Rendering is done client-side by calling `t(titleKey, variables)` — switching the UI language
instantly updates all notification text without re-fetching from the server.

i18n keys live under `notifications.*` across all 4 locale files.

### REST API

All routes require authentication. Unread count is Redis-cached with a short TTL and invalidated on
write.

```
GET  /api/notifications               — paginated list, unread first
GET  /api/notifications/unread-count  — { count: N } (Redis-cached)
PATCH /api/notifications/:id/read     — mark one notification read
PATCH /api/notifications/read-all     — mark all notifications read
DELETE /api/notifications/:id         — delete one notification
```

### Web bell UI

- Bell icon in the Navbar with a red badge showing the unread count.
- Clicking the bell opens a dropdown panel listing the most recent notifications; unread rows have a
  distinct background tint and accent border using theme tokens.
- Clicking a notification row opens a detail modal (portal-rendered to `document.body`) instead of
  navigating away, preserving the current page context.
- A "View all" link opens the full notification list page.
- Polling interval: 60 seconds via `setInterval`; cancelled on unmount.
- All colours use existing CSS variables — no hardcoded values.

### Mobile bell UI

- Bell icon in the app header with a red dot badge mirroring web behaviour.
- Tapping the bell opens a bottom sheet listing notifications; tapping a row opens a detail modal.
- Polling is AppState-aware: paused when the app is backgrounded, resumed on foreground.
- Theme tokens from the generated palette are used throughout.

### `order_shipped` notification

The `order_shipped` notification is fired by the `onOrderStatusChanged` hook
(`backend/src/services/orders/orderFinancials.hook.js`) when an order transitions to `dispatched` —
either via the manual status dropdown or when the **first shipment** (label or manual tracking) is
created on the order. The `variables` map includes `carrier`, `trackingUrl`, and `orderId`. The body
key is chosen based on whether a tracking URL is available:

- `notifications.types.order_shipped.body` — shown when `trackingUrl` is present; the client renders
  it as a clickable link to the carrier tracking page.
- `notifications.types.order_shipped.bodyNoTracking` — shown when no tracking URL is available
  (manual shipments without a public URL).

The notification `link` field is set to `trackingUrl` when present, otherwise to `/orders/<orderId>`
so tapping the notification still navigates to the order detail page.

The companion **shipped email** (`sendOrderShippedEmail`) uses a **three-way template** keyed by
tracking state: no tracking → `htmlNoTracking` / `plainNoTracking`; tracking + URL → `html` /
`plain`; tracking without URL → `htmlTrackingNoUrl` / `plainTrackingNoUrl`. This prevents a broken
`<a href="">` link when a manual shipment has a tracking number but no public tracking URL.

### Wiring new notification types

Every place that fires a transactional email now also calls
`fireNotification(userId, type, variables)` immediately after. The two calls are independent — a
notification failure never suppresses the email and vice versa.

To add a new notification type:

1. Add the type string to the `type` enum in `notification.model.js`.
2. Add `titleKey` and `bodyKey` i18n entries in all 4 locale files under `notifications.<type>.*`.
3. Call `fireNotification(userId, type, variables)` at the relevant service/controller callsite.

---

## Admin user management

The admin panel includes a dedicated **Users** tab (`AdminUsersPanel` in
`frontend/src/pages/AdminPage.jsx`) for managing platform users.

**Backend endpoints (`/api/admin/users/*`):**

```
GET    /api/admin/users              — paginated list; filterable by role, status, and free-text search
POST   /api/admin/users/:id/suspend  — suspend a user account (body: { reason })
POST   /api/admin/users/:id/unsuspend — lift a suspension
DELETE /api/admin/users/:id          — permanently delete and anonymise (reuses deleteUserAccount service)
```

**Behaviour:**

- Suspended users are rejected at login with a clear error message; all active sessions are
  immediately invalidated via Redis token revocation.
- Admin-initiated deletion reuses the shared `deleteUserAccount` service, cascading PII
  anonymisation, seller data scrubbing, and `UserDeletionAudit` creation — identical to self-service
  deletion.
- Admins cannot suspend, unsuspend, or delete themselves or other admin accounts; these requests
  return 403.

**Schema additions (User model):**

```
suspendedAt      Date    — timestamp of most recent suspension
suspendedReason  String  — operator-supplied reason text
suspendedBy      ObjectId — admin userId who initiated the suspension
lastLoginAt      Date    — updated on each successful login (used for activity display)
```

**Frontend (`AdminUsersPanel`):**

- Filter bar: role selector, status selector (active / suspended / deleted), free-text search.
- Paginated table with action buttons: Suspend, Unsuspend, Delete.
- Each destructive action opens a confirmation modal; suspension prompts for a reason string.

---

## Admin audit-log dashboard

Gated by `FEATURE_ADMIN_AUDIT_DASHBOARD=true`. Adds a read-only **Audit Log** tab to the admin panel
that surfaces five audit collections:

| Audit type            | Source collection          | What it shows                                                   |
| --------------------- | -------------------------- | --------------------------------------------------------------- |
| Seller lifecycle      | `SellerLifecycleAudit`     | KYC decisions, closures, status transitions with actor + reason |
| Seller profile access | `SellerProfileAccessAudit` | Admin views of tier-1 seller PII (govId, birthDate, banking)    |
| COD refund access     | `CodRefundAccessAudit`     | Admin decryption of sensitive COD refund bank details           |
| User deletion         | `UserDeletionAudit`        | Self-service and admin-initiated soft/hard-delete events        |
| Document access       | `DocumentAccessAudit`      | KYC document view and download events                           |

**Backend endpoints (`/api/admin/audit/*`):**

```
GET /api/admin/audit/seller-lifecycle    — date range + pagination + whitelisted filters
GET /api/admin/audit/seller-profile      — date range + pagination
GET /api/admin/audit/cod-refund          — date range + pagination
GET /api/admin/audit/user-deletion       — date range + pagination
GET /api/admin/audit/document-access     — date range + pagination
```

All endpoints are read-only (`GET`), admin-only, and reject any query parameters not on the
whitelist to prevent unintended data exposure.

**Frontend table:**

- Columns adapt per audit type (e.g., seller lifecycle shows `sellerId`, `action`, `actor`,
  `reason`; document access shows `documentId`, `tier`, `actorId`).
- Human-readable identifiers (seller business name, user name + email) are batch-resolved server-
  side and returned alongside the raw IDs.
- Date-range picker (from / to) and pagination controls.
- CSV export downloads the currently filtered result set.

---

## GDPR & user data export

**Consent logging**

- `POST /api/users/consent-log` — public, no-auth endpoint (rate-limited 10 req/min per IP). Logs
  the visitor's cookie consent decision. Anonymous visitors are identified by a client-generated
  UUID stored in localStorage (`eshop_anon_id`). IP addresses are hashed (SHA-256 truncated to 16
  chars) before storage — the raw IP is never persisted. `ConsentLog` fields: `anonymousId`,
  `userId` (nullable), `action` (`accepted` / `declined` / `withdrawn`), `ipAddress` (hashed),
  `userAgent`, `version`, `timestamp`. See [Architecture page — Consent Management](#) for the full
  flow. The same consent decision drives **GA4 Consent Mode v2**: `analytics_storage` defaults to
  denied (pushed before `gtag.js` loads in `frontend/src/lib/analytics.js`) and analytics events
  fire only after the visitor grants consent.
- `POST /api/users/me/merge-consent` — auth required. Called on login and signup to merge consent
  records previously associated with the anonymous `eshop_anon_id` into the authenticated user
  record.
- `POST /api/users/me/consent` — auth required. Records consent events for the authenticated user
  (e.g., marketing opt-in, terms acceptance) to the `ConsentLog` collection. Entries are never
  modified, preserving a full immutable audit trail.

**Account deletion (soft-delete lifecycle)**

- `DELETE /api/users/me` — requires re-authentication: local-provider accounts must supply
  `confirmationPassword` in the request body; OAuth accounts skip the password check.
- On success the controller (inside a single MongoDB transaction):
  1. Anonymizes all PII inline on the User document (`name`, `email`, `phone`, addresses, OAuth
     tokens, cart, wishlist, `password`) and sets `deletedAt` + optional `deletionReason`.
  2. Scrubs `shippingAddress`, `billingAddress`, and `phoneNumbers` on all related Orders.
  3. Scrubs unencrypted and encrypted PII fields on the linked Seller record (if any) and sets
     `seller.deletedAt`.
  4. Archives all associated SellerDocuments (`lifecycleStatus: 'archived'`).
  5. Deletes ConsentLog records and the Subscriber record for that email.
  6. Creates an append-only `UserDeletionAudit` entry (`actor: 'self'`, `action: 'soft_delete'`)
     with a `retentionExpiresAt` timestamp.
- After the transaction commits, Cloudinary KYC assets are destroyed best-effort (failures are
  logged; the transaction is not rolled back).
- Purge job (`scripts/purge-expired-soft-deleted-users.js`) — run via cron or manually —
  hard-deletes documents whose `deletedAt` exceeds the retention window
  (`USER_DELETION_RETENTION_DAYS`, default 30). The Mongoose `pre('deleteOne', { document: true })`
  hook writes a `hard_delete` audit record before each removal.

**Data export (Articles 15 & 20)**

- `GET /api/users/me/export` — streams a JSON attachment (`Content-Disposition: attachment`)
  containing:
  - `profile` — all user fields with encrypted PII decrypted via Mongoose getters; password and
    token fields excluded.
  - `cartItems`, `wishlist` — current embedded arrays.
  - `orders` — full order documents including line-items, addresses, and payment references.
  - `seller` — full seller application fields (decrypted); raw IBAN excluded, only
    `payout.bank.last4` included.
  - `sellerDocuments` — document metadata plus 24-hour signed Cloudinary URLs for each KYC file.
  - `sellerBilling`, `ledgerEntries`, `sellerLifecycleAudit`, `documentAccessAudit`.
  - `consentLogs` — full consent history.
  - `subscriber` — newsletter subscription record.
  - `deletionAudit` — `UserDeletionAudit` entries for this user.

**Store closure vs. account deletion**

Store closure and user account deletion are separate, coexisting paths:

|                       | User account deletion                                                      | Seller store closure                                                                        |
| --------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Trigger**           | `DELETE /api/users/me`                                                     | `POST /api/seller/close-store`                                                              |
| **Scope**             | Removes the user record and all linked seller data in a single transaction | Closes the seller/store only; the user account stays active for shopping                    |
| **User account**      | Soft-deleted; purged after `USER_DELETION_RETENTION_DAYS`                  | Untouched — user can continue to log in and shop                                            |
| **PII anonymization** | Immediate (within the deletion transaction)                                | Deferred to Phase 3 (finalizer cron, after the waiting period)                              |
| **Financial records** | Retained per legal obligations                                             | Retained per legal obligations                                                              |
| **Data export**       | Available until deletion completes                                         | Available; a seller with `pending_closure` status can still call `GET /api/users/me/export` |

Personal data is anonymized at Phase 3 of store closure; financial and order records are retained as
required by law. Orders continue to resolve to the anonymized Store record so historical order
history remains consistent.

---

## Security & observability

- Cookies: `HttpOnly`, `SameSite=None` (when cross-site), and domain configured to allow
  cross-subdomain cookies for frontend + API. The `Secure` flag is computed dynamically per request
  via `isSecureCookie(req)`: it is set when `SameSite=None`, when the request is served over HTTPS
  (`req.secure`), or when `NODE_ENV=production`. All cookie option sets are factory functions that
  accept the request object, ensuring correct behavior in staging and preview environments served
  over HTTPS.
- **Session invalidation on credential change:** password reset and `unlinkSocial` immediately clear
  both the access-token and refresh-token cookies in addition to revoking all Redis tokens, fully
  signing out the user on all cookie-bearing clients.
- **CSRF:** `csrf-csrf` (double-submit cookie pattern) replaces the removed `csurf` dependency. The
  `X-Mobile-Client` CSRF bypass has been removed; all clients must go through the standard
  double-submit flow. The `/api/csrf-token` endpoint issues the CSRF cookie.
- **Rate limiting** (tiered):
  - Global **GET / HEAD**: 1 000 requests per 15 min — applied to all roles, including admin.
  - Global **mutating** (POST / PUT / PATCH / DELETE): 100 requests per 15 min.
  - Per-route overrides: COD checkout — 10 req / 15 min; GDPR export — 5 req / 60 min.
- Password hashing uses bcrypt with a cost factor of **12** (up from 10). Existing hashes are
  transparently rehashed at login time.
- Sanitization and NoSQL injection protection for request bodies and query params.
- **Logging:** Winston with daily rotation; `redactSensitive()` is applied at the top-level logger
  format and covers **all transports** (Console and DailyRotateFile). The sensitive-key list covers
  25 fields (up from 7) including common PII, token, and credential field names. Redaction traverses
  full error objects including axios error shapes (removing `Authorization` headers and cookie
  values from error metadata). The admin audit middleware logs only a whitelist of safe body keys —
  raw request bodies are never written to audit logs. All raw `console.*` calls in production paths
  have been replaced with the Winston logger; ESLint enforces `no-console` for backend code
  (allowing `warn`/`error` only).
- **Metrics:** Prometheus-compatible scrape at `GET /api/monitoring/metrics` requires a
  `Bearer <MONITORING_METRICS_TOKEN>` header in production. The endpoint returns `404 Not Found` if
  `MONITORING_METRICS_TOKEN` is not configured, preventing accidental metric exposure.
  Datadog-friendly `datadog.metric` log events are still emitted alongside the scrape endpoint. For
  the token rotation strategy and future multi-scraper support, see
  [`docs/security/monitoring-metrics-access.md`](docs/security/monitoring-metrics-access.md).
- **Mobile SSL pinning:** `react-native-ssl-pinning` is fail-closed in production — if the native
  module is missing and `EXPO_PUBLIC_SSL_PINNING_CERTS` is set, the app throws rather than falling
  back to an unpinned connection.
- **Job-health alerting:** settlement and payout cron job health alerts fan out via email (to the
  admin recipient list) and optionally to a Slack webhook (`OPS_SLACK_WEBHOOK`).

**Error taxonomy and handling**

```mermaid
graph TD
    ERR["Runtime Error"] --> OE["Operational Error"]
    ERR --> PE["Programmer Error"]
    OE --> R["Retryable\n429 / 503 / timeout"]
    OE --> NR["Non-Retryable\n400 / 401 / 403 / 404 / 422"]
    PE --> SN["Sentry capture + alert"]
    R --> BK["Exponential backoff\nbackground jobs"]
    NR --> CR["Structured JSON response\nto client"]
```

**Tracing one financial event from Stripe id to audit trail**

```mermaid
graph LR
    A["Stripe event.id"] --> B["[Webhook] log<br/>+ Redis NX key"]
    B --> C["[refund-ledger:webhook] log<br/>orderId, refundId, amountCents,<br/>APP_VERSION, GIT_SHA, pid"]
    C --> D["LedgerEntry.actor + idempotencyKey<br/>(queryable audit trail)"]
    D --> E["Sentry (scrubbed) on error"]
```

**Security middleware chain**

```mermaid
graph TD
    REQ["Incoming Request"] --> HLM["Helmet + CSP"]
    HLM --> COR["CORS origin whitelist"]
    COR --> RL["Rate Limiter\n1000 GET / 100 POST per 15 min"]
    RL --> SAN["mongo-sanitize + HPP"]
    SAN --> CSR["CSRF double-submit token"]
    CSR --> JWT["JWT httpOnly cookie auth"]
    JWT --> APP["Application Logic"]
```

### Field-level PII encryption

Sensitive personal data is encrypted at rest using AES-256-GCM via
`backend/src/utils/fieldEncryption.js`. The server **throws at startup** if `FIELD_ENCRYPTION_KEY`
is absent in production.

Encrypted fields:

```
Seller model
  application.firstName, .middleName, .lastName
  application.govIdNumber, .birthDate, .taxId
  application.identityProof
  payout.bank.iban

User model
  phone
  shippingAddress (embedded object)
  billingAddress  (embedded object)
```

Multi-key rotation is supported via `FIELD_ENCRYPTION_KEYS` (comma-separated list of historical
keys). New writes always use `FIELD_ENCRYPTION_KEY`; decryption attempts each key in the rotation
list until one succeeds. After a rotation window, remove the old key from `FIELD_ENCRYPTION_KEYS`.

**Decryption resilience & operations:**

- `decryptField` returns `null` instead of throwing when a value cannot be decrypted with any
  configured key (e.g. mid-rotation or after a key mismatch), so a bad key degrades gracefully
  instead of crashing the request with a 500.
- On startup the server logs the fingerprints of all configured keys (primary + rotation) in every
  environment, so operators can confirm which keys are loaded without exposing key material.
- The `npm run reencrypt:pii` script (run from `backend/`) re-encrypts stored PII with the current
  primary key; it is a dry-run by default and persists changes only with `--apply`.

### Role & Permission Matrix

```
Role      | Canonical source            | Route-level check                   | Notes
----------|-----------------------------|------------------------------------|------------------------------------------------------
admin     | User.role                   | restrictTo('admin')                 | Full platform access
staff     | User.role                   | restrictTo('admin', 'staff')        | TODO: scope down — currently has admin-level access
                                                                              | on all routes where both roles are listed
seller    | Seller.status + kyc.status  | requireVerifiedSeller middleware     | User.role = 'seller' is a convenience label only;
          |                             |                                      | it is NOT used as the sole authorization gate on any
          |                             |                                      | endpoint. Canonical check: seller.status === 'active'
          |                             |                                      | && seller.kyc.status === 'verified'
customer  | User.role (default)         | protectRoute only                   | Standard authenticated user
```

**System sellers** (platform admin store, shipping liability) use special `ownerUserId` references
and must never be modified through the standard seller application flow.

---

## Technical SEO

### Crawl control (robots.txt)

`frontend/public/robots.txt` defines the crawl policy for all search engines:

- **Allowed:** All public routes — homepage, product pages, category pages, store pages, deals, new
  releases, best sellers, search, about, contact, policies.
- **Disallowed:** Authentication routes (`/login`, `/signup`, `/oauth-signup`, `/verify-email`,
  `/forgot-password`, `/reset-password`), user-gated pages (`/checkout`, `/checkout-success`,
  `/purchase-cancel`, `/orders`, `/profile`, `/cart`, `/wishlist`), admin and seller dashboards
  (`/secret-dashboard`, `/seller/dashboard`, `/seller/apply`, `/seller/pre-apply`), disputes,
  notifications, marketing campaigns, mobile bridge pages, and unsubscribe pages.
- **Sitemap declaration:** `Sitemap: https://www.vexflare.com/sitemap.xml`

The file is served as a static asset from the Vercel frontend deployment. Vercel's SPA rewrite
explicitly excludes `/robots.txt` from being rewritten to `index.html`.

### Dynamic sitemap

The sitemap is generated dynamically by the backend API and served at `GET /api/sitemap.xml`. A
Vercel rewrite proxies `https://www.vexflare.com/sitemap.xml` to
`https://api.vexflare.com/api/sitemap.xml`, keeping the public URL at the canonical path.

**What the sitemap includes:**

| URL type     | Pattern                  | Source query                                                                                 | Priority | Changefreq   |
| ------------ | ------------------------ | -------------------------------------------------------------------------------------------- | -------- | ------------ |
| Static pages | `/`, `/deals`, `/about`… | Hardcoded list (15 pages)                                                                    | 0.2–1.0  | daily–yearly |
| Products     | `/product/:_id`          | `Product.find({ visibility: 'visible', approvalStatus: 'approved', hidden: { $ne: true } })` | 0.9      | weekly       |
| Stores       | `/stores/:slug`          | `Store.find({ status: 'active' })`                                                           | 0.8      | weekly       |
| Categories   | `/category/:slug`        | `Category.find({ isActive: true })`                                                          | 0.7      | weekly       |

**Implementation details:**

- **Service:** `backend/src/services/sitemap.service.js` — `generateSitemap()` runs three parallel
  MongoDB queries (`.select()` + `.lean()` for minimal memory), builds the XML via template
  literals, and caches the result.
- **Route:** `backend/src/routes/sitemap.route.js` — `GET /sitemap.xml` handler. Returns
  `Content-Type: application/xml; charset=utf-8` with
  `Cache-Control: public, max-age=3600, s-maxage=3600`.
- **Caching:** Redis key `sitemap:xml` with a **1-hour TTL** (`cacheGetJSON` / `cacheSetJSON` from
  `lib/cache/cache.js`). On cache miss, the service regenerates from the database. On Redis failure,
  it falls through to fresh generation without crashing.
- **Base URL:** Reads `CLIENT_URL` env var (defaults to `https://www.vexflare.com`), trailing slash
  stripped.
- **XML escaping:** An `escapeXml()` helper handles `&`, `<`, `>`, `"`, `'` in slugs.
- **`lastmod`:** Uses `updatedAt` from each document, formatted as `YYYY-MM-DD`.

**Vercel rewrite** (`frontend/vercel.json`):

```json
{ "source": "/sitemap.xml", "destination": "https://api.vexflare.com/api/sitemap.xml" }
```

The previous static `frontend/public/sitemap.xml` (15 hardcoded URLs, no products/stores/categories)
has been removed.

### Structured data (JSON-LD)

Structured data is rendered as `<script type="application/ld+json">` tags via a `JsonLd` React
component in the document head.

| Schema         | Page          | Component                                 | Key properties                                                          |
| -------------- | ------------- | ----------------------------------------- | ----------------------------------------------------------------------- |
| Organization   | Homepage      | `frontend/src/pages/HomePage.jsx`         | name, url, logo, contactPoint (customer service)                        |
| OnlineStore    | Homepage      | `frontend/src/pages/HomePage.jsx`         | name, url, currenciesAccepted (USD), paymentAccepted, SearchAction      |
| Product        | Product page  | `frontend/src/components/ProductPage.jsx` | name, description, image[], url, offers (price, currency, availability) |
| BreadcrumbList | Product page  | `frontend/src/components/ProductPage.jsx` | Home → Category → Product (position-tracked)                            |
| BreadcrumbList | Category page | `frontend/src/pages/CategoryPage.jsx`     | Home → ancestor categories → current category                           |

Product availability maps to `https://schema.org/InStock` or `https://schema.org/OutOfStock` based
on stock status. The SearchAction target is
`https://www.vexflare.com/search?q={search_term_string}`.

### Canonical URLs & hreflang

Implemented in `frontend/src/hooks/useDocumentHead.js`, applied on every page:

- **Canonical:** `<link rel="canonical" href="https://www.vexflare.com{pathname}">` — all pages
  point to the `www` domain, preventing duplicate content across bare domain and subdomains.
- **Hreflang:** Four `<link rel="alternate" hreflang="...">` tags for `en`, `ar`, `es`, `fr`, plus
  an `x-default` pointing to the base canonical URL. Format:
  `https://www.vexflare.com{pathname}?lang={locale}`.
- **Open Graph:** `og:title`, `og:description`, `og:url`, `og:image` (defaults to
  `https://www.vexflare.com/branding/og-image.png`).
- **Twitter Card:** `summary_large_image` format with title, description, and image.
- **Robots meta:** Default `index, follow` on all pages; pages can opt into `noindex, follow` via
  the `noindex` parameter.

### Page indexing coverage

| Route pattern                                 | Indexed | Notes                                   |
| --------------------------------------------- | ------- | --------------------------------------- |
| `/`                                           | Yes     | Homepage                                |
| `/deals`, `/best-sellers`, `/new`             | Yes     | Collection pages                        |
| `/search`                                     | No      | `noindex` — dynamic query results       |
| `/product/:id`                                | Yes     | In sitemap with `lastmod`               |
| `/stores/:slug`                               | Yes     | In sitemap with `lastmod`               |
| `/category/:slug`                             | Yes     | In sitemap with `lastmod`               |
| `/about`, `/contact`                          | Yes     | Static informational                    |
| `/services`, `/case-studies`, `/architecture` | Yes     | Feature-flagged but included in sitemap |
| `/policies/*`                                 | Yes     | Privacy, terms, shipping, refund        |
| `/support/delete-account`                     | Yes     | Legal compliance page                   |
| `/login`, `/signup`, `/oauth-signup`          | No      | Disallowed in robots.txt                |
| `/checkout`, `/orders`, `/profile`            | No      | Disallowed in robots.txt                |
| `/cart`, `/wishlist`                          | No      | Disallowed in robots.txt                |
| `/seller/*`, `/secret-dashboard`              | No      | Disallowed in robots.txt                |
| `/disputes`, `/notifications`                 | No      | Disallowed in robots.txt                |

### SEO roadmap (Stage 4)

Remaining work for future iterations:

- **FAQ schema** — add `FAQPage` JSON-LD on product pages (from seller-provided Q&A).
- **Review schema** — add `AggregateRating` and `Review` structured data on product pages (data
  already exists in the review model).
- **Image alt audit** — verify all product images, category banners, and store logos have
  descriptive `alt` text.
- **Core Web Vitals** — LCP, CLS, INP measurement and optimization pass.
- **Sitemap index** — refactor to a sitemap index with sub-sitemaps if product count exceeds 40K.
- **Open Graph per product** — set `og:image` to the product's primary image on product pages
  (currently uses default branding image).

---

## Testing & CI

- Tests: Jest unit & integration tests with mocks for Redis / email / cloudinary, database-backed
  tests using **mongodb-memory-server** (in-memory MongoDB) for model/service/integration specs,
  **Supertest** for API/route-level tests, and **Cypress** for frontend E2E flows.
- Scale: **3,333 automated tests across 235 suites** (backend, all passing), plus 340 frontend tests
  across 47 suites — see [Test status](#test-status).
- Coverage: see [Coverage](#coverage) below.
- CI: the full suite runs on every pull request and push to `main` (GitHub Actions); a failing suite
  blocks merge — see [CI/CD pipeline](#cicd-pipeline).

### Coverage

Backend coverage (unit + integration, `npm run test:coverage`):

| Metric     | Coverage |
| ---------- | -------: |
| Statements |   88.36% |
| Branches   |   74.10% |
| Functions  |   89.08% |
| Lines      |   90.02% |

### Critical workflows covered

The 3,333 backend tests provide unit, integration, and E2E coverage for the platform's core business
workflows:

- **Auth & session lifecycle** — login, registration, OAuth bridge codes, refresh-token rotation,
  the sliding 30-day session window, email-based OAuth auto-link (Google + Facebook), and the web
  checkAuth single-flight lock (`auth.controller.spec.js`, `auth.route.spec.js`, `auth.e2e.spec.js`,
  `bridgeCode.*`, `oauthLoginLink.spec.js`, `mobileDeepLink.spec.js`,
  `useUserStore.checkAuth.test.js`).
- **Cart, checkout & stock reservation** — atomic stock decrement/restore on cancel/expire and
  Stripe Checkout session creation/failure paths (`cart.controller.spec.js`, `cart.e2e.spec.js`,
  `checkout.e2e.spec.js`, `checkout-failure.e2e.spec.js`, plus frontend `CartPage.test.jsx` /
  `CheckoutPage.test.jsx`).
- **Stripe payments, webhooks & disputes** — PaymentIntent verification, idempotent webhook replay,
  refunds, and `charge.dispute.*` chargeback ingestion (`payment.controller.spec.js`,
  `payment.webhook.controller.spec.js`, `payment.webhook.disputes.spec.js`,
  `stripeDispute.webhook.spec.js`).
- **Cash-on-delivery (COD)** — eligibility scoring, region-aware availability, abuse/risk controls,
  refund workflow, and the encrypted-details purge cron (`codEligibility.service.spec.js`,
  `payment.controller.cod-risk.spec.js`, `payment.codAbuse.spec.js`,
  `codRefundWorkflow.service.spec.js`, `codRefundFlow.integration.spec.js`).
- **Order lifecycle, returns & refunds** — status transition enforcement, return requests, and
  prepaid/COD refund approval (`order.controller.spec.js`,
  `order.controller.returnRequestGaps.spec.js`, `prepaidRefund.service.spec.js`,
  `admin.approve-refund-prepaid.spec.js`, `seller.orders.controller.refunds.spec.js`).
- **Seller onboarding, KYC & store closure** — application submission, document upload/review,
  Stripe Connect onboarding, and the closure/eligibility/finalization flow
  (`seller.apply.route.spec.js`, `seller.kyc.*.spec.js`, `connectOnboarding.service.spec.js`,
  `seller.closure.controller.spec.js`, `sellerClosure.e2e.spec.js`).
- **Settlements, payouts & ledger** — settlement scheduling/execution/reversal, manual payouts, and
  ledger/liability reconciliation (`settlementExecution.service.spec.js`,
  `settlementScheduler.service.spec.js`, `settlementBatchReverse.service.spec.js`,
  `manualSettlementPayout.service.spec.js`, `ledger.service.spec.js`,
  `liabilityReconciliation.service.spec.js`).
- **Platform & seller financials / billing** — admin platform financial reporting, per-seller
  financial summaries, and subscription billing idempotency/pagination
  (`admin.platformFinancials.controller.spec.js`, `sellerFinancialSummary.service.spec.js`,
  `sellerBilling.service.spec.js`, `billing.history.controller.spec.js`).
- **Shipping & carrier integration** — shipment creation/labeling across Aramex, Shippo, and manual
  providers, plus customer tracking and delivery-region resolution (`shipping/dispatcher.spec.js`,
  `shipping/providers/*.spec.js`, `seller.shipments.controller.spec.js`,
  `order.shipmentTracking.controller.spec.js`, `shipping.e2e.spec.js`).
- **Reviews & dispute resolution** — purchase-gated reviews with rating aggregation, and the 7-state
  dispute lifecycle with SLA enforcement and chargeback linkage (`review.*.spec.js`,
  `ratingAggregate.service.spec.js`, `dispute.*.spec.js`, `dispute/stateMachine.spec.js`, frontend
  `DisputeListPage.test.jsx` / `DisputeDetailPage.test.jsx`).
- **GDPR data export & account deletion** — full user export bundle assembly and account
  deletion/anonymization with seller cascade (`gdpr/assembleUserExport.spec.js`,
  `users/deleteUserAccount.service.spec.js`).
- **Admin moderation, audit & user management** — product moderation, the audit-log dashboard, and
  admin user suspension/deletion (`admin.moderation.products.controller.spec.js`,
  `admin.audit.controller.spec.js`, `admin.users.controller.spec.js`).
- **Tax calculation & remittance** — Avalara provider integration, tax code mapping, and remittance
  reporting (`tax/avalara.provider.spec.js`, `tax/taxRate.service.spec.js`,
  `taxRemittance.service.spec.js`).
- **Product catalog & storefront** — product CRUD/variants, search, category browsing, and store
  pages (`createProduct.spec.js`, `searchProducts.spec.js`, `products.e2e.spec.js`,
  `store.public.e2e.spec.js`, frontend `StorefrontPage.public.integration.test.jsx`).

All paths above are relative to `backend/src/tests/` unless prefixed `frontend/`.

### How to run tests

**Financial chaos/regression scenarios (manual):**

- Simulate duplicate settlement schedule triggers and verify deduped response (`created:false`,
  `reason:'duplicate'`).
- Simulate transient Stripe payout failures and verify retry backoff / escalation to
  `manual_required` when retry budget is exhausted.
- Simulate persistence failure boundaries (batch creation vs payout creation) and verify
  reconciliation invariants remain intact.

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

- **Backend:** 235 suites, 3,333 tests — all passing.
- **Frontend:** 47 suites, 340 tests — all passing.
- **Coverage:** see [Coverage](#coverage) above (90.02% lines / 88.36% statements / 89.08% functions
  / 74.10% branches; unit + integration). Full test suite includes authorization flows, stock
  reservation, Stripe webhooks, and end-to-end checkout scenarios — see
  [Critical workflows covered](#critical-workflows-covered).
- **CI enforcement:** Tests run on every pull request and push to `main` with no
  `continue-on-error`; a failing test suite blocks merge. See
  [`docs/ci-cd-setup.md`](docs/ci-cd-setup.md) for the full pipeline description.

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

### CI/CD pipeline

The repository ships two GitHub Actions workflows:

- **`ci.yml`** — lint, backend tests (with Redis service container), frontend tests, and Vite
  production build. Runs on every push to `main` and every PR targeting `main`.
- **`security.yml`** — Gitleaks secret scanning (full git history) and `npm audit` across all
  workspaces. Runs on every push.

Dependency updates are automated via **Dependabot** (monthly, npm + GitHub Actions). A
**CODEOWNERS** file is included, and `lint-staged` is wired into the pre-commit hook.

The backend also ships a **multi-stage Dockerfile** (`node:20-bookworm-slim`, non-root `appuser`,
`HEALTHCHECK` on `/health`, `npm audit` in the builder stage).

For the full setup guide including branch protection rules and secret scanning configuration, see
[`docs/ci-cd-setup.md`](docs/ci-cd-setup.md).

---

**CI/CD pipeline**

```mermaid
graph LR
    PR["Pull Request"] --> GH["GitHub Actions CI"]
    GH --> L["ESLint + Stylelint"]
    GH --> T["Jest + Supertest\nmongodb-memory-server"]
    L --> MA["Merge to main"]
    T --> MA
    MA --> VD["Vercel Deploy\nfrontend SPA"]
    MA --> RD["Railway Deploy\nbackend API"]
    MA --> EA["EAS Update\nmobile OTA"]
```

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

> The TLS private key and certificate files under `backend/cert/` are Git-ignored and must never be
> committed. See [`cert/README.md`](cert/README.md) for mkcert regeneration instructions and
> [`SECURITY.md`](SECURITY.md) for the credential revocation log and `SESSION_SECRET` rotation
> notes.

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
ACCESS_TOKEN_EXPIRE                # JWT expiry for access tokens (optional; default: 15m)
ACCESS_TOKEN_SECRET
REFRESH_TOKEN_EXPIRE               # Controls JWT expiry, cookie maxAge, and Redis key TTL
                                   # simultaneously — change only this one value to keep all
                                   # three in sync. (optional; default: 30d)
SESSION_SECRET
SESSION_REDIS_PREFIX
REFRESH_TOKEN_SECRET
MAX_SESSION_LIFETIME_DAYS
COOKIE_DOMAIN
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
MONGO_URI                    # MongoDB connection string — REQUIRED IN PRODUCTION.
                              # Note: all references previously using MONGODB_URI have been
                              # standardized to MONGO_URI. Update any existing deployment
                              # configs accordingly.
UPSTASH_REDIS_URL (or REDIS connection string)
```

**Third-party services**

```
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
STRIPE_PRO_PRICE_ID          # optional fallback if FeeConfig has no stripePriceId
RESEND_API_KEY               # active email provider (backend/src/lib/email.js imports Resend)
SENDGRID_API_KEY             # fallback email provider; requires swapping the import in email.js
CLOUDINARY_CLOUD_NAME
CLOUDINARY_API_KEY
CLOUDINARY_API_SECRET
SENTRY_DSN
SENTRY_RELEASE
```

**Security / encryption**

```
FIELD_ENCRYPTION_KEY               # required in production — AES-256-GCM primary key for
                                   # field-level PII encryption (Seller + User models).
                                   # Server throws at startup if absent in NODE_ENV=production.
                                   # Dev default: any non-empty string (e.g. "dev_field_enc_key")
                                   # Production: generate with `openssl rand -hex 32`
FIELD_ENCRYPTION_KEYS              # optional comma-separated rotation list of historical keys
                                   # used for decryption only. New writes always use
                                   # FIELD_ENCRYPTION_KEY. Remove old keys after rotation window.
                                   # Example: FIELD_ENCRYPTION_KEYS=new_key,old_key
```

**Seller / KYC**

```
SELLER_DATA_SECRET                 # encryption key for application.banking at rest
SELLER_DOC_VIEW_TOKEN_SECRET       # signing key for short-lived KYC document view tokens
                                   # (falls back to ACCESS_TOKEN_SECRET if unset)
KYC_DOCUMENT_RETENTION_DAYS        # days before archived seller documents are hard-deleted
                                   # by the purge script (default: 2555 = 7 years)
USER_DELETION_RETENTION_DAYS       # days before soft-deleted users are hard-deleted
                                   # by the purge script (default: 30)
SELLER_CLOSURE_WAITING_DAYS        # waiting period (in days) between closure initiation and
                                   # permanent Phase 3 deletion (default: 90)
                                   # Dev: shorten to 1 or 0 for fast local testing
SELLER_CLOSURE_FINALIZER_CRON      # cron expression for the sellerClosureFinalizer daily job
                                   # (default: "0 2 * * *" — 02:00 UTC)
                                   # Dev: "* * * * *" to trigger every minute for testing
SELLER_CLOSURE_FINALIZER_ENABLED   # set to "false" to disable automatic finalizer scheduling
                                   # (default: "true"); safe to disable during incident response
```

**Shipping / Carrier**

```
SHIPPING_PROVIDER                  # Active carrier adapter: 'disabled' | 'manual' | 'aramex'
                                   #                         | 'shippo'
                                   # 'disabled' (default) returns { ok: false } without throwing
                                   # — safe for local dev. 'manual' accepts caller-supplied
                                   # tracking numbers. 'aramex' calls the Aramex Shipping API.
                                   # 'shippo' calls the Shippo API (labels + tracking).
                                   # REQUIRED IN PRODUCTION (set to 'manual', 'aramex' or 'shippo')

# Aramex credentials — required when SHIPPING_PROVIDER=aramex
ARAMEX_USERNAME                    # Aramex account username
ARAMEX_PASSWORD                    # Aramex account password
ARAMEX_ACCOUNT_NUMBER              # Aramex account number
ARAMEX_ACCOUNT_PIN                 # Aramex account PIN
ARAMEX_ACCOUNT_ENTITY              # Aramex entity code
ARAMEX_ACCOUNT_COUNTRY_CODE        # ISO 3166-1 alpha-2 country code for the Aramex account
ARAMEX_BASE_URL                    # Aramex API base URL
                                   # Sandbox: https://ws.dev.aramex.net
                                   # Production: https://ws.aramex.net
ARAMEX_WEBHOOK_SECRET              # HMAC secret for validating incoming Aramex webhook callbacks
                                   # REQUIRED IN PRODUCTION when SHIPPING_PROVIDER=aramex

# Shippo (Phase 3: carrier-calculated rates + tracking). Single platform-wide account.
SHIPPO_API_KEY                     # Shippo API token. Test tokens start with shippo_test_;
                                   # use a live token in production. REQUIRED for carrier mode
                                   # and live tracking; absent => adapters return shippo_config_missing
                                   # and carrier mode falls back to carrier.fallbackRate.
SHIPPO_ENVIRONMENT                 # 'sandbox' (dev) | 'live' (prod) — informational marker
SHIPPO_API_BASE_URL                # optional, default https://api.goshippo.com
SHIPPO_TIMEOUT_MS                  # optional, default 15000
SHIPPO_MAX_RETRIES                 # optional, default 3
SHIPPO_RETRY_BASE_DELAY_MS         # optional, default 300
SHIPPO_LABEL_FILE_TYPE             # optional, default PDF
SHIPPO_MAX_RATE_ATTEMPTS           # optional, default 6 — label purchase tries up to N carriers,
                                   # skipping ones whose account isn't activated
SHIPPO_TEST_TRACKING_NUMBER        # optional, default SHIPPO_TRANSIT — sandbox-only sample number
SHIPPO_FROM_EMAIL                  # optional — sender email used on labels when the store has none
SHIPPO_FROM_PHONE                  # optional — sender phone used on labels when the store has none
                                   # (a non-empty sender email/phone is required by some carriers,
                                   # e.g. USPS, to purchase a label)

# Phase 3 feature flags (default off; enable per environment after validation)
FEATURE_SHIPPO_RATES               # 'true' => carrier-mode stores get live Shippo rates at checkout;
                                   # off => fall back to carrier.fallbackRate
FEATURE_SHIPPO_TRACKING            # 'true' => customer tracking endpoint does a live carrier sync;
                                   # off => serves cached shipment events only

# Accounting / Liability reconciliation (all default off)
FEATURE_SHIPPING_LABEL_LEDGER      # 'true' => post a shipping_label_cost ledger debit against the
                                   # Shipping Liability seller when a Shippo label is purchased.
                                   # Off by default until backfill of historical shipments is decided.
FEATURE_TAX_REMITTANCE             # 'true' => enable TaxRemittance model + admin
                                   # GET/POST /api/admin/platform-financials/tax-remittances.
FEATURE_LIABILITY_RECONCILIATION   # 'true' => start the monthly liability reconciliation cron job.
FEATURE_SELLER_SHIPPING_PAYOUT     # 'true' => flat/tiered shipping posts a payout-eligible
                                   # shipping_revenue credit to the seller (shipping_revenue_refund
                                   # on refund) instead of the shipping_fee/Shipping Liability pair.
                                   # Carrier-mode shipping is unaffected. Forward-only.
FEATURE_TAX_LIABILITY_SELLER       # 'true' => operator-side tax/tax_refund/tax_remittance mirrors
                                   # post against TAX_SELLER_ID instead of PLATFORM_SELLER_ID.
                                   # Forward-only; see docs/tax-book.md.
TAX_SELLER_ID                      # MongoDB ObjectId of the Tax-Liability synthetic seller (slug
                                   # tax-liability), derived at boot by ensureAdminStore — leave
                                   # blank; do not hand-set. Used when
                                   # FEATURE_TAX_LIABILITY_SELLER=true.
FEATURE_REGION_SHIPPING_MODE       # 'true' => checkout rejects COD (cod_not_available_carrier)
                                   # when the destination's effective shipping.regions[].mode is
                                   # 'carrier'. The per-region mode schema/resolver/seller UI are
                                   # additive and always-on; only the COD rejection is gated.

# Reconciliation job tuning (only read when FEATURE_LIABILITY_RECONCILIATION=true)

Production

LIABILITY_RECONCILIATION_CRON      # cron expression; default '30 3 1 * *' (03:30 UTC 1st of month)
LIABILITY_RECONCILIATION_TIMEZONE  # default 'UTC'
LIABILITY_RECONCILIATION_RUN_ON_BOOT # default 'false'; set true only on single-instance deploys
LIABILITY_RECONCILIATION_PERIOD    # override period, e.g. '2026-05'; defaults to previous month

Development / local testing

LIABILITY_RECONCILIATION_CRON=*/2 * * * *  # frequent cadence so you can see a snapshot get created/updated quickly
LIABILITY_RECONCILIATION_TIMEZONE=UTC      # keep UTC for deterministic period math while testing
LIABILITY_RECONCILIATION_RUN_ON_BOOT=true  # trigger an immediate run on server start for fast feedback (single-instance dev)
LIABILITY_RECONCILIATION_PERIOD=2026-05    # pin to whatever month you've seeded ledger entries for; previousMonthPeriod() won't match seeded test data otherwise — remove this once you're done testing so it doesn't get left set

```

**Payment provider + COD refund/payout policy**

```env
# PAYMENT_PROVIDER is reserved for future multi-gateway support and is not currently read by
# the application. Stripe is always the active payment gateway.
# PAYMENT_PROVIDER=stripe
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
# CHIMONEY_SOURCE_CURRENCY is reserved for future use and not currently read by the application.
COD_REFUND_CHIMONEY_WEBHOOK_STORE_RAW_DEBUG=false
```

Development/testing:

```env
COD_REFUND_PURGE_CRON=* * * * *
COD_REFUND_PURGE_TZ=UTC
CHIMONEY_TIMEOUT_MS=8000
CHIMONEY_MAX_RETRIES=1
CHIMONEY_RETRY_BASE_DELAY_MS=200
# CHIMONEY_SOURCE_CURRENCY is reserved for future use and not currently read by the application.
COD_REFUND_CHIMONEY_WEBHOOK_STORE_RAW_DEBUG=true
```

**Production vs development guidance for the keys above**

- **Production**
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
SUBSCRIPTION_RENEWAL_RUN_ON_BOOT     # default false; set true only on single-instance deploys

PAYMENT_GRACE_ENABLED                # default true; set false to skip grace period on card failure
PAYMENT_GRACE_PERIOD_DAYS            # default 7; days the seller has to fix payment before downgrade
```

**Dispute SLA escalation**

```
FEATURE_DISPUTES_V1                      # set to "true" to enable all dispute routes and UI
                                          # (default: unset / disabled)
DISPUTE_SLA_ESCALATION_ENABLED           # set to "false" to disable the SLA escalation cron
                                          # (default: "true")
DISPUTE_SLA_ESCALATION_CRON             # cron expression for the SLA escalation job
                                          # (default: "0 8 * * *" — 08:00 UTC daily)
                                          # Dev: "* * * * *" to trigger every minute for testing
```

**In-app notifications and audit log**

```
FEATURE_IN_APP_NOTIFICATIONS             # set to "true" to enable /api/notifications/* routes,
                                          # the bell UI (web + mobile), and 60-second polling.
                                          # (default: unset / disabled)

FEATURE_ADMIN_AUDIT_DASHBOARD            # set to "true" to enable /api/admin/audit/* endpoints
                                          # and the Audit Log tab in the admin panel.
                                          # (default: unset / disabled)
```

**Cron job boot-kickoff and ops alerting**

```
# Boot-kickoff flags — all default false (cron-trigger only; safe for multi-replica)
SETTLEMENT_RECONCILIATION_RUN_ON_BOOT    # run settlement reconciliation on server start
PAYOUT_RETRY_RUN_ON_BOOT                 # run payout retry job on server start
COD_PAYOUT_RECONCILIATION_RUN_ON_BOOT   # run COD payout reconciliation on server start
SUBSCRIPTION_RENEWAL_RUN_ON_BOOT         # run subscription renewal on server start
SELLER_CLOSURE_FINALIZER_RUN_ON_BOOT     # run seller closure finalizer once on server start
                                          # (default false; set true only on single-instance deploys)
LIABILITY_RECONCILIATION_RUN_ON_BOOT     # run liability reconciliation job on server start
                                          # (default false; requires FEATURE_LIABILITY_RECONCILIATION=true)

# Ops alerting
OPS_SLACK_WEBHOOK                        # optional Slack incoming webhook URL for job-health alerts
                                         # (fans out alongside the email recipient list)
SETTLEMENT_RECONCILIATION_MAX_REPORT_RECIPIENTS  # cap on email recipients for reconciliation reports
                                                   # (default 5)
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

# OAuth signup bridge code — REQUIRED IN PRODUCTION
OAUTH_BRIDGE_SECRET          # HMAC signing key for one-shot bridge codes used in OAuth signup.
                              # Replaces the previous pattern of passing providerId/email/name in
                              # URL query parameters. Generate with `openssl rand -hex 32`.
                              # Server throws at startup if absent in NODE_ENV=production.
OAUTH_BRIDGE_TTL_SECONDS     # Bridge code expiry window (default 300 seconds / 5 minutes).
                              # Codes are single-use and expire after this window.
```

**Application settings**

```
PORT
NODE_ENV
TRUST_PROXY              # Express trust proxy hop count. Set to 1 when behind a single reverse
                         # proxy (Railway, Vercel, nginx). Controls X-Forwarded-For trust depth.
                         # Dev default: false (or 0)  Production recommended: 1
RATE_WINDOW_MS           # shared limiter window in ms (default 3600000)
SELLER_REQ_LIMIT         # legacy seller write cap alias (fallback)
SELLER_READ_LIMIT        # seller GET/HEAD budget (default 1000/hr)
SELLER_WRITE_LIMIT       # seller POST/PATCH/DELETE budget (default 1000/hr)
ADMIN_REQ_LIMIT          # legacy admin write cap alias (fallback)
ADMIN_READ_LIMIT         # admin GET/HEAD budget (default 10000/hr)
ADMIN_WRITE_LIMIT        # admin POST/PATCH/DELETE budget (default 10000/hr)
MONITORING_METRICS_TOKEN # Bearer token required to access GET /api/monitoring/metrics in
                         # production. If unset, the endpoint returns 404. Generate with
                         # `openssl rand -hex 32`. Not required in development.
MOBILE_DASHBOARD_BRIDGE_SECRET # Required signing key for the mobile seller dashboard bridge
                               # JWT. No fallback to ACCESS_TOKEN_SECRET — server throws at
                               # startup if absent in production.
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
# Controls JWT expiry, cookie maxAge, and Redis key TTL — change only this one value.
REFRESH_TOKEN_EXPIRE=30d
MAX_SESSION_LIFETIME_DAYS=30
JWT_SECRET=changeme_jwt_secret
DEBUG_REFRESH=false

# Mobile seller dashboard bridge (required in production)
MOBILE_DASHBOARD_BRIDGE_SECRET=changeme_bridge_secret

# Field-level PII encryption (required in production)
FIELD_ENCRYPTION_KEY=dev_field_enc_key_32bytes_________
FIELD_ENCRYPTION_KEYS=dev_field_enc_key_32bytes_________

# Monitoring metrics endpoint protection
# Leave empty in dev; set to a random token in production
MONITORING_METRICS_TOKEN=

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

# Payment grace period (card-failure recovery window)
PAYMENT_GRACE_ENABLED=true
PAYMENT_GRACE_PERIOD_DAYS=7

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

# Email Service (Resend — active provider; SendGrid available as fallback)
# Set RESEND_API_KEY to use Resend (the default). To use SendGrid instead, swap the
# import in backend/src/lib/email.js and set SENDGRID_API_KEY.
# From addresses MUST be on a Resend-verified domain — use the verified subdomain
# (email.vexflare.com), not the apex (vexflare.com), or Resend rejects every send.
RESEND_API_KEY=
RESEND_VERIFIED_DOMAINS=email.vexflare.com
RESEND_FROM_EMAIL=orders@email.vexflare.com
RESEND_FROM_NAME="Vexflare"
SENDGRID_API_KEY=
SENDGRID_FROM_EMAIL=your-email@example.com
SENDGRID_FROM_NAME="Vexflare"
SUPPORT_EMAIL=support@email.vexflare.com

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

# Disputes
FEATURE_DISPUTES_V1=true
DISPUTE_SLA_ESCALATION_ENABLED=true
DISPUTE_SLA_ESCALATION_CRON=* * * * *

# In-app notifications and audit log
FEATURE_IN_APP_NOTIFICATIONS=true
FEATURE_ADMIN_AUDIT_DASHBOARD=true

# Accounting / Liability reconciliation (all off by default; enable after validating in dev)
FEATURE_SHIPPING_LABEL_LEDGER=false
FEATURE_TAX_REMITTANCE=false
FEATURE_LIABILITY_RECONCILIATION=false

LIABILITY_RECONCILIATION_CRON=*/2 * * * *  # frequent cadence so you can see a snapshot get created/updated quickly
LIABILITY_RECONCILIATION_TIMEZONE=UTC      # keep UTC for deterministic period math while testing
LIABILITY_RECONCILIATION_RUN_ON_BOOT=true  # trigger an immediate run on server start for fast feedback (single-instance dev)
LIABILITY_RECONCILIATION_PERIOD=2026-05    # pin to whatever month you've seeded ledger entries for; previousMonthPeriod() won't match seeded test data otherwise — remove this once you're done testing so it doesn't get left set


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

**Redis responsibilities**

```mermaid
graph LR
    R[("Redis")]
    R --> A["Session Store"]
    R --> B["Auth Token Cache"]
    R --> C["Application Cache"]
    R --> D["Distributed Locks"]
```

**Cache read path (stampede-safe)**

```mermaid
graph TD
    A["Read request"] --> B{"Redis GET hit?"}
    B -->|"hit"| C["Return cached JSON"]
    B -->|"miss"| D["cacheSetNXJSON\nset-if-not-exists"]
    D -->|"first writer"| E["Query MongoDB"]
    E --> F["cacheSetJSON with TTL"]
    F --> G["Return fresh data"]
    D -->|"already written"| H["Short wait + retry GET"]
    H --> B
```

**Stock reservation with distributed lock**

```mermaid
graph TD
    A["checkout: reserve variant"] --> B["cacheAcquireLock(variantId)"]
    B --> C{"lock acquired?"}
    C -->|"yes"| D["read stock from DB"]
    D --> E{"stock > 0?"}
    E -->|"yes"| F["MongoDB transaction\ndecrement stock"]
    F --> G["cacheReleaseLock"]
    G --> H["proceed to payment"]
    E -->|"no"| I["cacheReleaseLock"]
    I --> J["return out-of-stock"]
    C -->|"busy"| K{"retry budget?"}
    K -->|"yes, backoff"| B
    K -->|"exhausted"| L["return 409 Conflict"]
```

### Environment variables

**Cache / driver-based Redis**

- `REDIS_DRIVER=rest|tcp|auto`
- REST:
  - `UPSTASH_REDIS_REST_URL`
  - `UPSTASH_REDIS_REST_TOKEN`
- TCP:
  - `REDIS_URL` (or `UPSTASH_REDIS_URL`) as `rediss://default:<password>@<host>:6379`
- Resilience:
  - `REDIS_OP_TIMEOUT_MS` (default `1500`) — per-operation timeout so a stalled cache endpoint
    cannot block a request.
  - `UPSTASH_MAX_RETRIES` (default `1`) — caps Upstash REST client retries to fail fast instead of
    piling up backoff delays.

**Sessions (TCP-only)**

- `SESSION_REDIS_URL=redis://127.0.0.1:6379` (local) OR `rediss://...` (managed)
- `SESSION_REDIS_PREFIX` (recommended)
  - `session:dev:` for dev
  - `session:prod:` for prod
- `SESSION_REDIS_COMMAND_TIMEOUT_MS` (default `1000`) — per-command timeout for the session client.
  Combined with a disabled offline queue, this ensures a stalled session store fails fast and never
  defers the response flush.

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

# Check TTL for the sliding-window session key
TTL session_active_since:<userId>

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

### Seller Onboarding Flow

![Seller onboarding — Customer account entry](./docs/screenshots/customer_account_page_apply_to_become_a_seller_.png)
_Customer account page entry point to apply as a seller._

![Seller onboarding — Pre-onboarding questionnaire](./docs/screenshots/seller_pre_onboarding_questionnair.png)
_Initial seller pre-onboarding questionnaire screen._

![Seller onboarding — Seller information](./docs/screenshots/seller_onboarding_seller_information_form.png)
_Seller information form (identity and business basics)._

![Seller onboarding — Store info](./docs/screenshots/seller_onboarding_store_info_form.png) _Store
profile and storefront setup form._

![Seller onboarding — Business policy](./docs/screenshots/seller_onboarding_business_policy_form.png)
_Business policy form for shipping/returns and policy details._

![Seller onboarding — Verification](./docs/screenshots/seller_onboarding_verification_form.png) _KYC
verification form for identity and compliance checks._

![Seller onboarding — Banking](./docs/screenshots/seller_onboarding_banking_form.png) _Banking
details form used for payout setup._

![Seller onboarding — Admin received application](./docs/screenshots/admin_dashboard_seller_application_received.png)
_Admin dashboard view showing incoming seller applications._

![Seller onboarding — Admin application review](./docs/screenshots/admin_dashboard_seller_application_expanded.png)
_Admin expanded view for reviewing seller application details._

![Seller onboarding — Admin document review](./docs/screenshots/admin_dashboard_admin_expand_to_view_seller_application_uploaded_documents.png)
_Admin expanded document viewer for uploaded seller verification documents._

---

### Seller Dashboard

![Seller dashboard — Homepage (seller logged in)](./docs/screenshots/homepage_seller_LoggedIN.png)
_Storefront homepage viewed while logged in as an approved seller (Seller link visible in the
navbar)._

![Seller dashboard — Orders panel](./docs/screenshots/seller_orders_panel.png) _Seller orders panel
listing orders that include the seller's products, with status._

![Seller dashboard — Order filters](./docs/screenshots/seller_ordrs_filters_panel.png) _Order
filters panel — search by order ID, address & date, or email, and filter by status._

![Seller dashboard — Order fulfilment panel](./docs/screenshots/seller_dashboard_oder_fulfilment_and_orderfilter_panel.png)
_Seller order fulfilment panel with filters and status controls._

![Seller dashboard — Product creation](./docs/screenshots/seller_dashboard_create_product_form.png)
_Seller create product form with multilingual name/description fields, tax category, shipping
dimensions, and variant support._

![Seller dashboard — Products tab](./docs/screenshots/seller_products_tab.png) _Products tab —
catalog grouped by category, with bulk deal adjustment and review status._

![Seller dashboard — Financials panel](./docs/screenshots/seller_financials_panel.png) _Seller
financials panel showing the current balance breakdown (sales, commission, fees, refunds) and
upcoming/recent payouts._

![Seller dashboard — Financials tab](./docs/screenshots/seller_dashboard_financials_tab.png)
_Financials transaction ledger with per-order shipping income, commission, fees, and tax line
items._

![Seller dashboard — Financial deductions breakdown](./docs/screenshots/seller_dashboard_financials_panel_deductions_Breakdown.png)
_Detailed deductions breakdown in financials._

![Seller dashboard — Plan summary](./docs/screenshots/seller_dashboard_Plan_summary.png) _Seller
subscription plan summary._

![Seller dashboard — Profile settings](./docs/screenshots/seller_dashboard_profile_settings.png)
_Seller profile settings — store name, logo, banner, description, tagline, and contact details._

![Seller dashboard — Payment method update](./docs/screenshots/seller_dashboard_payment_method_update_form_expanded.png)
_Expanded payment method update form._

![Seller dashboard — Bank update consent](./docs/screenshots/seller_dashboard_bank_update_consent.png)
_Consent dialog shown before submitting a bank account update, explaining the 1-2 business day
review and storefront pause._

![Seller dashboard — Application details](./docs/screenshots/seller_dashboard_application_details_panel_showing_document_tiers_and_redacted_document.png)
_Application details with document tiers and redacted documents._

![Seller dashboard — Shipping settings](./docs/screenshots/seller_shipping_settings.png) _Seller
shipping settings — default shipping rate, radius-based local zone pricing, and per-region rates
with map-based location selection._

![Seller dashboard — Store policy settings](./docs/screenshots/seller_store_policy_settings.png)
_Seller store policy settings — shipping and returns/refunds policy editors._

![Seller dashboard — Store settings (pause & QR share)](./docs/screenshots/seller_dashboard_updated_store_setings_expanded_seller_pause_or_copyQR_Code_to_their_store_for_share.png)
_Store settings expanded — pause store toggle and share-storefront QR code / link._

![Seller dashboard — Product moderation state](./docs/screenshots/seller_newly_created_products_are_draft_needto_submit_for_review.png)
_Newly created products saved as draft and ready for submission for review._

![Seller dashboard — Upgrade flow to Stripe](./docs/screenshots/seller_upgrading_to_pro_plan_routed_toStripe_to_pay.png)
_Upgrade to Pro plan routed to Stripe checkout._

---

### Admin Dashboard

![Admin dashboard — Platform management context](./docs/screenshots/admin_dashboard_have_seller_dashboard_to_manage_main_website_products_and_Dashboard_for_admin_platform_level_management.png)
_Admin dashboard context for platform-level management._

![Admin dashboard — Order tab platform visibility](./docs/screenshots/admin_panel_order_tab_can_see_the_whole_platform_orders.png)
_Admin orders tab can view platform-wide orders._

![Admin dashboard — Approved sellers](./docs/screenshots/admin_dashboard_approved_sellers_panel.png)
_Approved sellers management panel._

![Admin dashboard — Category tree](./docs/screenshots/admin_dashboard_category_tree_panel_where_admin_create_and_edit_categories_for_thewhole_website_and_mobile_app.png)
_Category tree panel for web and mobile taxonomy management._

![Admin dashboard — Commissions](./docs/screenshots/admin_dashboard_commissions_panel.png)
_Commission configuration panel._

![Admin dashboard — Settlements](./docs/screenshots/admin_dashboard_settlements_panel.png)
_Settlements panel._

![Admin dashboard — Marketplace financials](./docs/screenshots/admin_dashboard_marketplace_financials_panel.png)
_Marketplace financials panel._

![Admin dashboard — Adjustments](./docs/screenshots/admin_dashboard_adjustments_panel.png) _Platform
adjustments panel._

![Admin dashboard — Seller adjustments](./docs/screenshots/admin_dashboard_seller_adjustments_panel.png)
_Seller-specific adjustments panel._

![Admin dashboard — Product moderation](./docs/screenshots/admin_Moderation_tab_product_moderation_panel.png)
_Product moderation panel in admin moderation tab._

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

All credentials are scoped to the staging environment (`https://vexflare.com` and the Expo mobile
client). Update or rotate as needed before sharing broadly.

| Role            | Email / Username   | Password  |
| --------------- | ------------------ | --------- |
| Regular shopper | `demo@example.com` | `demo123` |

> **Note:** These are placeholder credentials. Contact the platform administrator for actual demo
> access.

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
- [`cert/README.md`](cert/README.md) — mkcert regeneration instructions for the backend local
  development TLS certificate.
- [`SECURITY.md`](SECURITY.md) — credential revocation log and `SESSION_SECRET` rotation notes.
- [`docs/ci-cd-setup.md`](docs/ci-cd-setup.md) — full CI/CD pipeline documentation including
  workflow jobs, branch protection rules, Dependabot configuration, Dockerfile hardening, and the
  Vercel proxy double-egress observation.
- [`docs/monitoring/alerts.md`](docs/monitoring/alerts.md) — alert definitions, thresholds, and
  on-call runbook for the production monitoring stack.
- [`docs/security/monitoring-metrics-access.md`](docs/security/monitoring-metrics-access.md) —
  `MONITORING_METRICS_TOKEN` rotation strategy and future multi-scraper support plan.
- [`docs/security/webhook-rotation.md`](docs/security/webhook-rotation.md) — rotation runbooks for
  Stripe, Chimoney, and OAuth bridge secrets.
- [`docs/notifications.md`](docs/notifications.md) — in-app notification centre reference: model,
  REST API, bell UI behaviour, and guide for wiring new notification types.
- [`docs/settlements-runbook.md`](docs/settlements-runbook.md) — settlement and payout operations
  runbook including distributed lock keys, `RUN_ON_BOOT` vars, and manual fallback procedures.
- [`docs/tax-book.md`](docs/tax-book.md) — admin tax guide: internal `TaxRule`s and precedence,
  product tax categories, the Avalara integration (credentials, auto-fallback, cross-env retry, tax
  codes), provider health, the fallback chain, and what's persisted on each order.
- [`docs/DAISYUI-THEME.md`](docs/DAISYUI-THEME.md) — shared DaisyUI theming internals for web and
  mobile.
- [`docs/THEME.md`](docs/THEME.md) — native token architecture for both platforms.
- [`docs/deployment.md`](docs/deployment.md) — release runbook covering frontend, backend, and
  mobile rollouts for coordinated multi-person deploys.
- [`docs/security/access-controls.md`](docs/security/access-controls.md) — account provisioning and
  least-privilege role documentation template.
- [`mobile/env.md`](mobile/env.md) — environment matrix, API profiles (emulator/LAN/tunnel/prod),
  and deployment checklist for the Expo app.
- [`mobile/docs/docs/mobile/android-build-config.md`](mobile/docs/docs/mobile/android-build-config.md)
  — native build configuration (gradle.properties, signing files, `eas.json` notes).

---

## Contact / commercial enquiries

For purchase, licensing, deployment assistance, or a demo with admin credentials contact:
**[ahmedmounib2@gmail.com](mailto:ahmedmounib2@gmail.com)**

---
