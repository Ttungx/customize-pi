# Configuration reference

Use this file for paths, scope and precedence, project trust, settings and
their defaults, tool selection, compaction, reload behavior, and environment
variables. Verify exact types in the installed `docs/configuration.md`,
`docs/settings.md`, and `schemas/settings.schema.json`.

## Scope and paths

- Agent directory: `~/.pi/agent`, overridden by `PI_CODING_AGENT_DIR`.
- User settings: `<agent-dir>/settings.json`.
- Project settings: `.pi/settings.json`, loaded after project trust.
- User MCP: `<agent-dir>/mcp.json`; project MCP: `.pi/mcp.json` (trusted only).
- User models: `<agent-dir>/models.json`.
- Credentials and runtime state: `auth.json`, `mcp-auth.json`, `trust.json`,
  `mcp.log` (and `mcp.log.1`) inside the agent directory. Keep them private.
- Keybindings: `<agent-dir>/keybindings.json`.
- Sessions: `<agent-dir>/sessions/`, overridden by `sessionDir`,
  `PI_CODING_AGENT_SESSION_DIR`, or `--session-dir`.
- Global resources: `<agent-dir>/{extensions,skills,prompts,themes}/`.
- Project resources: `.pi/{extensions,skills,prompts,themes}/`.
- Context and prompt files: `AGENTS.override.md`, `AGENTS.md`, `CLAUDE.md`,
  `SYSTEM.md`, `APPEND_SYSTEM.md` in the agent directory; `AGENTS.md`,
  `CLAUDE.md`, `.pi/SYSTEM.md`, `.pi/APPEND_SYSTEM.md` in the project.

Project settings override user settings. Resource lists are combined. A project
package normally replaces a user package with the same identity; `autoload:
false` turns the project entry into a filtering delta instead. For
`SYSTEM.md`/`APPEND_SYSTEM.md`, the trusted project file replaces the
agent-directory file rather than being appended to it.

Relative resource paths resolve from the agent directory in user settings and
from the project `.pi` directory in project settings; `~` and absolute paths
work in both. Settings and package declarations accept `!pattern`, `+path`, and
`-path` entries for glob exclusion and exact inclusion/exclusion.

## Project trust

Trust gates loading of `.pi/settings.json`, `.pi/mcp.json`, `.pi/extensions`,
`.pi/skills`, `.pi/prompts`, `.pi/themes`, `.pi/SYSTEM.md`,
`.pi/APPEND_SYSTEM.md`, and project `.agents/skills` found in the working
directory or an ancestor. A bare `.pi` directory does not need trust.

The decision order is: `--approve`/`--no-approve`, then an extension handling the
`project_trust` event, then a saved decision for the closest parent directory in
`trust.json`, then `defaultProjectTrust` (`ask`, `always`, `never`; global
setting only). `/trust` saves a decision for future processes.

Two footguns worth stating out loud:

- `sessionDir` from project settings is read while selecting a session, before
  trust is resolved. Declining trust cannot undo that lookup.
- Context files (`AGENTS.md`, `CLAUDE.md`, and overrides) load regardless of
  trust. Treat instructions in an unfamiliar folder as untrusted input even
  when trust is declined.

Print, JSON, and RPC modes cannot show the trust prompt. An automated run needs
`--approve`, `--no-approve`, or a saved decision; `defaultProjectTrust: "ask"`
skips protected resources in those modes.

Trust is not a sandbox. Enabled tools, extensions, MCP servers, and child
processes keep the OS permissions of the Pi process.

## Settings groups and defaults

Add `"$schema": "https://pi.dev/schemas/settings.schema.json"` for editor
completion; the same schema ships in `<pi-package>/schemas/settings.schema.json`.

### Model and thinking

| Setting | Default | Notes |
|---|---|---|
| `defaultProvider`, `defaultModel` | automatic | Startup selection |
| `defaultThinkingLevel` | `medium` | `off`…`max` |
| `modelThinkingLevels` | none | Per `provider/modelId` startup level |
| `thinkingBudgets` | built-in | Token budgets for minimal/low/medium/high |
| `enabledModels` | all | Patterns for startup selection and cycling |
| `hideThinkingBlock` | `false` | Hide thinking in the transcript |
| `showCacheMissNotices` | `false` | Cache misses, warming, compaction usage |
| `cacheWarming` | `streaming` | `off`/`streaming`/`idle`; global only, costs money |

### Tools

| Setting | Default | Notes |
|---|---|---|
| `defaultTools` | `read`, `bash`, `edit`, `write` | Plain names replace; `+name`/`-name` adjust |
| `codemode.mode` | `on` | `only` hides active tools and routes them through scripts |
| `codemode.inlineBudget` | `3000` | Estimated tokens for inline tool declarations |

Built-in tools: `read`, `bash`, `powershell`, `edit`, `write`, `grep`, `find`,
`ls`. Built-in extensions register `codemode` and `tool_search` inactive; the
MCP extension activates them when a server needs them. An empty
`defaultTools: []` disables all built-in tools but not extension or SDK tools.

### Interaction

`steeringMode` and `followUpMode` (`one-at-a-time` by default), `externalEditor`,
`doubleEscapeAction` (`tree`), `treeFilterMode` (`default`),
`defaultProjectTrust` (`ask`, global only).

### Sessions and context

`sessionDir`, `sessionShareDestination` (`auto` uses Radius when authenticated
and otherwise a private gist; `gist` and `radius` force one destination).
`/share`, `/export`, and `/bug` all need a review pass before upload.

### Compaction and branch summaries

| Setting | Default | Notes |
|---|---|---|
| `compaction.enabled` | `true` | Automatic compaction |
| `compaction.reserveTokens` | `16384` | Free-context threshold below which compaction starts |
| `compaction.keepRecentTokens` | `20000` | Recent tokens kept without summarizing |
| `compaction.modelOverrides` | none | Per `provider/modelId` token settings |
| `branchSummary.reserveTokens` | `16384` | Branch summary reservation |
| `branchSummary.skipPrompt` | `false` | Default to no summary |

Since 1.4.0 summaries use the model's output limit instead of a fixed cap, and
`reserveTokens` only decides when compaction starts. `generateSummary()` ignores
its `reserveTokens` argument, so an extension that passed one is not shortening
summaries. Auto-compaction triggers when
`contextTokens > contextWindow - reserveTokens`; per-model overrides resolve
field by field, and invalid token values error when read.

### Terminal, images, markdown

| Setting | Default | Notes |
|---|---|---|
| `theme` | `system` | Also accepts `"light/dark"` |
| `quietStartup` | `false` | `"header"` keeps version/key hints only |
| `tuiMode` | `fullscreen` | `regular` restores native scrollback |
| `fullscreenExitOutput` | `transcript` | Or `resume-hint` |
| `fullscreenScrollbar` | `auto` | `auto`/`always`/`hidden` |
| `fullscreenCopyOnSelect` | `true` | Copy on selection |
| `fullscreenWheelScrollLines` | `auto` | Or 1–100 |
| `editorPaddingX`, `outputPad` | `0`, `1` | Horizontal padding |
| `autocompleteMaxVisible` | `5` | 3–20 |
| `showHardwareCursor` | `false` | Uses the terminal cursor only |
| `terminal.showImages`, `imageWidthCells`, `clearOnShrink` | `true`, `60`, `false` | Inline images |
| `terminal.showTerminalProgress` | `false` | OSC 9;4 progress |
| `terminal.hyperlinks`, `images`, `trueColor` | `auto` | Override detection |
| `images.autoResize`, `blockImages` | `true`, `false` | Model-bound images |
| `markdown.codeBlockIndent`, `mermaid` | `"  "`, `streaming` | Rendering |

### Network and retries

| Setting | Default | Notes |
|---|---|---|
| `transport` | `auto` | Or `sse`/`websocket`/`websocket-cached` |
| `httpProxy` | none | Global only; sets `HTTP_PROXY`/`HTTPS_PROXY` |
| `httpIdleTimeoutMs` | `300000` | `0` disables |
| `websocketConnectTimeoutMs` | `15000` | `0` disables |
| `retry.enabled`, `maxRetries` | `true`, `3` | Agent-level retries |
| `retry.baseDelayMs`, `retry.maxAgentDelayMs` | `2000`, `60000` | Backoff |
| `retry.provider.timeoutMs` | `httpIdleTimeoutMs` | Provider request timeout |
| `retry.provider.maxRetries` | `0` | Keep at 0 unless required |
| `retry.provider.maxRetryDelayMs` | `60000` | `0` removes the cap |

Provider-level retries can delay Pi's own handling of quota and usage-limit
errors, which is why the default is off.

### Shell, resources, updates

`shellPath` (leading `~` allowed), `shellCommandPrefix`, `npmCommand` (argv
array, default `npm`). Resources: `packages`, `extensions`, `skills`, `prompts`,
`themes`, `enableSkillCommands` (`true`). Updates and telemetry:
`collapseChangelog`, `enableInstallTelemetry`, `enableAnalytics`,
`warnings.anthropicExtraUsage`.

`queueMode` and `websockets` are legacy names that Pi migrates to
`steeringMode` and `transport`; write the new names.

## Tool selection with `defaultTools` and `--tools`

- Plain names replace the inherited selection. `+name` adds, `-name` removes.
  In one list, plain names form the selection and the modifiers apply in order.
- A CLI list of only `+name`/`-name` entries adjusts the resolved
  `defaultTools` for that run, including on `/reload`.
- `--tools` plain names accept `*` patterns; `+`/`-` entries do not. Use
  `--exclude-tools` (`-xt`) for pattern-based exclusion.
- `--no-tools` (`-nt`, everything off) and `--no-builtin-tools` (`-nbt`, built-ins
  off, extension/SDK tools on) override settings.
- `--tools` does not remove MCP tools unless an entry starts with `mcp__`.
  `pi --tools read,codemode,'mcp__radius__*'` keeps only `radius` tools, and
  `--no-mcp` disables MCP for one run.
- `/reload` enables newly added tools. It does not disable ones removed from the
  setting, and it does not re-enable a tool the user disabled in-session.

## Reload matrix

| Change | Action |
|---|---|
| Settings, keybindings, extensions, skills, prompts, themes, context files | `/reload` |
| Active user theme file (`<agent-dir>/themes/<name>.json`) | Hot-reloads |
| `models.json` | `/model` re-reads it |
| MCP file changes outside the session | `/reload` |
| `/mcp` enable, disable, exposure, reconnect | Applies to the session |
| Saved project-trust decision | New Pi process |
| Default model/thinking for new sessions | New sessions |
| Tools removed in `defaultTools` or disabled in-session | Restart to restore |

`/reload` also reloads the extension runtime: code after `await ctx.reload()`
belongs to the old runtime and must not reuse session-bound state.

## Environment variables worth knowing

| Variable | Purpose |
|---|---|
| `PI_CODING_AGENT_DIR` | Agent directory |
| `PI_CODING_AGENT_SESSION_DIR` | Session storage; `--session-dir` wins |
| `PI_OFFLINE` | Disable automatic network activity |
| `PI_CODEMODE_MAX_WORKERS` | Codemode workers per process (`2` by default) |
| `PI_CLIPBOARD_OSC52` | Also copy via OSC 52 for remote clients |
| `PI_PROGRAM_STATUS` | Force `1`/`0` OSC 7501 status reporting |
| `PI_HYPERLINKS`, `PI_IMAGE_PROTOCOL`, `PI_TRUE_COLOR` | Capability overrides; settings win |
| `PI_CACHE_RETENTION` | `long` requests extended prompt caching |
| `PI_TUI_ESC_TIMEOUT` | Escape disambiguation delay in ms |
| `PI_SESSION_ID`, `PI_SESSION_FILE`, `PI_PROVIDER`, `PI_MODEL`, `PI_REASONING_LEVEL` | Exposed to tools and child processes |
| `HTTP_PROXY`, `HTTPS_PROXY` | Proxy for Pi-managed HTTP clients |

## Sessions

`--continue`, `--resume`, `--session`, `--session-id`, `--fork`, `--name`,
`--no-session`, and `--session-dir` cover storage and lookup. In-session, `/new`,
`/resume`, `/name`, `/session`, `/tree`, `/fork`, `/clone`, `/compact`,
`/import`, `/export`, and `/share` operate on the JSONL tree. Sessions are
grouped by working directory; the active branch is what the model sees, and
compaction appends a summary without deleting entries.
