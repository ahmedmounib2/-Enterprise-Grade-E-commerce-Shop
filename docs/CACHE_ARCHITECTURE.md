# Recommendation caching architecture

This document is the single source of truth for how Vexflare caches **recommendation datasets**
(Featured Products, Best Sellers, New Releases, Today's Deals, a storefront's own carousels, Cart's
"People Also Bought") across Web and Mobile, and — just as importantly — what is **intentionally
never cached**.

## The one rule that never changes

**Product detail pages always fetch fresh.** `GET /products/:id` (web's `/product/:id`, mobile's
`ProductScreen`) is never cached, on either platform. Price, stock, discounts, ratings, and
availability must reflect the database on every single visit. Nothing in this document applies to
that endpoint or its consumers — only to the recommendation lists that surround it.

## Cache-by-cache reference

| Cache                                                                             | Platform | Datasets it backs                                                                                                                               | Keyed by                           | TTL                             |
| --------------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | ------------------------------- |
| `useProductStore.js` (`fetchFeaturedProducts`/`isFeaturedFresh`)                  | Web      | Featured Products (Home, Category, Storefront)                                                                                                  | global (one marketplace-wide list) | 5 min                           |
| `useProductStore.js` (`fetchStatusLists`/`isStatusListsFresh`)                    | Web      | Best Sellers, New Releases, Today's Deals (Home, Category)                                                                                      | global (`GET /products/statuses`)  | 5 min                           |
| `frontend/src/lib/storefrontCache.js`                                             | Web      | A storefront's own product list + its seller-scoped Best/New/Deals carousels (Storefront page, Product page)                                    | store slug                         | 5 min                           |
| `frontend/src/lib/peopleAlsoBoughtCache.js`                                       | Web      | Cart's "People Also Bought"                                                                                                                     | seller + category + cart contents  | 5 min                           |
| `mobile/src/services/featuredApi.js`                                              | Mobile   | Featured Products (Home, Category, Storefront)                                                                                                  | global                             | 5 min                           |
| `mobile/src/services/statusesApi.js`                                              | Mobile   | Best Sellers, New Releases, Today's Deals (Home, Category, Product's related carousels, the dedicated Best/New/Deals screens, `ProductsScreen`) | global (`GET /products/statuses`)  | 5 min                           |
| `mobile/src/services/storeApi.js`                                                 | Mobile   | A storefront's own product list + its seller-scoped Best/New/Deals carousels (Storefront screen, Product screen)                                | store slug                         | 5 min                           |
| `mobile/src/services/peopleAlsoBoughtCache.js`                                    | Mobile   | Cart's "People Also Bought"                                                                                                                     | seller + category + cart contents  | 5 min                           |
| `frontend/src/stores/useCategoryStore.js` (`CATEGORIES_TTL_MS`)                   | Web      | Category tree (navigation, not a recommendation dataset)                                                                                        | tree query options                 | 60 s (unchanged — out of scope) |
| `mobile/src/services/categoryApi.js` (`BROWSE_TTL_MS`)                            | Mobile   | Category browse (product listing within a category — catalog browsing, not a recommendation dataset)                                            | category slug                      | 60 s (unchanged — out of scope) |
| Backend `featured_products` (Redis, via `backend/src/lib/cache/featuredCache.js`) | Backend  | The authoritative source `GET /products/featured` reads through                                                                                 | single global key                  | 5 min                           |

**Why 5 minutes, not 60 seconds:** recommendation flags (`isFeatured`, `status`) change through a
deliberate seller/admin curation action, not continuous drift — there is no scenario where a
customer needs to see a newly-flagged product within seconds. A longer TTL trades a small,
acceptable staleness window for far fewer redundant refetches. This is safe specifically _because_
mutation-triggered invalidation (below) is the primary freshness mechanism — the TTL only matters as
a fallback bound for the (should-never-happen) case where an invalidation call is missed.

Category-tree and category-browse caches were deliberately left at 60 s: they are catalog/browsing
data, not recommendation data, and are out of scope for this document's TTL policy.

## Seeding pattern: why a warm cache never shows a loading frame

Every consumer of these caches seeds its **initial render state synchronously** from the cache,
instead of starting at an empty/loading default and populating asynchronously:

- **Zustand-backed (web `useProductStore.js`)**: the store itself is a module-level singleton, so
  simply reading it in a component (`useProductStore()`) already reflects the current value on the
  very first render — no seeding gymnastics needed. The corresponding `fetchX({force})` action
  checks `isXFresh()` _before_ touching its `loading` flag, so a warm-cache call never toggles
  loading at all.
- **Module-level Map/singleton caches consumed via local component state (web
  `storefrontCache.js`/`peopleAlsoBoughtCache.js`, all of mobile's services)**: the component reads
  `getCachedX()`/`isXFresh()` and passes the result into a `useState(() => ...)` **lazy
  initializer**, so the very first render already shows the cached content with no loading flag set.
- **The one exception — `ProductPage.jsx`'s seller-scoped Best/New/Deals carousels**: the seller
  slug isn't known until the (always-fresh, never-cached) product fetch resolves, so there's no
  "mount time" moment to seed from. Instead, the effect that reads `storefrontCache.js` is a
  `useLayoutEffect` (not `useEffect`) that checks `isStorefrontFresh(sellerSlug)` synchronously: on
  a hit, it calls `setState` synchronously within the layout-effect phase, so React re-renders with
  the cached data _before the browser paints_ — zero loading frame. On a miss, it falls through to a
  real (necessarily async) fetch with a genuine loading state, which is correct: that case is an
  actual cold load.

**The bug this fixes, concretely:** a plain `useEffect` always runs _after_ the browser paints the
current commit. Even when the cache lookup inside it is synchronous and finds a hit, the state
update is still one tick behind — guaranteeing at least one painted frame of "loading" before the
real content appears, even though the data was sitting in memory the whole time. `useLayoutEffect`
(or seeding via a lazy `useState` initializer, when the key is already known at mount) closes that
gap by moving the state update before paint.

Mobile screens follow the same principle: `HomeScreen.js`'s and `CategoryScreen.js`'s
`loadFeatured`/`loadBuckets`/`loadStatuses` functions only set their own `loading*` flag to `true`
when `getCachedX()` returns nothing (or a caller explicitly forces a refresh) — never
unconditionally — so a background reconfirmation of an already-fresh cache never re-hides the
already-correct, already-seeded carousel behind a spinner.

## Mutation invalidation architecture

Two layers exist, matched to what each layer can actually reach:

### Backend (authoritative — reaches every client)

`backend/src/lib/cache/featuredCache.js` exports one function, `safeClearFeaturedCache()`, that
clears the backend's own `featured_products` Redis key (the cache `GET /products/featured` reads
through, 5-minute TTL). It is called — **only** — from mutations that can change which products
qualify for the featured/best-seller/new-release/todays-deal lists:

| Controller                                | Mutation                            | Why it invalidates                                                                                                                                                      |
| ----------------------------------------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `product.controller.js`                   | `createProduct` (POST)              | A new product can be created already `isFeatured`/flagged.                                                                                                              |
| `product.controller.js`                   | `updateProductFields` (PATCH)       | Can change `isFeatured`/`hidden`/`stock`/`status`/`deal`.                                                                                                               |
| `product.controller.js`                   | `editProduct` (PUT)                 | Same fields as above, via a full edit.                                                                                                                                  |
| `product.controller.js`                   | `deleteProduct` (DELETE)            | Only when the deleted product was `isFeatured` — a deleted product can no longer appear anywhere.                                                                       |
| `seller.products.controller.js`           | `updateSellerProductFields` (PATCH) | Mirrors the admin twin: invalidates when `stock`/`status`/`isFeatured`/`hidden` change; **not** on `price`/`deal`/variant-stock, which don't affect the featured query. |
| `admin.moderation.products.controller.js` | `approveProduct` (PUT)              | A newly-approved product becomes visible for the first time.                                                                                                            |
| `admin.moderation.products.controller.js` | `rejectProduct` (PUT)               | A rejected product is hidden — could remove it from a list it was in.                                                                                                   |
| `admin.moderation.products.controller.js` | `requestProductChanges` (PUT)       | Same hiding side effect as reject.                                                                                                                                      |
| `store.controller.js`                     | `updateStoreStatus` (PUT)           | Bulk-hides/unhides every one of a store's products at once — invalidates unconditionally rather than querying every affected product first.                             |

**Deliberately _not_ wired** (audited and confirmed to have no effect on the featured query):

- `reviewStore` — changes `Store.reviewStatus` (a policy-review workflow field). Product eligibility
  (`getPublicEligibleSellerIds`) is based on `Seller.status`/`Seller.kyc.status`, never
  `Store.reviewStatus` — this mutation cannot change what the featured list returns.
- Price and deal (`%`) changes anywhere — `getFeaturedProducts`'s query filters only on
  `isFeatured`/`stock > 0`/seller eligibility, never price or deal.
- Variant-stock updates — the featured query filters on the _top-level_ `stock` field only, which
  variant-stock updates never touch.

**No GET/read handler anywhere in the backend invalidates any cache.** This was audited across every
controller (`grep`-level sweep of every `cacheDel`/`cacheScanDel` call site, cross-checked against
each enclosing route's HTTP method) — every call site sits on a genuine mutation route
(`POST`/`PUT`/`PATCH`/`DELETE`).

There is no backend cache for `GET /products/statuses` (Best Sellers/New Releases/Today's Deals) —
it has always queried MongoDB fresh on every request. The "staleness" for that dataset is entirely a
client-side (browser tab / mobile app) TTL concern; see below.

### Frontend (same-session — reaches only the tab/app that performed the mutation)

A backend mutation cannot reach into a _different_ browser tab's or device's in-memory cache — there
is no push channel (no websockets/SSE) in this architecture, and adding one is out of scope. What
**is** achievable, and implemented: `useProductStore.js`'s own mutation actions
(`createProduct`/`updateProductFields`/`editProduct`/`deleteProduct`) reset
`statusListsFetchedAt: 0` after a successful mutation, in the same `set()` call that already patches
`featuredProducts`. This means that if an admin/seller performs one of these mutations and then
navigates to Home/Category _in the same tab_, the next `fetchStatusLists()` call sees a "stale"
timestamp and refetches for real — instead of serving a copy that's technically still inside its
5-minute TTL window but is known to be wrong.

For a _different_ customer, on a different device/tab, within the TTL window: the backend's own data
is already correct (no backend cache to be stale), but that customer's own client-side cache entry
won't refresh until its TTL naturally expires. This is an accepted, explicit limitation of a
client-side-cache architecture with no real-time push — not a bug to be silently worked around.

Mobile has no product-mutation screens at all (product/seller/admin management is web-only), so
there is no equivalent "same-session" invalidation need on mobile — it is a pure consumer of these
caches.

## Web/Mobile parity

Every recommendation dataset is cached identically on both platforms, with the same TTL and the same
invalidation boundaries:

- Featured Products: `useProductStore.js` (web) / `featuredApi.js` (mobile).
- Best Sellers / New Releases / Today's Deals: `useProductStore.js`'s `fetchStatusLists` (web) /
  `statusesApi.js` (mobile) — both call the same combined `GET /products/statuses` endpoint.
- A storefront's own catalog + seller-scoped status carousels: `storefrontCache.js` (web) /
  `storeApi.js` (mobile).
- Cart's "People Also Bought": `peopleAlsoBoughtCache.js` on both platforms, same same-seller →
  same-category → marketplace-random hierarchy, same cache key shape (seller + category + cart
  contents).

## What is intentionally never cached

- **Product detail pages** (`GET /products/:id`) — see "the one rule that never changes" above.
- **Category tree / category browse** — out of scope for this document; see
  `useCategoryStore.js`/`categoryApi.js`, which keep their own 60 s TTL for navigation/catalog
  freshness, unrelated to recommendation curation.
- **Cart contents, checkout, orders** — always fetched/mutated fresh; nothing about a cart's own
  items is cached (only the _recommendation row shown alongside it_ is).
