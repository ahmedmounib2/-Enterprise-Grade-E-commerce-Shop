# Tax Book — admin & maintainer guide

How tax is calculated, configured, and audited in `ahmedmonib-eshop-demo.com`. The engine lives in
`backend/src/services/tax/`; this guide is the operational reference. For the architecture summary
see the **Category-based tax engine** section of the root `README.md`.

- [Tax Book — admin \& maintainer guide](#tax-book--admin--maintainer-guide)
  - [1. Overview \& the fallback chain](#1-overview--the-fallback-chain)
  - [2. Internal TaxRules](#2-internal-taxrules)
    - [Precedence (how the winning rule is chosen)](#precedence-how-the-winning-rule-is-chosen)
    - [Matching is tolerant (so real-world addresses resolve)](#matching-is-tolerant-so-real-world-addresses-resolve)
  - [3. Product tax categories](#3-product-tax-categories)
  - [4. Avalara integration](#4-avalara-integration)
    - [Getting credentials](#getting-credentials)
    - [Configuration](#configuration)
    - [Auto-fallback](#auto-fallback)
    - [Cross-environment retry (env mismatch self-correction)](#cross-environment-retry-env-mismatch-self-correction)
    - [Rate-limit resilience](#rate-limit-resilience)
  - [5. Avalara tax codes](#5-avalara-tax-codes)
  - [6. TaxProvider health](#6-taxprovider-health)
  - [7. Order tax audit](#7-order-tax-audit)
  - [8. Quick admin recipes](#8-quick-admin-recipes)
  - [9. Ship-from address and Avalara nexus](#9-ship-from-address-and-avalara-nexus)
    - [Why ship-from matters](#why-ship-from-matters)
    - [How sellers configure it](#how-sellers-configure-it)
    - [Activation guard](#activation-guard)
    - [Avalara early-exit guard](#avalara-early-exit-guard)
    - [How tax estimate and checkout both use ship-from](#how-tax-estimate-and-checkout-both-use-ship-from)
    - [`buildTransaction` fallback for missing street](#buildtransaction-fallback-for-missing-street)

---

## 1. Overview & the fallback chain

Tax is computed **per order line** by the line's category and the destination jurisdiction. The
active provider is chosen by the `TAX_PROVIDER` env var (`internal` | `avalara`). Whatever the
provider, the result is the same shape, and tax can **never break checkout** because of a three-step
fallback:

```
Avalara  ──(any failure)──▶  internal TaxRules  ──(no rule matches)──▶  env TAX_RATE
```

- `TAX_PROVIDER=internal` (default) → admin `TaxRule`s, then `TAX_RATE`.
- `TAX_PROVIDER=avalara` → Avalara, and on **any** Avalara failure (missing creds, 4xx/5xx,
  network/timeout, trial expired) it falls back to the internal `TaxRule`s, which themselves fall
  back to `TAX_RATE` (default `0.14`).
- **Cross-border zero-rating (internal engine only):** when the store's ship-from country differs
  from the destination country, the internal engine returns `$0` tax — cross-border sales are
  zero-rated in the internal fallback. Avalara handles cross-border jurisdiction natively when it is
  active.

The cart and checkout show tax informationally via `POST /api/tax/estimate`; the **authoritative**
tax is recomputed server-side at order creation (Stripe and COD flows) and persisted on the order.

---

## 2. Internal TaxRules

Admin-managed rates live in the `TaxRule` collection and are edited in the **Tax Rules** admin panel
(`AdminTaxRulesPanel`) → `GET/POST/PUT/DELETE /api/admin/tax-rules`.

A rule is:

| Field                                     | Meaning                                                                   |
| ----------------------------------------- | ------------------------------------------------------------------------- |
| `jurisdiction.country`                    | ISO-3166 alpha-2 (e.g. `EG`, `US`), or `*` for a **global** default.      |
| `jurisdiction.state`                      | Subdivision code (e.g. `ALEXANDRIA`, `CA`) or `null` for country-level.   |
| `categoryKey`                             | Product tax-category key this rule applies to (e.g. `standard`, `dress`). |
| `rate`                                    | Decimal fraction, e.g. `0.14` = 14 %.                                     |
| `taxExempt`                               | When `true`, the rate is forced to 0 (and the Avalara line uses `NT`).    |
| `avalaraTaxCode`                          | Optional AvaTax code override for this category/jurisdiction (see §5).    |
| `effectiveStartDate` / `effectiveEndDate` | Effective-dated window (defaults: epoch → year 9999).                     |
| `enabled`                                 | Disabled rules are ignored.                                               |

### Precedence (how the winning rule is chosen)

`findTaxRule` scores every enabled, in-window rule with a **single combined** score and picks the
highest:

```
score = categoryExact*10 + jurisdiction
        jurisdiction:  state-match 3  >  country-match 2  >  global '*' 1
        categoryExact: exact category 1 ,  "standard" fallback 0
```

Consequences:

- An **exact category** rule always beats a `standard` rule **within the same jurisdiction**
  (`EG + dress 30%` wins over `EG + standard 50%` for a dress).
- A more specific **jurisdiction** wins within the same category tier (`EG-ALEXANDRIA + t-shirt 10%`
  wins over `EG + standard`).
- `standard` is the **fallback category**: if no rule matches the product's category, the `standard`
  rule for that jurisdiction applies; if no jurisdiction matches at all, the engine uses env
  `TAX_RATE`.

### Matching is tolerant (so real-world addresses resolve)

- **Country** — a typed country name is resolved to its ISO code, so `"Egypt"` matches a rule
  `country: EG`.
- **State** — matched through the shared address normalizer against the delivery-region subdivision
  catalog: administrative suffixes, transliterations and Arabic all fold together, so
  `"Alexandria Governorate"`, `"Al Iskandariyah"` and `"الإسكندرية"` all match a rule state
  `ALEXANDRIA`. When the state field is empty the **city** is used as the fallback.
- **Category key** — normalized before comparison: gender prefixes (`Men's`/`Women's`/`Kids'`) are
  stripped and plural/hyphen differences folded, so a rule keyed `"Men's Tshirts"` matches a product
  key `"t-shirt"`, and `"Dresses"` matches `"dress"`. **Tip:** still prefer the product-slug value
  shown in the rule dropdown — the panel displays the exact key each option will match.

---

## 3. Product tax categories

Every product has a `taxCategory` key (string, default `"standard"`), editable on the seller
**Create/Edit Product** form. The key the engine actually matches against rules is derived per line
by `resolveLineTaxCategory(product)`:

1. An explicit, **non-`standard`** `taxCategory` (a deliberate seller override) — wins.
2. Else the product's `subCategorySlug` (e.g. `dress`, `t-shirt`).
3. Else the product's `categorySlug`.
4. Else `standard`.

So an admin rule keyed `dress` applies to dress products **even if the seller never touched
`taxCategory`** (it defaults to `standard`). The product-category options in the rule dropdown come
from the live category tree (`GET /api/public/tax-categories` → `{ groups: { special, product } }`),
so the keys an admin can pick are the same slugs products resolve to.

---

## 4. Avalara integration

When `TAX_PROVIDER=avalara`, `services/tax/providers/avalara.provider.js` posts a **non-committing**
`SalesOrder` quote to AvaTax `POST /transactions/create` (so no live transaction is recorded), maps
the response back to the internal `perLine` shape, and persists the response metadata on the order.

### Getting credentials

1. Create an Avalara account (sandbox or production) at <https://www.avalara.com>.
2. In the AvaTax console, find your **Account ID** and generate a **License Key** (Settings →
   License and API Keys).
3. Note your **company code** (Settings → Manage Companies). The generic `DEFAULT` returns
   `404 CompanyNotFound` on most accounts — use the account's real code.

### Configuration

```
TAX_PROVIDER=avalara
AVALARA_ACCOUNT_ID=<your account id>
AVALARA_LICENSE_KEY=<your license key>        # secret
AVALARA_ENVIRONMENT=sandbox|production         # MUST match the tier the credentials belong to
AVALARA_COMPANY_CODE=<your real company code>  # default 'DEFAULT'
AVALARA_TIMEOUT_MS=8000                         # request timeout before falling back to internal
```

- `AVALARA_ENVIRONMENT` picks the base URL: `sandbox` → `https://sandbox-rest.avatax.com/api/v2`,
  `production` → `https://rest.avatax.com/api/v2`. **Sandbox and production credentials are not
  interchangeable** — production creds 401 against the sandbox URL and vice-versa.

### Auto-fallback

On **any** Avalara error the provider logs the exact status / Avalara error code / message / body
(grep `[tax] Avalara`) and returns the internal `TaxRule` result instead — checkout proceeds with
internal rates. Missing or invalid credentials therefore degrade gracefully rather than failing the
order.

### Cross-environment retry (env mismatch self-correction)

Because the env-vs-credentials mismatch is easy to get wrong, the provider is resilient: if a
transaction **401s** against the configured environment, it retries the request once against the
**other** environment. If the credentials authenticate there, it uses that result and logs a loud
`Set AVALARA_ENVIRONMENT=<env>` hint — so checkout still gets a real quote, but you should fix the
env var to avoid a wasted first request per quote. The health probe (§6) does the same and reports
`reason: environment_mismatch` with the environment the credentials actually belong to.

### Rate-limit resilience

Three layers protect the Avalara integration from excessive API calls:

| Layer                | Where                               | What it does                                                                                                                                           |
| -------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 2 s debounce         | frontend (`POST /api/tax/estimate`) | Waits 2 s after the last address keystroke before firing the estimate request; prevents a burst of calls during fast address entry.                    |
| 5-char street guard  | frontend                            | Skips the estimate call when the street field has fewer than 5 characters, preventing noise requests on partial input.                                 |
| 60 s in-memory cache | `avalara.provider.js`               | Caches each Avalara quote keyed by the full `address + items` fingerprint for 60 s; repeated estimates for the same cart do not consume Avalara quota. |

The debounce and guard apply to the **estimate** endpoint only; order-creation and refund calls are
not debounced (they happen once on submit and must complete synchronously).

---

## 5. Avalara tax codes

Each order line is sent to Avalara with a tax code. The default mapping
(`services/tax/providers/taxCodeMap.js`) is:

| Platform category                                                          | AvaTax code | Meaning                                   |
| -------------------------------------------------------------------------- | ----------- | ----------------------------------------- |
| `standard`, `digital`, `exempt`, unknown                                   | `P0000000`  | Tangible personal property (safe default) |
| `food`                                                                     | `PF050032`  | Food for home consumption                 |
| apparel (`dress`, `t-shirt`, `shirt`, `jacket`, `jeans`, `suit`, `hat`, …) | `PC040100`  | Clothing (general)                        |
| footwear (`shoes`, `shoe`)                                                 | `PC040108`  | Footwear                                  |
| (a line marked `taxExempt`)                                                | `NT`        | Non-taxable (exemption)                   |

**Overriding a code:** set the `avalaraTaxCode` field on a `TaxRule` for that category/jurisdiction
— it always wins over the map. Find the right AvaTax code with Avalara's tax-code search tool:
<https://taxcode.avatax.avalara.com>. Example: to tax `digital` goods as a specific SaaS code in the
US, create a
`TaxRule { country: 'US', categoryKey: 'digital', avalaraTaxCode: '<code from the tool>' }`.

---

## 6. TaxProvider health

`getTaxProviderHealth` (60 s TTL cache) backs `GET /api/admin/tax/health` and the **green/red
badge** on the Tax Rules admin panel.

- For `internal` it always reports `{ provider:'internal', healthy:true }` (no external dependency).
- For `avalara` it pings `GET /utilities/ping`. Avalara returns HTTP 200 even with bad credentials
  but reports `authenticated:false`, so **healthy requires a 2xx AND `authenticated:true`**. The
  response carries `{ environment, status, authenticated, reason }`; a misconfigured env reports
  `reason: 'environment_mismatch'` with the environment that actually authenticates.

Check it manually (admin session):

```
GET /api/admin/tax/health            # cached (≤60s)
GET /api/admin/tax/health?fresh=true # bypass the cache
```

A `reason` of `not_configured` (no creds), `not_authenticated` (bad creds for this env), `http_<n>`,
`timeout`, or `network_error` tells you exactly why the badge is red.

---

## 7. Order tax audit

Every order persists what was charged, for audit and refunds:

```
order.tax = {
  provider,                 // 'internal' | 'avalara'
  jurisdiction: { country, state },
  breakdown: [ { productId, taxCategory, taxableBase, rate, taxAmount, exempt } ],
  meta                      // provider response metadata, or null
}
```

For Avalara, `meta` holds the AvaTax `taxSummary`, the list of `jurisdictions`, the transaction
code, and a timestamp. For the internal provider `meta` is `null`. The field is `Mixed`, so swapping
or adding a provider needs no schema migration.

---

## 8. Quick admin recipes

- **Charge 15 % VAT everywhere:** one rule
  `{ country:'*', state:null, categoryKey:'standard', rate:0.15 }`. (Or just set `TAX_RATE=0.15` and
  add no rules.)
- **Reduced rate for food in Egypt:** `{ country:'EG', categoryKey:'food', rate:0.05 }` and set
  those products' `taxCategory` to `food` (or rely on a `food` subcategory slug).
- **Exempt a category:** create the rule with `taxExempt: true` (rate forced to 0; Avalara line →
  `NT`).
- **State-specific surcharge:**
  `{ country:'EG', state:'ALEXANDRIA', categoryKey:'standard', rate:0.20 }` — it wins over the
  country-level `standard` rule for Alexandria addresses.
- **Switch to Avalara:** set the `AVALARA_*` vars + `TAX_PROVIDER=avalara`, confirm the admin badge
  is green (`/api/admin/tax/health`), and keep your internal `TaxRule`s in place — they're the
  safety net.
- **Roll back instantly:** set `TAX_PROVIDER=internal` (or just leave the broken Avalara creds — the
  fallback already keeps charging the internal/`TAX_RATE` amounts).

---

## 9. Ship-from address and Avalara nexus

### Why ship-from matters

Several US states use **origin-based sourcing**: tax is assessed at the seller's origin
jurisdiction, not the customer's destination. Without the correct ship-from address, Avalara applies
the destination jurisdiction rules for all orders — potentially wrong in origin-based states and
misleading anywhere the origin and destination differ. Setting the store's ship-from address lets
Avalara compute the nexus correctly for every transaction.

### How sellers configure it

In **Store Settings → Ship-from address** (always visible, not gated by shipping mode), sellers fill
in five fields:

| Field            | Store document path                       |
| ---------------- | ----------------------------------------- |
| Street           | `store.shipping.carrier.originStreet`     |
| City             | `store.shipping.carrier.originCity`       |
| State / Province | `store.shipping.carrier.originState`      |
| Postal code      | `store.shipping.carrier.originPostalCode` |
| Country (ISO-2)  | `store.shipping.carrier.originCountry`    |

The street field uses the **`AddressAutocomplete`** component (same Places-proxy backend as the
checkout) and a **draggable-pin `AddressMap`** sits below the grid — dragging the pin reverse-
geocodes and fills all five fields at once.

### Activation guard

A store **cannot go active** until all five origin fields are non-blank:

```
PUT /api/stores/:id/status  { status: 'active' }
→ 400 { code: 'ship_from_required', message: '…' }   (when any field is empty)
```

Admin accounts bypass this guard (so platform operators can activate stores on behalf of sellers
during onboarding). Stores that were already active before this requirement was introduced are not
blocked; they receive a `shipFromComplete: false` flag and see an amber warning banner in the seller
dashboard until the address is saved.

### Avalara early-exit guard

`quoteWithAvalara` rejects calls where the ship-from street is blank _before_ making any HTTP
request:

```js
// If a shipFrom was provided but originStreet is empty, Avalara would return
// InvalidAddress — skip it and let the caller fall back to internal TaxRules.
if (args.shipFrom !== undefined && !args.shipFrom?.originStreet?.trim()) {
  throw new AvalaraError("ship_from_incomplete: ShipFrom address missing");
}
```

This means an incomplete origin degrades gracefully to internal `TaxRule`s rather than producing an
Avalara error or a misleading zero-tax result.

### How tax estimate and checkout both use ship-from

Both `POST /api/tax/estimate` and the Stripe/COD checkout resolve the store origin the same way:

1. Load the store by `storeId` (derived from the products in the cart, with the `storeId` field in
   the request body as a reliable fallback for products created before that field was indexed).
2. Read `store.shipping.carrier.origin*` and pass it as `shipFrom` to `getTaxProvider().quote()`.
3. The estimate frontend (web and mobile) sends `storeId` and a full `destination` (including
   `street` and `postalCode`) so Avalara gets the precise jurisdiction rather than just
   country/state. The mobile app previously sent only country/state — it now sends the complete
   destination object including street and postal code, and the correct `storeId`.

### `buildTransaction` fallback for missing street

When the destination address has no street (e.g. the customer has only entered country + state), the
`buildTransaction` helper passes an **empty string** — not `'N/A'` — as `line1` in the ShipTo block.
An empty string lets Avalara apply city/state-level jurisdiction rules; the literal `'N/A'` was
previously causing `InvalidAddress` rejections that silently fell back to `TAX_RATE`.
