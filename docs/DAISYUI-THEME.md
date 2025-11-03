# DaisyUI theme system

This document explains how the DaisyUI themes are wired up for the storefront (web) and the Expo
mobile app. Treat it as the handbook for updating colours, adding new palettes, or rolling
campaign-specific themes.

## Key concepts

- **Native themes ≠ DaisyUI themes** — the files in `themeTokens.js` are still responsible for the
  handcrafted light/dark palettes. DaisyUI themes exist purely for the Tailwind/Daisy ecosystem and
  are bridged in `styleThemes.js` so the two systems never fight each other.
- **Single shared source of truth** — `shared/theme/daisyThemes.js` defines every DaisyUI palette.
  The web imports it directly while the mobile workspace generates a converted mirror for React
  Native.
- **Complete catalogue** — the files are generated from the official DaisyUI CSS themes (35 presets
  as of v5.3) and also include the bespoke native storefront palette so the theme picker always has
  a "house" option.
- **Light and dark variants** — every DaisyUI theme exports both modes. When a visitor toggles
  appearance, DaisyUI simply swaps between `*-light` and `*-dark` without touching the native theme
  preference logic.

## File locations

| Platform | Path                                        | Purpose                                                       |
| -------- | ------------------------------------------- | ------------------------------------------------------------- |
| Shared   | `shared/theme/daisyThemes.js`               | DaisyUI palette definitions consumed by all platforms.        |
| Web      | `frontend/theme/styleThemes.js`             | Bridges DaisyUI palettes with the native tokens/context.      |
| Web      | `frontend/src/contexts/ThemeContext.jsx`    | Applies DaisyUI IDs + CSS vars to the DOM at runtime.         |
| Mobile   | `mobile/src/theme/generated/daisyThemes.js` | Generated mirror for NativeWind/Tailwind RN.                  |
| Mobile   | `mobile/src/theme/styleThemes.js`           | Mobile bridge that keeps native + DaisyUI logic separated.    |
| Mobile   | `mobile/src/theme/ThemeContext.js`          | Propagates active palette/fonts across the Expo app.          |
| Shared   | `frontend/tailwind.config.js`               | Registers DaisyUI and loads the exported themes for Tailwind. |
| Docs     | `docs/DAISYUI-THEME.md` (this file)         | Reference guide for future theme changes.                     |

## How themes are registered

1. `shared/theme/daisyThemes.js` exports an array of high-level theme objects
   (`DAISY_STYLE_THEMES`). Each object contains both the light and dark mode palette.
2. `frontend/theme/styleThemes.js` consumes those objects, converts them into app-friendly palettes,
   and exposes helper arrays for the context and Tailwind.
3. `frontend/tailwind.config.js` imports the prebuilt helpers and passes them to the DaisyUI plugin.
4. `mobile/scripts/build-mobile-theme.mjs` reads the shared catalogue and serialises it into
   `mobile/src/theme/generated/daisyThemes.js` so Metro loads a pure JavaScript module instead of
   the original ESM source.
5. At runtime, `frontend/src/contexts/ThemeContext.jsx` sets `data-theme="<id>-light|dark"` on the
   `<html>` element (and mirrors the choice in local storage) so DaisyUI swaps palettes without
   touching the native token selection. The same context also writes CSS variables for legacy
   components while keeping `color-scheme` up to date.
6. On mobile, `mobile/src/theme/ThemeContext.js` exposes the converted palettes through
   `useThemePreference()` so NativeWind-powered components and bespoke views can read the active
   DaisyUI colours.

### Keeping in sync with upstream DaisyUI

The `daisyThemes.js` files are generated from the official DaisyUI CSS theme tokens so the palettes
always match what upstream ships. If DaisyUI releases a new preset:

1. Fetch the updated CSS from the DaisyUI repository and regenerate `shared/theme/daisyThemes.js`
   (the current file header calls out that it is auto-generated).
2. Run `npm run gen:mobile-theme` (or `npm -w mobile run prepare-theme`) to rebuild
   `mobile/src/theme/generated/daisyThemes.js` so Expo shares the same palette catalogue.
3. Bump `DAISY_THEME_VERSION` if you remove or rename theme IDs so downstream caches know to
   refresh.

## Making colour changes

1. Decide whether you are editing an existing theme or adding a new one.
2. Update the relevant palette inside `shared/theme/daisyThemes.js`. Every token is documented
   inline to explain where it appears. Start with the commented "Abyss" entry as a reference for how
   the keys map to UI areas.
3. Run `npm run gen:mobile-theme` so the Expo project receives the updated, converted palette in
   `mobile/src/theme/generated/daisyThemes.js`.
4. Restart your dev server (`npm -w frontend run dev`) so Tailwind picks up the updated
   configuration and `ThemeContext` rehydrates the new palette IDs.

### Adding a brand-new theme

1. Duplicate one of the existing entries inside `DAISY_STYLE_THEMES`.
2. Change `id`, `label`, and `description`.
3. Fill the `light` palette with the theme's lighter colours and the `dark` palette with the darker
   tones. Always supply matching `*-content` colours so text remains legible.
4. Save the file and restart the dev server.
5. Run `npm run gen:mobile-theme` to regenerate the Expo mirror. The `styleThemes.js` bridge files
   will automatically expose the new entry to the pickers.

### Palette token reference

The table below lists the DaisyUI tokens we rely on and the parts of the interface they power. Use
it alongside the comments in the `Abyss` theme to quickly locate the value you need to adjust.

| Token               | Description                                                                |
| ------------------- | -------------------------------------------------------------------------- |
| `base-100`          | Primary page background.                                                   |
| `base-200`          | Card and panel background.                                                 |
| `base-300`          | Border colour for outlines, separators, and subtle UI chrome.              |
| `base-content`      | Default text colour on `base-*` surfaces.                                  |
| `neutral`           | Muted interface elements (tags, input chrome) and fallback borders.        |
| `neutral-content`   | Text/icons displayed on neutral elements.                                  |
| `primary`           | Main brand/action colour for buttons and key links.                        |
| `primary-content`   | Foreground colour used on top of the primary colour.                       |
| `secondary`         | Secondary actions and highlight elements.                                  |
| `secondary-content` | Foreground colour used on secondary accents.                               |
| `accent`            | Supporting accent colour for tertiary elements such as badges.             |
| `accent-content`    | Foreground colour on accent surfaces.                                      |
| `info`              | Informational alerts or indicator surfaces.                                |
| `info-content`      | Foreground colour on informational surfaces.                               |
| `success`           | Positive state backgrounds.                                                |
| `success-content`   | Foreground colour on success surfaces.                                     |
| `warning`           | Warning/attention backgrounds.                                             |
| `warning-content`   | Foreground colour on warning surfaces.                                     |
| `error`             | Destructive or error backgrounds.                                          |
| `error-content`     | Foreground colour on error surfaces.                                       |
| `color-scheme`      | Signals whether the palette is a light or dark variant (used by browsers). |

### Syncing with the mobile app

The Expo project can consume the same palettes through NativeWind or another Tailwind-to-RN
solution. Because both files export identical data structures you can:

Run `npm run gen:mobile-theme` (or `npm -w mobile run prepare-theme`) whenever you approve a palette
update so the Expo mirror stays in lockstep with the shared source of truth. Expo's `prestart`
script runs the same generator automatically, but it's still useful to execute it manually when you
edit palettes outside the mobile workspace.

### Consuming Daisy colours in React Native

Mobile screens should always read colours from the theme context to stay in sync with the selected
palette. Use `useThemePalette()` for high-level tokens (surface, text, status colours) and
`activeStyleVariant.daisy` when you need raw DaisyUI keys that are not part of the unified palette.
Avoid hard-coding hex or rgba strings inside components – the Expo client now mirrors the full web
catalogue, so every accent can be drawn from the shared tokens.

## Commands & tooling

- **Install dependencies**: `npm install`
- **Start the frontend with DaisyUI**: `npm -w frontend run dev`
- **Build the frontend**: `npm -w frontend run build`
- **(Optional) Lint after changes**: `npm -w frontend run lint`

No additional scripts are required; Tailwind automatically pulls the latest palette definitions on
restart.

## Troubleshooting

- If colours do not change after editing the theme file, confirm that the dev server was restarted.
  Tailwind caches the config on boot.
- Verify the `data-theme` attribute on `<html>` or `<body>` matches the generated theme name
  (`<id>-light` or `<id>-dark`).
- Double-check for typos in hex codes; invalid values silently fall back to defaults.

## When to bump `DAISY_THEME_VERSION`

Increment the exported `DAISY_THEME_VERSION` constant when you make a breaking change (e.g.,
removing a theme or renaming IDs). This lets downstream consumers invalidate caches or migrations
based on the version number.
