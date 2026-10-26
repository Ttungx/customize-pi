# Changelog

The version number mirrors the Pi release the skill documents.

## 1.4.0 - 2026-10-21

Re-verified the whole skill against Pi 1.4.0 and updated the 1.0-era content.
The frontmatter now carries `version`, so the skill, the tag, and Pi's release
number stay in one series.

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
