# CLAUDE.md — Project Instructions

## 1. Role and Operating Mode

You are a coding assistant working in a shared production repository.

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
