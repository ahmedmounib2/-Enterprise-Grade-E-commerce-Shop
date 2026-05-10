# Claude + Figma Integration Setup

Automated Hi‑Fi mockup creation, Lo‑Fi wireframing, prototyping, case study pages, and
design‑to‑code conversion using Claude Desktop + Claude Code CLI connected to Figma via MCP (Model
Context Protocol).

---

## Architecture Overview

```
┌─────────────────┐     stdio      ┌──────────────────┐   WebSocket   ┌────────────────┐
│  Claude Desktop │ ────────────▶  │  MCP Go Server   │ ◀───────────   │  Figma Plugin  │
│  (Cowork chat)  │                │ (127.0.0.1:1994) │                │ (inside Figma) │
└─────────────────┘                └──────────────────┘                └────────────────┘
```

Three components must be running simultaneously:

1. **Claude Desktop** launches the MCP server via `npx`
2. **MCP Go server** listens on `127.0.0.1:1994`
3. **Figma Plugin** connects to the server via WebSocket and manipulates the canvas

No Figma API token, no rate limits, no paid seat required. All communication happens locally.

---

## Part 1 — One-Time Setup

### A. Install Node.js on Windows

```powershell
winget install OpenJS.NodeJS.LTS
```

Restart your PC after installation.

### B. Download and Register the Figma Plugin

1. Download `plugin.zip` from
   [github.com/vkhanhqui/figma-mcp-go/releases](https://github.com/vkhanhqui/figma-mcp-go/releases)
2. Unzip to a permanent folder (e.g., `C:\Users\Admin\figma-mcp-plugin\`)
3. Open **Figma Desktop**
4. Go to **Plugins → Development → Import plugin from manifest**
5. Select `manifest.json` from the unzipped folder
6. The plugin "Figma MCP Go" is permanently registered in your development plugins list

### C. Configure Claude Desktop

1. Press `Win + R`, paste this path, and hit Enter:

   ```
   %LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude
   ```

2. Open `claude_desktop_config.json` in VS Code or Notepad.

3. Replace the contents with (validate at [jsonlint.com](https://jsonlint.com) first):

```json
{
  "preferences": {
    "coworkScheduledTasksEnabled": true,
    "ccdScheduledTasksEnabled": true,
    "sidebarMode": "task",
    "coworkWebSearchEnabled": true,
    "epitaxyPrefs": {
      "starred-local-code-sessions": [],
      "starred-cowork-spaces": [],
      "starred-session-groups": [],
      "dframe-local-slice": {
        "pinnedOrder": [],
        "customGroupAssignments": {},
        "customGroupOrder": {}
      }
    }
  },
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@vkhanhqui/figma-mcp-go"]
    }
  }
}
```

1. Save the file.

### D. Verify the Setup

1. Open Figma Desktop → your target file
2. Run the plugin: **Plugins → Development → Figma MCP Go**
3. Wait until the plugin window shows **"Connected"** on `127.0.0.1:1994`
4. Start Claude Desktop
5. New chat → type:

   ```
   What MCP tools do you have available? List only tools starting with "figma_"
   ```

6. You should see **72 tools** listed (`create_frame`, `create_text`, `set_fills`,
   `set_auto_layout`, `set_reactions`, etc.)

---

## Part 2 — Every-Session Routine

Repeat this each time you want Claude to create or modify designs:

1. **Open Figma Desktop** → open your target file
2. **Run the plugin**: Plugins → Development → Figma MCP Go
3. **Wait** for the plugin window to show **"Connected"**
4. **Start Claude Desktop** (or restart if already running)
5. **New chat** → verify tools:

   ```
   What MCP tools start with "figma_"?
   ```

6. **Paste your design prompt** (Hi‑Fi, Lo‑Fi, prototype, case study)

> ⚠️ **Keep the plugin window open** the entire time Claude is drawing. Minimize it, but don't close
> it. Closing the plugin kills the WebSocket bridge.

> ⚠️ **Order matters**: If the plugin shows "Disconnected" after starting Claude, close the plugin,
> restart Claude Desktop, then re-open the plugin.

---

## Part 3 — Troubleshooting

| Symptom                                             | Fix                                                                                                                                |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Plugin shows `Disconnected` while Claude is running | Close plugin window → wait 3 seconds → Plugins → Development → Figma MCP Go                                                        |
| Claude shows "MCP figma: Server disconnected"       | Quit Claude → open plugin → restart Claude                                                                                         |
| Claude has no `figma_` tools                        | Ensure plugin window is open and shows "Connected", then restart Claude                                                            |
| `npx @vkhanhqui/figma-mcp-go` fails with 404        | Package may have been unpublished. Check [GitHub releases](https://github.com/vkhanhqui/figma-mcp-go/releases) for the latest name |
| Reinstalled Windows                                 | Repeat Part 1 from scratch                                                                                                         |
| JSON config error                                   | Validate at [jsonlint.com](https://jsonlint.com) — missing commas or extra braces are the most common issues                       |

---

## Part 4 — Figma Token Management

`@vkhanhqui/figma-mcp-go` does **not** use a Figma personal access token — it communicates locally
via the plugin bridge. No token renewal needed.

If you ever switch to a token‑based MCP server for the CLI, tokens expire every **90 days**. Renewal
steps:

1. Go to [Figma Settings → Security → Personal access tokens](https://www.figma.com/settings)
2. Delete the old token, generate a new one
3. Update the token in Claude Desktop config (`FIGMA_ACCESS_TOKEN` field)
4. Update the CLI: `claude mcp remove figma` →
   `claude mcp add figma -e FIGMA_ACCESS_TOKEN=NEW_TOKEN -- npx -y <package-name>`
5. Restart Claude Desktop

Set a recurring calendar reminder 3 days before expiration.

---

## Part 5 — Claude Code CLI for Design-to-Code

The CLI's built‑in `claude.ai Figma` connector handles design‑to‑code extraction (read‑only). No
additional MCP server needed.

### Verify CLI Figma access

```bash
claude mcp list
```

Look for: `claude.ai Figma: https://mcp.figma.com/mcp - ✓ Connected`

### Point Claude to a specific frame

**By frame name:**

```
Convert the Figma frame named "ProductDetail – Mobile – HiFi" from my file at
https://www.figma.com/design/FILE_ID/ into a React Native screen. Use existing
mobile component patterns and theme tokens.
```

**By node ID** (right‑click frame → Copy/Paste as → Copy link to selection):

```
Convert the Figma frame at node-id 1234:5678 from my file at
https://www.figma.com/design/FILE_ID/ into a React component.
```

**Full prototype flow:**

```
From my Figma file at https://www.figma.com/design/FILE_ID/, find all frames
in the "Desktop – Customer Journey" prototype flow. Convert them, in order,
into React components under frontend/src/pages/checkout/.
```

### Optional: Token-Based MCP Server for CLI

The built-in `claude.ai Figma` connector is sufficient for most design-to-code work. If you need a
dedicated token-based server (e.g., `figma-developer-mcp`) for heavier extraction or write
capabilities from the CLI:

**Setup:**

```bash
claude mcp add figma -e FIGMA_ACCESS_TOKEN=figd_YOUR_TOKEN -- npx -y figma-developer-mcp
```

**Verify:**

```bash
claude mcp list
```

Look for: `figma: npx -y figma-developer-mcp - ✓ Connected`

**Token renewal (every 90 days):**

```bash
claude mcp remove figma
claude mcp add figma -e FIGMA_ACCESS_TOKEN=NEW_TOKEN -- npx -y figma-developer-mcp
```

**Note:** `@vkhanhqui/figma-mcp-go` (used for Desktop) does not work in the CLI — it requires a
Figma plugin bridge that only runs inside Figma Desktop. The CLI uses the built-in connector or a
token-based server instead.

---

## Part 6 — Desktop vs CLI Usage

| Task                              | Desktop (Cowork) | CLI (Terminal)                |
| --------------------------------- | ---------------- | ----------------------------- |
| Create Hi‑Fi / Lo‑Fi / prototypes | ✅ Use this      | 🚫 Not suited                 |
| Case study pages                  | ✅ Use this      | 🚫 Not suited                 |
| Design‑to‑code conversion         | Works            | ✅ Faster, native to dev flow |
| Code audit / review               | Works            | ✅ Better                     |
| Quick single‑frame extraction     | Overkill         | ✅ Paste node ID, get code    |

**Rule**: Create designs in Desktop, generate code in CLI. Both share `CLAUDE.md` rules.

---

## Part 7 — Model Selection Strategy

| Task                   | Model                  | Effort | Why                             |
| ---------------------- | ---------------------- | ------ | ------------------------------- |
| Audit / deep analysis  | `opusplan` or Opus 4.7 | High   | Multi‑file reasoning, read‑only |
| Feature implementation | Sonnet 4.6             | Medium | Well‑defined CRUD + schema      |
| State machine logic    | Sonnet 4.6             | Medium | Straightforward                 |
| Figma / UI work        | Sonnet 4.6             | Medium | Vision + creative coding        |
| Quick fixes, renames   | Haiku 4.5              | Low    | Speed over depth                |

**Switch models:**

- **Desktop**: Click the model name next to the send button → dropdown → More models
- **CLI**: Type `/model` → select from menu
- **CLI permanent default** (add to `~/.zshrc`):

  ```bash
  export ANTHROPIC_MODEL="claude-sonnet-4-6"
  ```

**Quota strategy**: Default to Sonnet for daily work. Reach for `opusplan` for deep analysis. Use
pure Opus only 2–3 times/week. This cuts quota burn by ~40%.

---

## Part 8 — Bypassing Approval Gates for Implementation Tasks

If your project's `CLAUDE.md` has strict approval gates (section 14), start implementation sessions
with:

```
Proceed with the following task. As a temporary measure, skip the approval gates
listed in the project instructions – I have reviewed the scope and grant full
approval. If you encounter a genuine security or data‑loss risk, flag it and stop.
```

---

## Part 9 — Security: `.claudeignore`

Claude Desktop and CLI can read all files in your repo, including `.env`. Protect secrets:

Create `.claudeignore` at the repo root:

```
.env
.env.*
*.pem
*.key
```

Keep a clean `.env.example` visible so Claude can audit variable names and types without seeing real
values.

---

## Part 10 — Available Figma MCP Tools (72 total)

**Page operations**: `add_page`, `delete_page`, `rename_page`, `navigate_to_page`, `get_pages`

**Frame & shape creation**: `create_frame`, `create_rectangle`, `create_ellipse`, `create_text`,
`create_component`, `create_section`

**Styling**: `set_fills`, `set_strokes`, `set_opacity`, `set_corner_radius`, `set_effects`,
`set_blend_mode`, `set_auto_layout`, `set_constraints`, `set_text`

**Style management**: `create_paint_style`, `create_text_style`, `create_effect_style`,
`create_grid_style`, `apply_style_to_node`, `delete_style`, `get_styles`

**Variables**: `create_variable`, `create_variable_collection`, `bind_variable_to_node`,
`set_variable_value`, `get_variable_defs`, `add_variable_mode`, `delete_variable`

**Prototype**: `set_reactions`, `remove_reactions`, `get_reactions`

**Node operations**: `clone_node`, `move_nodes`, `resize_nodes`, `rotate_nodes`, `reorder_nodes`,
`reparent_nodes`, `group_nodes`, `ungroup_nodes`, `lock_nodes`, `unlock_nodes`, `delete_nodes`,
`rename_node`, `swap_component`, `detach_instance`, `batch_rename_nodes`

**Search & scan**: `search_nodes`, `scan_nodes_by_types`, `scan_text_nodes`, `find_replace_text`,
`get_node`, `get_nodes_info`, `get_selection`, `get_viewport`

**Export & metadata**: `get_screenshot`, `save_screenshots`, `export_frames_to_pdf`,
`export_tokens`, `get_design_context`, `get_document`, `get_metadata`, `get_fonts`,
`get_local_components`, `get_annotations`

**Images**: `import_image`

---

## Related Files

- `CLAUDE.md` — Project instructions for Claude (shared by Desktop and CLI)
- `.claudeignore` — Files excluded from Claude's view
- `frontend/src/index.css` — Design tokens (colors, spacing, typography)
- `tailwind.config.js` — Tailwind theme configuration
