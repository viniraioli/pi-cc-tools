# Code Context

## Files Retrieved
1. `extensions/index.ts` (lines 69-170) - settings schema/cache and the existing `writeSettingsKey()` persistence pattern in pi-cc-tools.
2. `extensions/index.ts` (lines 3865-4074) - current slash command style for `cc-tools`, `cc-theme`, and `cc-spinner`; shows completions, argument parsing, preview/status flows, and settings writes.
3. `node_modules/@mariozechner/pi-coding-agent/dist/core/extensions/types.d.ts` (lines 55-98) - extension UI APIs: `setStatus()`, `setFooter()`, `custom()`, `select()`, `input()`, `notify()`.
4. `node_modules/@mariozechner/pi-coding-agent/dist/core/footer-data-provider.d.ts` (lines 1-34) - confirms footer data provider exposes only git branch, provider count, and extension status map.
5. `node_modules/@mariozechner/pi-coding-agent/dist/modes/interactive/components/footer.js` (lines 1-195) - default footer rendering logic, including how extension statuses are sanitized, sorted, joined, and truncated.
6. `node_modules/@mariozechner/pi-coding-agent/examples/extensions/custom-footer.ts` (lines 1-60) - minimal example of replacing the built-in footer with `ctx.ui.setFooter()`.
7. `node_modules/@mariozechner/pi-coding-agent/examples/extensions/status-line.ts` (lines 1-31) - minimal example of extension-owned footer status items via `ctx.ui.setStatus(key, text)`.
8. `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-extmgr/src/utils/status.ts` (lines 33-79) - extmgr footer segment source and exact key `extmgr`.
9. `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-mcp-adapter/init.ts` (lines 126-128, 280-289, 316-319) - main MCP footer segment source and exact key `mcp`.
10. `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-mcp-adapter/commands.ts` (lines 169-192) - transient auth footer segment source and exact key `mcp-auth`.
11. `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-mcp-adapter/commands.ts` (lines 287-296, 359-417) - concrete example of an interactive settings panel built with `ctx.ui.custom(..., { overlay: true })`.
12. `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-subagents/src/slash/slash-commands.ts` (lines 200-222, 337-355) - another real footer segment source, exact key `subagent-slash`, and lifecycle clearing behavior.

## Key Code

### 1) pi-cc-tools command/settings pattern
- Settings are plain JSON in `.pi/settings.json`, read from project first then home, with a short cache:
  - `extensions/index.ts:117-138`
- Writes currently go to `HOME/.pi/settings.json` via `writeSettingsKey(key, value)`:
  - `extensions/index.ts:151-169`
- Existing commands follow a simple pattern:
  - register via `pi.registerCommand(...)`
  - optional `getArgumentCompletions(prefix)`
  - parse `args.trim().toLowerCase()`
  - preview/status with `ctx.ui.notify(...)`
  - persist with `writeSettingsKey(...)`
  - examples: `cc-tools`, `cc-theme`, `cc-spinner` in `extensions/index.ts:3867-4074`

### 2) Footer/status APIs that matter
- `ctx.ui.setStatus(key, text)` adds/removes one extension-owned status entry.
- `ctx.ui.setFooter(factory)` replaces the entire built-in footer.
- `ctx.ui.custom(...)` can open a focused overlay panel.
- Source: `node_modules/@mariozechner/pi-coding-agent/dist/core/extensions/types.d.ts:55-98`

### 3) How default footer renders extension statuses
From `node_modules/@mariozechner/pi-coding-agent/dist/modes/interactive/components/footer.js:188-195`:
- reads `footerData.getExtensionStatuses()`
- sorts entries alphabetically by status key
- sanitizes each text to one line
- joins them with spaces
- truncates the final status line to terminal width

Important constraint:
- the default footer has no per-segment visibility setting.
- selective hide/show requires either:
  1. core changes to footer/status APIs, or
  2. a custom footer that reproduces current default behavior and filters keys before rendering.

### 4) Known status keys likely visible in the screenshot
Concrete, attested keys found:
- `extmgr`
  - text like `N pkgs • ... • N updates`
  - source: `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-extmgr/src/utils/status.ts:44-75`
- `mcp`
  - transient: `MCP: connecting to ...`
  - steady state: `MCP: X/Y servers`
  - source: `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-mcp-adapter/init.ts:126-128,280-289,316-319`
- `mcp-auth`
  - transient during OAuth auth flow: `Authenticating <server>...`
  - source: `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-mcp-adapter/commands.ts:169-192`
- `subagent-slash`
  - transient progress like `running...` or `N tools ... | Ctrl+O live detail`
  - source: `C:/Users/Vinicios/.pi/agent/npm/node_modules/pi-subagents/src/slash/slash-commands.ts:200-222,337-355`

Likely screenshot interpretation:
- package/update counters → `extmgr`
- MCP server connectivity/auth text → `mcp` and sometimes `mcp-auth`
- slash/subagent progress text → `subagent-slash`
- any other segment would come from another extension calling `ctx.ui.setStatus(key, text)`; the core footer does not tag origin visually.

## Architecture

### Smallest implementation path
Implement `/cc-statusline` inside `pi-cc-tools`, not in core pi.

Why this is the smallest path:
- pi-cc-tools already has:
  - settings read/write helpers (`extensions/index.ts:117-169`)
  - runtime slash command registration (`extensions/index.ts:3867-4074`)
- pi already exposes enough API to replace the footer (`setFooter`) and inspect status entries (`footerData.getExtensionStatuses()`), without touching pi core.
- the only missing feature is selective filtering, which can be done in a custom footer.

### Recommended shape
1. Add new setting in pi-cc-tools, e.g. `hiddenFooterStatusKeys?: string[]`.
2. On extension load/session start, call `ctx.ui.setFooter(...)` with a footer factory that copies current default footer behavior from:
   - `node_modules/@mariozechner/pi-coding-agent/dist/modes/interactive/components/footer.js:60-195`
3. In that custom footer, filter:
   - `Array.from(footerData.getExtensionStatuses().entries())`
   - remove keys present in `hiddenFooterStatusKeys`
   - keep the rest of the default logic unchanged.
4. Add `/cc-statusline` command to edit that setting interactively.

### UI recommendation
Use a two-level approach:
- **MVP / lowest-risk:** `ctx.ui.select()` loop
  - options: `Toggle extmgr`, `Toggle mcp`, `Toggle mcp-auth`, `Toggle subagent-slash`, `Show current`, `Reset`, `Done`
  - simplest to ship; consistent with extmgr’s command UX.
- **Better UX, still bounded:** `ctx.ui.custom(..., { overlay: true })`
  - checkbox-like panel modeled after pi-mcp-adapter’s overlay panels in `commands.ts:287-296` and `359-417`
  - shows all currently known keys from `footerData.getExtensionStatuses()` plus known presets (`extmgr`, `mcp`, `mcp-auth`, `subagent-slash`)
  - supports toggle, preview, save/reset

### Command contract recommendation
`/cc-statusline`
- no args: open interactive panel
- `status`: notify current visible/hidden keys
- `show <key>` / `hide <key>`: direct CLI control
- `reset`: clear hidden list
- completions should include known keys and subcommands

### Data model recommendation
Prefer key-based hiding, not string matching.

Example setting shape:
- `hiddenFooterStatusKeys: string[]`

Reason:
- the footer sorts by key already
- extension text is unstable (`MCP: connecting...`, `MCP: 2/3 servers`, etc.)
- keys are stable and directly attested in code

## Start Here
Open `node_modules/@mariozechner/pi-coding-agent/dist/modes/interactive/components/footer.js` first.

Why:
- it defines the exact built-in footer behavior you need to preserve.
- the implementation decision hinges on one fact there: extension statuses are rendered only as a flat sorted map, so selective hide/show requires a custom footer wrapper.

## Concrete implementation advice
- **Recommended severity: low** for adding the command in pi-cc-tools only.
- **Main risk: medium** if you clone footer logic from pi core, because upstream footer changes could drift.
  - Mitigation: keep the copied renderer as close as possible to `footer.js:60-195`, and limit the customization to filtering `extensionStatuses`.
- **Do not** try to mutate other extensions’ `setStatus()` calls; there is no interception API surfaced here.
- **Do not** hide by matching rendered text; use keys.
- **If long-term core changes are allowed later:** the cleaner API would be a core footer setting/hook for filtering `getExtensionStatuses()` before default rendering. But that is not the smallest path.

## review-findings
- **Info:** `pi-cc-tools` already has the exact persistence and command scaffolding needed for `/cc-statusline` (`extensions/index.ts:117-169`, `3867-4074`).
- **Info:** Default pi footer renders extension statuses from `footerData.getExtensionStatuses()`, alphabetically by key, on a single truncated line (`node_modules/@mariozechner/pi-coding-agent/dist/modes/interactive/components/footer.js:188-195`).
- **Info:** Confirmed footer status producers relevant to the screenshot are `extmgr`, `mcp`, `mcp-auth`, and `subagent-slash` (`pi-extmgr/src/utils/status.ts:72`, `pi-mcp-adapter/init.ts:127,289,318`, `pi-mcp-adapter/commands.ts:170,191`, `pi-subagents/src/slash/slash-commands.ts:202,221,338,354`).
- **Risk:** Selective segment hiding cannot be done with the stock footer alone; implementing it in an extension requires `ctx.ui.setFooter(...)` and a copied/wrapped footer renderer (`types.d.ts:77-85`, `examples/extensions/custom-footer.ts:18-60`).

## residual-risks
- Footer renderer drift if pi core changes `FooterComponent` later.
- Unknown third-party status keys will only appear in the interactive UI after they have emitted at least one status, unless you add manual free-text show/hide support.
- Current pi-cc-tools write path persists only to `HOME/.pi/settings.json`; if you want project-local statusline preferences, `writeSettingsKey()` would need a scope-aware enhancement.

```acceptance-report
{
  "criteriaSatisfied": [
    {
      "id": "criterion-1",
      "status": "satisfied",
      "evidence": "Included concrete findings with exact file paths, cited line ranges, attested status keys (extmgr/mcp/mcp-auth/subagent-slash), and implementation guidance centered on extensions/index.ts and pi footer APIs."
    }
  ],
  "changedFiles": [],
  "testsAddedOrUpdated": [],
  "commandsRun": [
    {
      "command": "find/grep/read across extensions, pi-coding-agent dist/examples, and installed pi-extmgr/pi-mcp-adapter/pi-subagents packages",
      "result": "passed",
      "summary": "Mapped command/settings patterns, footer APIs, default footer renderer, and real status producers."
    },
    {
      "command": "rg -n \"setStatus\\(|extmgr|mcp-adapter|footer\" $HOME/.pi/agent -S",
      "result": "passed",
      "summary": "Confirmed installed extensions emitting footer statuses and their exact status keys."
    }
  ],
  "validationOutput": [
    "Verified output file written: C:/Users/Vinicios/Desktop/pi-cc-tools/scout-statusline.md"
  ],
  "residualRisks": [
    "Custom footer implementation in pi-cc-tools must track upstream pi FooterComponent changes.",
    "Unknown extension status keys may require dynamic discovery or manual entry in the settings UI.",
    "Current pi-cc-tools settings writer targets HOME/.pi/settings.json only, which may not match desired project-local scope."
  ],
  "noStagedFiles": true,
  "notes": "No repository files were edited; findings were written only to the requested scout artifact."
}
```