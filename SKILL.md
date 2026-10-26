---
name: customize-pi
version: 1.4.0
description: "Customize the Pi coding agent (pi.dev) — settings.json, keybindings.json, mcp.json and MCP OAuth, models.json, providers/auth, codemode, extensions, skills, prompt templates, themes, Pi packages, project trust, sessions, and terminal/Windows setup. Use this skill whenever the user wants to change, add, fix, migrate, review, or explain any Pi configuration or Pi-native resource, including files under ~/.pi/agent/, a project .pi/, ~/.agents/skills/, or project .agents/skills/, and whenever a Pi command or behavior misbehaves (`pi mcp`, `/mcp`, `/settings`, `/model`, `/reload`, trust prompts, fullscreen TUI, shell selection, package loading). Covers Pi 1.4. Do not use it for the user's application code or for configuring a different agent such as Claude Code, Codex, or opencode."
---

# Customize Pi

Pi is a small terminal coding agent: the core stays lean and most behavior
lives in configuration and resources — `settings.json`, MCP servers,
extensions, skills, prompt templates, themes, and Pi packages. This skill
routes a customization request to the right file, the right scope, and the
right verification step, so the change lands where Pi will actually load it.

Verified against Pi **1.4.0** (2026-10-21). Pi moves fast and minor releases
carry breaking changes: 1.0 made fullscreen the default, 1.0.3 renamed the
`azure-openai-responses` provider to `azure`, 1.2 moved the theme schema and
made theme objects strict, 1.4 changed what `compaction.reserveTokens` means.
So treat every exact field, default, and command here as "verified at 1.4.0,
confirm again if `pi --version` reports something newer".

The frontmatter `version` mirrors the Pi release this skill documents, so the
installed skill, the git tag, and the Pi changelog stay readable as one series.
Bump it together with the references whenever Pi ships a release; a mismatch
between `version` and `pi --version` is the signal to re-check the details
before editing a user's configuration.

## Source of truth and progressive disclosure

1. Identify the resource being changed (a setting, an MCP server, a skill, a
   model, a package, a keybinding, ...).
2. Read the matching reference under `references/`. That is where the working
   detail lives; this file only routes and carries the invariants.
3. Confirm the exact schema, default, or version behavior against the installed
   docs, not from memory: `<pi-package>/docs/*.md`, where `<pi-package>` is the
   directory holding the running `@earendil-works/pi-coding-agent` package.
   Fall back to `https://pi.dev/docs/latest/` when the install has no docs
   (single-binary installs).
4. Inspect the existing file before editing. Preserve unrelated settings and
   unknown intent; never rewrite a whole JSON file to change one key.
5. State what will change and whether it needs `/reload` or a restart, before
   applying it, unless the user has already authorized the edit.

Two maintenance shortcuts are easy to forget:

- `<pi-package>/CHANGELOG.md` is the fastest way to tell whether a symptom is
  already fixed or whether a field was renamed. Search it for the setting or
  command name.
- Machine-readable schemas ship in `<pi-package>/schemas/` and are published as
  `https://pi.dev/schemas/{settings,models,keybindings,theme}.schema.json`.
  When a field type is in doubt, the schema is authoritative; `pi --help`,
  `pi <command> --help`, and `/hotkeys` are authoritative for the CLI and UI.
- `docs/docs.json` lists every installed doc page, which makes it a fast index
  when the topic is new.

## Reference map

- `references/configuration.md`: paths, scope and precedence, project trust,
  every settings group with defaults, tool selection, compaction, reload matrix,
  environment variables.
- `references/mcp-and-codemode.md`: `mcp.json`, project overrides, exposure,
  OAuth, resource tools, and the codemode script API and limits.
- `references/resources.md`: skills, prompt templates, themes, Pi packages.
- `references/runtime.md`: models, `models.json`, providers and auth, classifier
  and image models, virtual models, the extension API.
- `references/security-and-windows.md`: trust and isolation, native Windows
  shells, fullscreen terminal behavior, keybindings, session commands.

If the user asks for a report, diagnosis, or review rather than an edit, do not
modify files. Show evidence, paths, a proposed diff, and residual risks.

## Locations and scope

The agent directory defaults to `~/.pi/agent`; `PI_CODING_AGENT_DIR` overrides
it. User configuration is global, applies to every project, and is always
loaded. Project configuration lives under `.pi/` and loads only after project
trust is granted.

| Path | Responsibility |
|---|---|
| `<agent-dir>/settings.json` | Global settings, resource declarations, packages |
| `.pi/settings.json` | Project settings and project packages (trusted) |
| `<agent-dir>/mcp.json`, `.pi/mcp.json` | MCP servers (project file is trusted-only) |
| `<agent-dir>/models.json` | Compatible endpoints, models, model overrides |
| `<agent-dir>/auth.json`, `mcp-auth.json`, `trust.json`, `mcp.log` | Credentials and runtime state; keep private |
| `<agent-dir>/keybindings.json` | Named keybindings |
| `<agent-dir>/{extensions,skills,prompts,themes}/` | Global resources |
| `.pi/{extensions,skills,prompts,themes}/` | Project resources (trusted) |
| `<agent-dir>/AGENTS.override.md`, `AGENTS.md`, `CLAUDE.md`, `SYSTEM.md`, `APPEND_SYSTEM.md` | Global context and system prompt |
| `AGENTS.md`, `CLAUDE.md`, `.pi/SYSTEM.md`, `.pi/APPEND_SYSTEM.md` | Project context and system prompt |
| `<agent-dir>/sessions/` | Persisted sessions unless overridden |

Project settings override user settings; resource lists are combined. A project
package normally replaces the same package from the user scope unless its
declaration uses `autoload: false`, which makes it a filtering delta instead.
Path entries resolve from the agent directory in user settings and from
`.pi/` in project settings, and accept `~` and absolute paths.

## Safe change workflow

Inspect with the least invasive command that answers the question:

- `pi --version` for the release the user is actually running.
- `pi --help`, `pi <command> --help`, and `/hotkeys` for the installed surface.
- `pi config` to inspect packages, discovered resources, and built-in
  extensions; `pi config --local` starts at project scope.
- `pi list` for configured packages.
- `pi mcp list --json` to diagnose MCP servers. It connects to every enabled
  server, so warn first when a server is expensive or stateful.
- `/settings`, `/mcp`, `/model`, `/session`, `/debug` inside a session, and the
  session JSONL for runtime diagnostics.

Apply deliberately:

- Put personal defaults in the agent directory and team behavior under `.pi/`,
  so the change is loaded by the people and projects that should get it.
- Prefer the documented management surface where one exists: `/settings` for
  preferences, `/mcp` for servers, `/login` for credentials, `pi install` for
  packages, `pi mcp add` for servers from a shell.
- Keep secrets out of anything that is copied verbatim or committed: use
  `/login`, `auth.json`, `${ENV_VAR}` interpolation, or a whole-value
  `!command`. Never paste a token into a project file, skill, or report.
- Back up a file before a risky migration, keep the diff minimal, validate JSON,
  then run a read-only check (`pi list`, `pi mcp list`, `pi config`) when one
  applies.
- Report the exact file changed, the validation performed, and whether the user
  needs `/reload` or a new process.

### Reload and restart

Pi hot-reloads most settings and resources. Run `/reload` after manually
changing settings, keybindings, extensions, skills, prompt templates, themes, or
context files. `/reload` enables tools newly added to `defaultTools`, but it
does not disable tools removed from that setting, nor re-enable tools the user
turned off in the session; `--tools` with plain names also overrides
`defaultTools` on reload.

`/model` re-reads `models.json`. The active user theme hot-reloads when its own
file changes. Saved project-trust decisions need a new process, because trust is
not switched in place. MCP edits made outside the session need `/reload`, while
`/mcp` changes apply to the active session. Startup defaults such as the saved
default model and thinking level only affect new sessions.

## settings.json

Pi 1.4 defaults that matter in conversation: `theme: "system"`,
`tuiMode: "fullscreen"`, `quietStartup: false`, default tools `read`, `bash`,
`edit`, `write`, automatic compaction, and agent-level retries. Set
`tuiMode: "regular"` when the user wants native terminal scrollback, and
`quietStartup: "header"` to keep the version/key hints while hiding the model
scope line and resource listing.

A minimal starting point:

```json
{
  "$schema": "https://pi.dev/schemas/settings.schema.json",
  "defaultProvider": "provider-id",
  "defaultModel": "model-id",
  "defaultThinkingLevel": "medium",
  "theme": "system",
  "tuiMode": "fullscreen",
  "quietStartup": false,
  "defaultTools": ["+codemode"],
  "compaction": { "enabled": true, "reserveTokens": 16384 },
  "retry": { "enabled": true, "maxRetries": 3 }
}
```

Tool selection is the rule most people get wrong. A list of plain names
replaces the selection; a list of only `+name` and `-name` entries adjusts the
inherited selection. So `["+codemode"]` adds codemode next to the defaults,
while `["-bash", "+powershell", "+grep"]` swaps the shell. An empty array
is not a way to say "keep the defaults": it disables every built-in tool. The
CLI mirrors this: `--tools read,grep` replaces,
`--tools +codemode` adjusts, patterns (`--tools 'mcp__radius__*'`) and
`--exclude-tools` exist, and plain names cannot be mixed with `+`/`-` entries.

Built-in extensions are addressed by name in `extensions`: `builtin:mcp`,
`builtin:llama.cpp`, `builtin:codemode`, `builtin:tool-search`. They load by
default, and `"extensions": ["-builtin:mcp"]` disables one. An extension that
registers `/mcp` replaces the built-in MCP support for its sessions, so check
loaded resources before blaming MCP itself.

`references/configuration.md` lists the remaining groups (`terminal.*`,
`images.*`, `markdown.*`, `warnings.*`, `codemode.*`, `branchSummary.*`,
`retry.provider.*`) with their defaults.

## Built-in MCP and codemode

Since 0.99, MCP, codemode, and tool search are built-in extensions. Do not
recommend a third-party adapter merely to get standard MCP; recommend one only
when the user wants a different implementation or a feature Pi does not ship.

```bash
pi mcp add <name> -- <command> [args...]
pi mcp add <name> --url https://example.com/mcp
pi mcp add -l <name> --env API_KEY='${TOOLS_KEY}' -- uvx tools-mcp
pi mcp list [--json]
pi mcp login <name>
pi mcp logout <name>
pi mcp remove <name>
```

`/mcp` inside a session inspects connections, tool lists, source, exposure,
enabled state, OAuth, and reconnect; it also saves enable/disable and exposure
changes back to the defining file. A project entry without `command`, `url`, or
`type` is a partial override that changes only `enabled`, `exposure`, and
`toolExposure` of the user-level server with the same name.

Exposure is one of `codemode` (default), `deferred`, `direct`, or `hidden`
(`codemode-deferred` is an accepted alias). `direct` declares tools to the
model, `codemode` makes them callable from scripts without declaring them,
`deferred` lets `tool_search` load them, and `hidden` keeps them unreachable.
`toolExposure` overrides per tool with exact names or `*` patterns. The
effective tool name is `mcp__<server>__<tool>`, with every character other than
letters, digits, and `_` replaced by `_`.

Enable codemode for every session with `"defaultTools": ["+codemode"]`, or for
one run with `pi --tools +codemode`. Scripts run in QuickJS with no filesystem,
network, timers, or Node APIs; they reach tools through `tools.<name>(args)` and
non-LLM models through `models`. Use `"name" in tools` rather than
`typeof tools.name` when probing, and `Promise.allSettled()` when one failed
call should not discard its siblings.

`references/mcp-and-codemode.md` carries the full server schema, OAuth options,
resource tools, connection behavior, and codemode limits.

## Skills, prompt templates, themes, and packages

A portable skill is a directory containing `SKILL.md` with `name` and
`description` frontmatter. The metadata is always advertised for routing; the
body loads on demand, so keep it under about 500 lines and push detail into
`references/`, `scripts/`, or `assets/`. Use `disable-model-invocation: true`
for explicit-only skills.

A direct `.md` file in a prompt directory becomes a slash command named after
the file, with optional `description` and `argument-hint` frontmatter and
`$1`/`$@`/`${1:-default}` substitution. Templates add prompt text; anything
that needs state, tools, or events belongs in an extension.

Themes are strict JSON: `name` plus `colors`, optional `$schema`, `appearance`,
`vars`, and `export`. `system` derives colors from the terminal and is reserved.
The `theme` setting also accepts a `"light/dark"` pair, and `--use-theme` picks
the theme for one run.

```bash
pi install npm:@scope/package@1.0.0
pi install git:github.com/user/repo@v1
pi list
pi config
pi remove <source>
pi update --extensions
```

`-l`/`--local` writes to project settings and `pi -e <source>` tries a package
for one run. Pin versions and refs for reproducibility, and review extension
code before installing because it runs inside the Pi process.

`references/resources.md` has discovery paths, frontmatter fields, theme color
forms, and package manifest rules.

## Models, providers, auth, and extensions

Use `/login` and `/logout` for credentials; keep `auth.json` private. Keys may
also come from the environment, `models.json` `apiKey`, or a command. Use
`models.json` for OpenAI-, Anthropic-, Google-, or Azure-compatible endpoints,
and build a provider extension only for custom auth, discovery, or wire
protocols.

Pi distinguishes `chat`, `image`, and `classifier` models. Only chat models
appear in `/model`; image and classifier models are reached from codemode
(`models.generateImages()`, `models.classify()`) or from extensions through
`ctx.modelRegistry`. Their usage is billed to the session.

Use an extension for executable behavior: tools, `/` commands, flags,
shortcuts, providers, virtual models, MCP registrations, lifecycle handlers,
state, or UI. Extensions run with the Pi process's full OS permissions. Keep
long-lived processes in `session_start`/`session_shutdown`, not in the factory,
and declare `outputSchema`/`structuredContent` when a tool returns data.

Read `docs/extensions.md` and `docs/providers.md` before writing against event
names or TypeScript types; those APIs move faster than config formats.
`references/runtime.md` summarizes the pieces that are stable enough to build on.

## Windows and terminal notes

Native Windows uses Git Bash by default. Pi checks `shellPath`, then Git Bash
under `%ProgramFiles%` and `%ProgramFiles(x86)%`, then `bash.exe` on `PATH`.
Add `powershell` to `defaultTools` (`["-bash", "+powershell"]` keeps the rest)
when the model should use PowerShell natively; `!` and `!!` editor commands keep
using Bash. On Windows and WSL, follow-up defaults to `Ctrl+Q` because Windows
Terminal reserves `Alt+Enter`, and `app.suspend` has no default.

Fullscreen is the default and owns the viewport, so native scrollback is gone
until `tuiMode: "regular"`. Terminals that misreport capabilities can be
corrected with `terminal.hyperlinks`, `terminal.images`, `terminal.trueColor`
(settings win over `PI_HYPERLINKS`, `PI_IMAGE_PROTOCOL`, `PI_TRUE_COLOR`). OSC
7501 program status lets terminals and dashboards show working/blocked/done.

## Security checklist

- Treat model-generated commands, project files, MCP servers, skills, packages,
  and extensions as executable, untrusted input.
- Project trust controls resource loading only. It is not a sandbox and does not
  restrict OS permissions, tool access, or context files such as `AGENTS.md`,
  which load regardless of trust.
- Prefer a container, VM, or dedicated low-privilege account for untrusted or
  unattended work.
- Use narrow, short-lived credentials and keep them out of project files.
- Review package and extension source before install; do not assume an MCP
  server or a skill is read-only.
- Review sessions, `/export`, `/share`, `/bug`, and `/debug` output before
  uploading: they can contain prompts, files, command output, and secrets.

## When proposing edits

Show a concise plan or unified diff before applying changes when the user asked
for review or has not authorized writing, and preserve the original file until
approval. After applying, report:

1. exact path(s) changed;
2. backup or diff status;
3. validation performed;
4. `/reload` versus restart requirement;
5. security or compatibility residuals, including a version assumption.
