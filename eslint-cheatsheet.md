# 🧠 ESLint Cheat Sheet for Learners

A developer-friendly overview of what each ESLint rule in your config teaches you.

---

## 📦 Core ESLint Rules (JavaScript/TypeScript)

| Rule                          | What It Teaches You                                                                                   |
| ----------------------------- | ----------------------------------------------------------------------------------------------------- |
| `no-unused-vars`              | Encourages you to remove or use declared variables — keeps code clean and focused.                    |
| `eqeqeq: ["error", "always"]` | Teaches strict equality (`===` instead of `==`) — avoids tricky type coercion bugs.                   |
| `no-console: "warn"`          | Reminds you to remove `console.log()` from production code — good hygiene.                            |
| `argsIgnorePattern: "^_"`     | Lets you name unused function parameters (like `_req`) without errors — helpful for Express handlers. |

---

## 🧠 TypeScript-Specific Rules

| Rule                                                      | What It Teaches You                                                                                   |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `@typescript-eslint/no-unused-vars`                       | Same as above but TS-aware — works better with type-only imports.                                     |
| `@typescript-eslint/no-explicit-any: "warn"`              | Discourages the use of `any` — nudges you toward better typing.                                       |
| `@typescript-eslint/explicit-function-return-type: "off"` | Allows implicit return types — fine for now, but later turn it on to write more explicit, clear APIs. |

---

## ⚛️ React Rules

| Rule                              | What It Teaches You                                                                |
| --------------------------------- | ---------------------------------------------------------------------------------- |
| `react/react-in-jsx-scope: "off"` | React 17+ no longer requires `import React` in JSX files — disables this old rule. |
| `react/jsx-uses-react: "off"`     | Prevents false "React is unused" errors in JSX — not needed anymore.               |
| `react/prop-types: "off"`         | You’re using TypeScript for props — `prop-types` are redundant.                    |

---

## 🪝 React Hooks Rules

| Rule                                  | What It Teaches You                                                                                         |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `react-hooks/rules-of-hooks: "error"` | Enforces **proper hook usage** — helps you avoid using hooks inside loops, conditions, or wrong places.     |
| `react-hooks/exhaustive-deps: "warn"` | Encourages proper dependency tracking in `useEffect` — prevents bugs from stale closures or missed updates. |

---

## 🔁 React Refresh Rule

| Rule                                   | What It Teaches You                                                                                            |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `react-refresh/only-export-components` | Prevents Fast Refresh from breaking — ensures **only components** are exported from modules that change often. |
| `allowConstantExport: true`            | Allows named constants alongside your component exports. Useful in Vite or Next.js.                            |

---

## ♿️ Accessibility (JSX A11Y)

| Rule                       | What It Teaches You                                                                                        |
| -------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `jsx-a11y/alt-text`        | Ensures `<img>` tags have meaningful `alt` — makes your app accessible to screen readers.                  |
| `jsx-a11y/anchor-is-valid` | Makes sure `<a>` tags aren’t broken (e.g., empty `href="#"`) — improves navigation and keyboard usability. |

---

## ⚙️ Parser & Environment Config

| Setting                                                   | What It Enables                                                |
| --------------------------------------------------------- | -------------------------------------------------------------- |
| `parser: @typescript-eslint/parser`                       | Enables ESLint to understand `.ts` and `.tsx` syntax.          |
| `parserOptions.project: "./tsconfig.json"`                | Reads your TS config — necessary for full type-checking rules. |
| `languageOptions.globals: globals.browser / globals.node` | Prevents false errors for `window`, `document`, or `process`.  |
| `sourceType: "module"`                                    | Allows you to use `import`/`export` syntax.                    |
| `ecmaFeatures.jsx: true`                                  | Enables JSX parsing in `.js` and `.jsx` files.                 |

---

## ✅ Summary for Learners

- Use warnings to improve code gradually — don’t treat them like errors.
- When a rule yells at you, **look up the rule docs** and try to understand **why** it exists.
- This config turns ESLint into your **teacher**, not just a nag.
