# Theme token system

Each client reads from a **single source of truth** so you can rebrand the experience by editing one
file per platform and, if needed, copy-pasting the tokens between them.

- **Web theme file:** [`frontend/src/theme/themeTokens.js`](../../frontend/src/theme/themeTokens.js)
- **Mobile theme file:** [`mobile/src/theme/themeTokens.js`](../../mobile/src/theme/themeTokens.js)
- **DaisyUI bridge:** [`shared/theme/daisyThemes.js`](../../shared/theme/daisyThemes.js) feeds both
  platforms via the style theme bridge and the generated mirror in
  `mobile/src/theme/generated/daisyThemes.js`.

Both files share the same structure (`NATIVE_LIGHT_THEME`, `NATIVE_DARK_THEME`,
`DEFAULT_STYLE_THEME_ID`, etc.). Copy the entire file from one platform to the other whenever you
want the website and mobile app to use identical colors and fonts.

## Editing the palette

1. Open the platform-specific `themeTokens.js` file.
2. Locate the `palette` object inside `NATIVE_LIGHT_THEME` (for light mode) or `NATIVE_DARK_THEME`
   (for dark mode).
3. Update the hex/RGBA value next to the descriptive comment. For example, to change the primary
   button color adjust `primary` and `primaryContent`.
4. Save the file – Vite/Metro will hot reload and apply the new CSS variables or React Native styles
   across the app.

If you change DaisyUI palettes instead, update `shared/theme/daisyThemes.js` and then run
`npm run gen:mobile-theme` (or `npm -w mobile run prepare-theme`) to regenerate the Expo mirror
before restarting the clients.

Every key in the palette is described inline:

- `background`, `topBar`, and `card` drive page surfaces.
- `border`, `pillBg`, and `muted` cover outlines, chips, and skeletons.
- `primary`, `secondary`, `accent`, and their `*Content` variants set button and badge colors.
- `info`, `success`, `warning`, and `error` map to status messaging.

## Updating typography

Each theme also exposes a `fonts` block. The web provider pushes these values into CSS custom
properties (`--font-base`, `--font-heading`) so the entire site switches typefaces automatically.
The mobile provider exposes the same data via context – use `useThemePreference()` to read `fonts`
if you need to style custom components.

To change typography:

1. Replace the `fonts.base` and/or `fonts.heading` strings with the desired font stacks (web) or
   installed React Native font family names (mobile).
2. For best parity, mirror the same values in both files before copying.

## Quick checklist after edits

- Toggle between light and dark mode on web and mobile to confirm the new palette looks correct.
- Verify headings and body text pick up the updated font tokens.
- If you copied tokens between platforms, clear caches/hot reload both clients to ensure they
  re-import the new definitions.
- Run the linters if you touched additional styling code: `npm run lint` (web) or
  `npm -w mobile run lint` (if applicable).

With this setup you can rebrand the entire product by editing two files and, if desired, copying one
into the other for perfect parity.

## Consuming tokens in components

Both the web and mobile apps expose helper hooks to avoid hard-coded colour strings:

- Use `useThemePalette()` in React Native to read surface, text, and status colours that already
  honour the active DaisyUI or native theme selection.
- Access `activeStyleVariant.daisy` from `useThemePreference()` when you need a raw DaisyUI token
  that is not mapped onto the palette yet.

Avoid embedding hex or rgba values inside components – doing so bypasses the shared theme system and
results in mismatched styling between the website and the mobile client.
