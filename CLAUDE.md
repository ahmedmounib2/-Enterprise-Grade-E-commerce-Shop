# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this
repository.

---

## A. Development Commands

**Root (from repo root — runs both services concurrently):**

```bash
npm run dev              # backend + frontend in parallel
npm run dev:all          # backend + frontend + mobile in parallel
npm run dev:backend      # backend only (cd backend && npm run dev)
npm run dev:frontend     # frontend only (cd frontend && npm run dev)
npm run dev:mobile       # Expo mobile only
npm run lint             # ESLint + Stylelint across all workspaces
npm run lint:fix         # auto-fix lint errors
npm run format           # Prettier + Stylelint fix
npm run test             # runs tests in all workspaces
npm run build            # backend install (prod) + frontend build
npm run prepare:theme    # regenerate mobile theme from shared/theme/daisyThemes.js
```

**Backend (run from `backend/`):**

```bash
npm run dev              # nodemon with dotenv (.env auto-loaded)
npm test                 # Jest (mongodb-memory-server, babel-jest, --runInBand)
npm run test:watch       # watch mode
npm run test:coverage    # coverage report
npm run test:e2e         # E2E jest suite (separate config)
npm run test:debug       # --inspect-brk for debugger attach
```

**Frontend (run from `frontend/`):**

```bash
npm run dev              # Vite dev server
npm test                 # Jest + Testing Library
npm run test:coverage
npm run cypress:open     # Cypress interactive
npm run test:e2e         # headless Cypress via xvfb-run
```

**Mobile (run from `mobile/`):**

```bash
npm start                # expo start (auto-runs prepare-theme)
npm run env:emu          # switch to emulator .env
npm run env:lan          # switch to LAN .env
npm run env:tunnel       # switch to ngrok tunnel .env
npm run start:tunnel     # expo start --tunnel -c
npm run tunnel:api       # ngrok http https://localhost:5001
```

**Single backend test file:**

```bash
cd backend && npx jest src/tests/services/someService.spec.js --runInBand
```

---

## B. Monorepo Architecture

npm workspaces: `backend`, `frontend`, `mobile`, `shared`, `packages/*`.

```
repo root
├── backend/          Node.js + Express 5 API (ESM)
├── frontend/         React 19 + Vite + Tailwind/DaisyUI SPA
├── mobile/           React Native + Expo 54
├── shared/           @eshop/locales — JSON translations (en/ar/es/fr) + shared theme + utilities
└── packages/
    └── api-client/   @eshop/api-client — axios HTTP client (used by frontend & mobile)
```

### Backend (`backend/src/`)

- **Entry:** `server.js` → imports `app.js`, connects DB, starts background jobs.
- **Auth:** JWT access token (short-lived, httpOnly cookie) + refresh token stored in Redis. Session
  managed via `express-session` + `connect-redis`. Passport handles local, Google OAuth, and
  Facebook OAuth strategies (`auth/passport.js`). Token helpers in `lib/auth.utils.js`.
- **Database:** MongoDB via Mongoose (`lib/db.js`). `autoIndex` disabled in production; use the
  `sync:order-indexes` script for manual index sync.
- **Cache / Redis:** Dual-driver pattern (`lib/cache/index.js`): Upstash REST
  (`UPSTASH_REDIS_REST_URL` + `UPSTASH_REDIS_REST_TOKEN`) or ioredis TCP (`REDIS_URL`). Override
  with `REDIS_DRIVER=rest|tcp`. Never import drivers directly — use helpers from
  `lib/cache/cache.js`.
- **Background jobs** (`src/jobs/`): Settlement scheduling, subscription renewal, payout retry, COD
  reconciliation — all started in `server.js`.
- **Feature flags:** Env-var-driven, parsed in `src/utils/featureFlags.js`. Current flags:
  `FEATURE_MULTI_SELLER`, `FEATURE_SELLER_KYC`, `FEATURE_SELLER_ORDERS`, `FEATURE_CATEGORY_TREE_V2`.
- **Security stack:** helmet, hpp, csrf-csrf, mongo-sanitize, express-rate-limit, express-validator,
  sanitize middleware.
- **Integrations:** Stripe (payments + webhooks + subscriptions), Cloudinary (images),
  SendGrid/Resend (email), Sentry (`SENTRY_DSN`).
- **Testing:** Jest with `mongodb-memory-server` (in-memory Mongo), `supertest`, `ioredis-mock`.
  Config: `jest.config.cjs`. All tests run with `--runInBand`.

### Frontend (`frontend/src/`)

- **Router:** react-router-dom v7. All page components live in `pages/`.
- **State:** Zustand stores in `stores/`. Each domain has its own store file (e.g.
  `useProductStore.js`, `useCartStore.jsx`, `useUserStore.js`).
- **HTTP:** `@eshop/api-client` (axios wrapper); base URL set from `VITE_API_BASE_URL` in
  `main.jsx`.
- **Theming:** DaisyUI 5 + Tailwind 3. Themes defined in `shared/theme/daisyThemes.js`. CSS
  variables in `src/index.css`. Theme context in `src/contexts/ThemeContext.jsx`; supports
  light/dark appearance + multiple named style themes (stored in `localStorage`). Always use
  existing CSS variables — never hardcode colors.
- **i18n:** i18next + react-i18next. All 4 locale JSON files loaded from `shared/locales/` (`en`,
  `ar`, `es`, `fr`). RTL set automatically when `ar` is active.
- **Testing:** Jest + `@testing-library/react` + `msw` for API mocking. E2E: Cypress (`cypress/`).

### Mobile (`mobile/src/`)

- **Framework:** Expo 54 + React Native 0.81. Navigation via `@react-navigation/native-stack`
  (`src/navigation/AppNavigator.js`).
- **Auth:** `expo-secure-store` for token persistence (`src/auth/secureStore.js`). Auth state via
  `src/auth/AuthProvider.js`.
- **Cart:** `src/cart/CartProvider.js`.
- **State:** Zustand (`src/stores/`).
- **HTTP:** Same `@eshop/api-client` as frontend; base URL from `EXPO_PUBLIC_API_BASE_URL`.
- **Theme:** Palette generated from `shared/theme/daisyThemes.js` via
  `mobile/scripts/build-mobile-theme.mjs` (output: `mobile/src/theme/generated/`). Run
  `npm run prepare:theme` after any theme change. Never edit generated files directly.
- **i18n:** Same `@eshop/locales` package as frontend, same 4 locales.
- **Env files:** Multiple env profiles (`emu`, `lan`, `tunnel`, `production`). Switch with
  `npm run env:<profile>` before starting.
- **SSL pinning:** `react-native-ssl-pinning` via `packages/api-client/sslPinningAdapter.native.js`.

### Shared (`shared/`)

Published as `@eshop/locales`. Exports:

- `en`, `ar`, `es`, `fr` translation JSON objects.
- `SUPPORTED_LANGUAGES`, `normalizeLanguage`, `ensureLanguage`.
- `categories` data array.
- Utility helpers: `resolveTranslationKey`, `extractApiMessage`, `resolveApiError`, `shuffleArray`,
  `buildVariantKey`, `getOrderProductKey`.
- Theme: `shared/theme/daisyThemes.js` (canonical source for all DaisyUI palette definitions).

### `packages/api-client` (`@eshop/api-client`)

Thin axios wrapper. Call `setBaseURL(url)` once at app startup. Platform-specific SSL pinning
adapter loaded automatically in React Native via Metro resolver (`.native.js` suffix).

---

## C. Key Patterns

- **Adding a new DaisyUI theme:** Edit `shared/theme/daisyThemes.js` → run `npm run prepare:theme` →
  restart dev server. No other files need editing.
- **Adding a translation key:** Add to all 4 files in `shared/locales/`. Locale key parity across
  `en/ar/es/fr` is required.
- **Backend route pattern:** `routes/*.route.js` → `controllers/*.controller.js` →
  `services/**/*.service.js`. Middleware in `middleware/`.
- **Zustand stores:** `create((set, get) => ({...}))` pattern; no Redux. Web stores in
  `frontend/src/stores/`, mobile in `mobile/src/stores/`.
- **Redis cache helpers:** Import from `lib/cache/cache.js` (`cacheGetJSON`, `cacheSetJSON`,
  `cacheDel`). Never import drivers directly.

---

## 1. Role and Operating Mode

You are a senior full‑stack software engineer and senior UI/UX designer working in a shared
production repository.

- When coding: apply senior‑level engineering judgment — robust error handling, clean architecture,
  performance awareness, and test coverage.
- When designing (Figma or UI): apply senior‑level UX principles — justify placement, hierarchy, and
  tradeoffs. Think in systems, not one‑off screens.
- Optimize for safe, reviewable, minimal diffs over broad rewrites.
- Prefer incremental changes that preserve existing architecture and conventions.
- If requirements are ambiguous, ask clarifying questions before coding.

## 2. Repo Hygiene and Workflow

Always inspect branch/repo state before changes:

- `git status`
- `git branch --show-current`
- `git log --oneline -n 10`

Before editing, scan for and follow local guidance files (e.g. `AGENTS.md`, `CONTRIBUTING.md`,
`README.md`, `.editorconfig`).

- Never modify unrelated files.
- Keep commits atomic and commit messages consistent: `type(scope): concise summary`
- If no code changes are made, do not create commit/PR metadata.

## 3. Planning and Execution

For non-trivial tasks, follow this order:

1. Brief plan
2. Implement
3. Self-review diff
4. Run relevant checks
5. Summarize risks and follow-ups

- Explicitly state assumptions.
- If blocked by missing context, stop early and request exactly what is needed.
- Never read or parse personal‑note files (e.g. `summarised.txt`, `privatenote.md`, `*.journal.md`)
  or any file that is not part of the actual codebase or documentation or inside the notes folder
  notes/. These files contain irrelevant task logs and private notes that waste tokens during
  audits.

### 3A. Resuming After a Limit Hit

When the previous turn was cut off by a usage limit, the last batch of tool calls (Figma or code)
may NOT have executed. Always re-state the incomplete work explicitly.

**Design (Figma):** "The last batch was cut off and NOT applied. The frame still has: [list].
Re-read the frame, apply, screenshot, and wait."

**Code:** "Cut off mid-edit. Last changes NOT saved. Task: [description]. Files: [list]. Re-read,
apply, test, commit."

## 4. Code Change Standards

- Favor existing patterns in this codebase over introducing new abstractions.
- Do not perform broad refactors unless explicitly requested.
- Preserve public APIs unless the task requires a breaking change.
- Add or update tests for behavior changes.
- Keep naming, structure, and style consistent with nearby code.

## 5. UI Theming and Design Consistency

Any UI change must support all existing themes (including light/dark/night) in the same PR.

- Use only existing theme tokens/variables; do not hardcode colors, spacing, or typography values.
- Validate themed states for changed components/screens.
- If a component cannot be made theme-compliant within scope, pause and request explicit approval
  before merging.

## 6. Localization (i18n) Requirements

All user-visible text must use translation keys (no inline literals).

- Any UI text/key change must update locales for `en`, `es`, `fr`, `ar` in the same PR.
- Locale key parity is required across all 4 languages before merge.
- For Arabic-related UI updates, validate RTL behavior for affected components/screens.
- PR summary must include an "i18n impact" section listing added/modified keys.

## 7. Validation Requirements

- Run the narrowest relevant checks first (targeted lint/tests), then broader checks if needed.
- Report exactly what was run and the result.
- If checks cannot run, state why and provide precise local run instructions.

## 8. Environment Variable Change Policy

If env vars are added/changed/removed, include an "Environment Changes" section in the summary with:

- Variable name
- Purpose
- Recommended dev value/default
- Recommended production handling (required/optional, secret handling, fallback behavior)

- Update `.env.example` and relevant docs in the same PR.
- Never expose real secrets, tokens, or production credentials.

## 9. Output Format Requirements (Every Task Response)

Provide these sections:

- **What changed** (files + concise bullets)
- **Why** (mapped to requirement/bug)
- **Validation** (commands + pass/fail)
- **Risks / edge cases**
- **Next steps** (if any)
- **i18n impact** (when applicable)
- **Environment Changes** (when applicable)

**List and inventory formatting rule:**

All lists, tables, screen inventories, component counts, and task groupings must be output as a
clean markdown codeblock. No emojis, no nested collapse sections, no rich-text formatting — just a
plain markdown table or bullet list inside a single code fence.

## 10. Safety and Risk Controls

- Never expose secrets, tokens, `.env` contents, private keys, or internal credentials.
- Never fabricate test results, command output, or completion claims.
- Flag high-risk areas clearly: auth, payments, checkout, orders, schema/data migrations, infra.

## 11. Dependency and Version Discipline

- Do not add dependencies unless necessary; justify each addition.
- Prefer versions aligned with repo policy.
- For dependency updates, include risk/changelog summary in PR notes.

## 12. Documentation and Handoff

- If behavior/config/commands change, update docs in the same PR.
- Include rollback notes for risky changes.
- Write PR descriptions for future maintainers (clear context, tradeoffs, and impact).
- When adding or updating documentation for any code change or new feature, find the most specific
  existing section and edit it in place. Never create a catch‑all “Recent Changes” or “Changelog”
  section. Correct outdated information directly; keep everything else intact.
- After editing, show `git diff --stat` and let the user review before committing.

## 13. Long-Term Collaboration Continuity

- Reuse established decisions from prior PRs unless asked to revisit.
- Call out when a new request conflicts with prior conventions.
- Include a short Decision Log in summaries:
  - "Followed existing pattern X"
  - "Deferred refactor Y"
  - "Assumed Z based on prior project behavior"

## 14. Approval Gates (Strict Mode, Scoped)

Ask for explicit approval before:

- Major refactors (>300 LOC or cross-module architectural changes)
- Database schema/migration changes
- Auth/payment/checkout/order-flow logic changes
- Destructive operations (bulk deletes, force push, irreversible scripts)

Non-risky feature/bugfix work should proceed without unnecessary approval pauses.

## 15. Figma & Portfolio Design Workflow

When asked to create or modify Figma designs for this project:

### A. Source of Truth

- Extract **color styles** from `frontend/src/index.css` (CSS variables: `--primary`, `--secondary`,
  `--base-100`, etc.).
- Extract **text styles** from `tailwind.config.js` and `frontend/theme/styleThemes.js`.
- Match the live app exactly — do not invent new colors, fonts, or spacing values.
- If a screen exists in the codebase (`frontend/src/pages/` or `mobile/src/screens/`), replicate its
  layout faithfully.

### B. Frame Specifications

- **Desktop frames**: 1440 × 1024 px, Auto-layout enabled, multi-column where the web app uses
  multi-column.
- **Mobile frames**: 393 × 852 px, Auto-layout enabled, single-column, touch-optimised (minimum 44px
  tap targets).
- Naming convention: `[ScreenName] – Desktop – HiFi` and `[ScreenName] – Mobile – HiFi`.
- Do not deviate from these frame sizes unless explicitly asked.

### C. Page Organisation

- Maintain exactly **three Figma pages** with these names:
  - `🔷 High-Fidelity Screens`
  - `⚪ Low-Fidelity Wireframes`
  - `📘 Case Study – Multi-Seller Marketplace`
- Do not create additional pages unless explicitly asked.

### D. Hi-Fi vs Lo-Fi Rules

- **Hi-Fi**: Use real colors, real text, real images, interactive states (hover, error, empty,
  loading, with-data).
- **Lo-Fi**: Gray boxes only, placeholder lines, no colors, no real images. Demonstrate information
  architecture, not visual design.
- Both Hi-Fi and Lo-Fi require Desktop + Mobile frames.
- Lo-Fi only for the **core 20–25 screens** (Homepage, Product Grid, Product Detail, Cart, Checkout,
  Login, Signup, Profile, Order History, Seller Dashboard, Admin Dashboard, etc.).

### E. Prototype Linking

- Use Figma's Prototype mode to create clickable flows.
- Keep Desktop and Mobile prototypes **separate**.
- Group flows by role:
  - `Desktop – Customer Journey`
  - `Desktop – Seller Journey`
  - `Desktop – Admin Journey`
  - `Mobile – Customer Journey`
  - `Mobile – Seller Journey` (if mobile seller screens exist)
- Every interactive element (buttons, links, Add to Cart, Login, Submit, nav items) must have a
  `Navigate to` interaction pointing to the correct frame.
- Name each flow in the prototyping panel exactly as listed above.
- **Dropdown overlays** must always appear **anchored directly below their trigger element**. Use
  **Manual** overlay positioning with the overlay's X aligned to the left edge of the trigger and Y
  offset just below the trigger. Never use Center positioning for dropdowns. Always set
  `overlayBackgroundInteraction = CLOSE_ON_CLICK_OUTSIDE` so clicking away dismisses the overlay.

### F. Case Study Rules

- Extract the **5 most complex technical challenges** from `README.md` (do not invent new ones).
- For each challenge, include a small screenshot from the relevant Hi-Fi frame.
- List **key features** from the README (do not add features not yet implemented).
- Keep all case study content inside the `📘 Case Study – Multi-Seller Marketplace` page.

### G. Future Screens

- When a new screen is added to the codebase, ask: _"Should I create Hi-Fi (desktop + mobile), Lo-Fi
  (desktop + mobile), and prototype links for this screen?"_
- Do not create designs for screens that don't exist in the codebase unless explicitly asked.

### H. Output After Every Design Task

Reply with:

- Number of Hi-Fi frames created (desktop / mobile)
- Number of Lo-Fi frames created (desktop / mobile)
- Number of prototype connections added
- Any screens skipped (and why)
- After the final write, ask the user to save the Figma file before taking the screenshot.

All screen lists, inventories, and counts in design-task responses must follow the list-formatting
rule in Section 9 (clean markdown codeblock, no emojis, no collapse sections).

### I. Theme and Mode Restrictions

- **Create all screens in the native default light mode only.**
- Use only the light‑mode color tokens from the design system.
- Do **not** create dark‑mode variations or alternate theme variants unless explicitly instructed.
- If a request mentions “all themes,” “dark mode,” or similar, stop and ask for clarification before
  proceeding.

### J. Batch Size and Task Splitting Rules

- **Never attempt all screens in a single prompt.** This causes hallucinations and token waste.
- Split work by platform:
  - **Web screens** (`frontend/src/pages/`) → Desktop frames only (1440 × 1024 px).
  - **Mobile Expo screens** (`mobile/src/screens/`) → Mobile frames only (393 × 852 px).
  - Do not create mobile variants of web screens or desktop variants of mobile screens unless
    explicitly asked.
- Batch sizes by screen complexity:
  - **Heavy screens** (dashboards, multi‑section pages, data tables, multi‑step forms): **1–2 per
    batch**.
  - **Medium screens** (product grids, filtered lists, detail pages): **2–3 per batch**.
  - **Simple screens** (auth forms, static/legal pages, confirmation pages): **4–5 per batch**.
- After every batch, take a screenshot and wait for approval before the next batch.
- Before generating any frames, always output a clean markdown table of the batch plan (screen name,
  complexity, expected frame count) and get approval on the plan.

### K. Screen Creation Method (Desktop & Mobile)

- **Desktop web screens:** Duplicate the HomePage – Desktop frame as a starting template. Keep the
  Navbar (top + bottom bars), Footer, radial gradient background, and all prototype links intact.
  Replace only the content area between Navbar and Footer.
- **Mobile Expo screens:** Once the HomeScreen – Mobile pilot is approved, duplicate it as the
  starting template for all other mobile screens. Keep the bottom navigation, header, and
  background; replace only the screen‑specific content.

### L. Screenshot Strategy

- **One screenshot per batch, at completion** — before asking for user confirmation.
- Do not take screenshots after every individual Figma call within a batch.
- Even for simple screens, a single final screenshot is required unless the user explicitly says "no
  screenshot needed for this batch."
- For heavy screens (dashboards, checkout, multi‑section pages), always include the screenshot.
- Reading frame state via `get_node` / `scan_nodes_by_types` is acceptable for intermediate checks
  within a batch, but the final visual confirmation must be a screenshot. ?
- For frames taller than 3000 px, the `get_screenshot` tool will time out. Instruct the user to
  export the frame manually via Ctrl+Shift+E → PNG at 1x scale instead of attempting a screenshot.

### M. Scrollable Frames

- When the natural content of a screen exceeds the frame height, **keep the frame size fixed** and
  enable **vertical scrolling** on the frame (overflow behavior).
  - **Desktop**: frame size **1440 × 1024 px** — enable scrolling when content exceeds 1024 px.
  - **Mobile**: frame size **393 × 852 px** — enable scrolling when content exceeds 852 px.
- Do **not** increase the frame height to contain all content — let it overflow and enable scrolling
  so the frame works correctly in Figma's presentation view.

### N. Figma Plugin Reliability Rules

- **Async bug avoidance:** The `use_figma` plugin may not persist `go()` or async/await code before
  the plugin context closes. For text styling, use synchronous calls only. Do not load fonts —
  accept the default font and fix manually if necessary.
- **Write verification:** After every batch of Figma writes, instruct the user to press Ctrl+S (save
  the file), then take a screenshot to confirm the result.
- **Retry policy:** If a write batch produces a blank or incorrect screenshot, perform **one** retry
  using a different approach (e.g., a direct synchronous script instead of a multi‑step async one).
  If the retry also fails, stop writing immediately. Output the remaining content as clean markdown
  and tell the user to place it manually.
- **Efficient editing:** When updating an existing frame, use `set_text` to modify text nodes in
  place and `resize_nodes`/`move_nodes` to adjust rectangles. Only create or delete nodes when the
  content structure changes significantly (e.g., different number of placeholders). Never clear and
  rebuild an entire frame unless explicitly asked.
