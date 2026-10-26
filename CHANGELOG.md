# Changelog

The version number mirrors the Pi release the skill documents.

## 1.4.0 - 2026-10-21

Re-verified the whole skill against Pi 1.4.0 and updated the 1.0-era content.
The frontmatter now carries the release as the `version` entry under `metadata`,
so the skill, the tag, and Pi's release number stay in one series.

### Second pass, same Pi version

A philosophy pass rather than a version pass: audit the skill against its own
contract — let a user make their agent change, debug, and tune their own Pi —
and fill what the 1.0-era content never covered.

- Added a mechanism-selection table to `SKILL.md`: the user's goal → context
  file, system prompt, skill, prompt template, settings, keybindings or theme,
  extension, package, or an isolated experiment. Choosing the surface is the
  first decision, and the skill previously jumped straight to file paths.
- Added diagnosis triage for the "it doesn't work" requests the philosophy
  promises to serve: which layer failed (scope, trust, load, override) for
  settings, tools, resources, MCP, and models, plus `/debug` and `pi-debug.log`.
- Added a read-only configuration audit checklist: shadowed scopes, legacy keys,
  misplaced secrets, unreviewed executable input, cost and context pressure, and
  stale resumed-session state.
- Added context and system-prompt guidance for `AGENTS.md`,
  `AGENTS.override.md`, `SYSTEM.md`, and `APPEND_SYSTEM.md`, including the trust
  asymmetry between context files and project system-prompt files.
- Moved the release marker under `metadata`. A top-level `version` key is outside
  the Agent Skills frontmatter and fails skill-creator's `quick_validate.py`;
  Pi's loader tolerates it, portable consumers do not.
- Added routes that were missing entirely: non-interactive runs (`-p`,
  `--mode json`/`rpc`, `--export`, SDK), local llama.cpp models, Bash alias
  setup through `shellCommandPrefix`, and migration from another agent.

### Corrected

- `retry.provider.*` is nested under `retry` (`retry.provider.timeoutMs`,
  `maxRetries`, `maxRetryDelayMs`); the old flat `provider.*` names were wrong.
- `--tools` now accepts `+name`/`-name` modifiers, which the 1.0 version said it
  did not. Plain names still replace the selection.
- `models.json` `!command` values run at request time and are not cached, not
  "cached for the process" as previously written.
- `compaction.reserveTokens` only decides when compaction starts; since 1.4.0
  summary requests use the model's output limit, and `generateSummary()` ignores
  its `reserveTokens` argument.
- Theme facts: the theme schema moved to `https://pi.dev/schemas/theme.schema.json`,
  theme objects are strict, `theme` accepts a `"light/dark"` pair, and
  `--use-theme` picks a theme for one run.

### Added

- Built-in extension names (`builtin:mcp`, `builtin:llama.cpp`,
  `builtin:codemode`, `builtin:tool-search`) and how to disable or replace them.
- `$schema` support for settings, models, keybindings, and themes, plus the
  bundled copies in `<pi-package>/schemas/`.
- MCP: partial project overrides (`enabled`/`exposure`/`toolExposure` only),
  `autoEnableCodemode`, `type` field and the `codemode-deferred` alias,
  `toolExposure` pattern precedence, the three resource tools, the 20 KB result
  truncation, `mcp.log`, `--no-mcp`, `--tools 'mcp__server__*'`, and the 1.2+
  `resources: true` shape of `pi mcp list --json`.
- OAuth: `clientRegistration: "cimd"`, `callbackPort`/`callbackUrl`, `scope`,
  and the RFC 9728/8414/9207 checks.
- Codemode: `// @options:` as a first line only, output and store limits, the
  two-worker pool with `PI_CODEMODE_MAX_WORKERS` and the 60-second wait,
  `==> text N/M <==` output framing, and the `models.*` API.
- Models: `pi auth check|print-api-key|print-bearer-token`, the classifier
  provider matrix (including GPT-6 Luna's API-key-only rule), image models,
  `inputLimits`, `promptCache`/`cacheWarming`, and
  `samplingParamsByThinkingLevel`.
- Extensions: `exposure: "model-only"`, `namespace.aliases`, `reachesTools`,
  `prepareLoadout`, `registerToolRenderer`, `registerMcpServer`/
  `unregisterMcpServer`/`getMcpServers`, nested `ctx.executeTool()` ids,
  `resources_discover`, and the `before_agent_start` union on `messageType`.
- Trust details: `trust.json`, the `project_trust` extension event, and context
  files loading regardless of trust.
- Terminal and Windows: `%ProgramFiles(x86)%` bash lookup, the PowerShell
  discovery order, capability overrides (`terminal.*` and `PI_*`), OSC 7501
  program status, and the Windows/WSL key defaults.
- `sessionShareDestination`, `/import`, `/name`, and the session storage
  precedence order.
