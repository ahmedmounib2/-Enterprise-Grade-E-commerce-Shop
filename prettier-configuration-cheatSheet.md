| 📌 General Formatting Rules                                            | Setting                                                        | Value                                                  | What It Means                        | Why It Matters                                             |                        |
| ---------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------ | ---------------------------------------------------------- | ---------------------- |
| ------------------------------------------------------------           |
| ---------------------------------------------------------------------- |                                                                | `semi`                                                 | `true`                               | Always                                                     |
| add semicolons at the end of statements                                | Avoids bugs caused by JavaScript’s automatic semicolon         |
| insertion (ASI)                                                        |                                                                | `singleQuote`                                          | `true`                               | Use `'` instead of `"` for strings                         | Matches popular        |
| style guides like Airbnb                                               |                                                                | `tabWidth`                                             | `2`                                  | Indent using 2 spaces                                      | Standard in most JS/TS |
| codebases                                                              |                                                                | `trailingComma`                                        | `"es5"`                              | Add trailing commas in arrays, objects, etc. (not          |
| functions)                                                             | Improves Git diffs and allows cleaner edits                    |                                                        | `printWidth`                         | `100`                                                      | Wrap lines after       |
| 100 characters                                                         | Keeps code readable on small screens or side-by-side panes     |                                                        | `bracketSpacing`                     |
| `true`                                                                 | Adds spaces inside object brackets: `{ foo: bar }`             | Improves readability of object                         |
| literals                                                               |                                                                | `arrowParens`                                          | `"always"`                           | Always use parentheses in arrow functions, e.g. `(x) => x` |
| Consistency and clarity for all function signatures                    |                                                                | `parser`                                               | `"typescript"`                       | Default parser                                             |
| is TypeScript                                                          | Ensures full support for `.ts` and `.tsx` files                |                                                        | `jsxSingleQuote`                     | `false`                                                    | Use                    |
| double quotes in JSX: `<div className="x" />`                          | Follows React community standard                               |                                                        |
| `embeddedLanguageFormatting`                                           | `"auto"`                                                       | Auto-format embedded languages (like CSS in `<style>`) |
| Keeps embedded code clean too                                          |                                                                | `bracketSameLine`                                      | `true`                               | JSX closing tag bracket stays on the                       |
| same line                                                              | Styling preference — often used in UI-heavy codebases          |                                                        | `quoteProps`                         | `"consistent"`                                             |
| Quote object keys if at least one key needs quotes                     | Keeps object definitions uniform                               |                                                        |
| `proseWrap`                                                            | `"always"`                                                     | Wrap Markdown or long text lines                       | Easier to read/edit documentation in |
| editors                                                                | 🔧 Overrides These settings apply only to specific file types. |

📄 JSON / YAML Files { "files": ["*.json", "*.yml", "*.yaml"], "options": { "tabWidth": 2,
"useTabs": false } } | Setting | Purpose | | ---------------- |
----------------------------------------------- | | `tabWidth: 2` | Two spaces for indentation | |
`useTabs: false` | Use spaces, not tabs — standard in config files | 📘 Markdown Files { "files":
"*.md", "options": { "proseWrap": "always", "singleQuote": false } } | Setting | Purpose | |
-------------------- | ------------------------------------------------ | | `proseWrap: always` |
Wrap long lines in Markdown | | `singleQuote: false` | Use double quotes for written text (prose
style) | 🔤 TypeScript JSX (.tsx) { "files": "*.tsx", "options": { "jsxSingleQuote": false } }

| Setting                 | Purpose                                                |
| ----------------------- | ------------------------------------------------------ |
| `jsxSingleQuote: false` | Enforces double quotes in JSX (consistent React style) |

🧠 Final Tips Run Prettier on your whole project: npx prettier --write . Check formatting without
writing: npx prettier --check . Use with ESLint for best results. Prettier handles style, ESLint
handles code quality.
