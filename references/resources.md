# Pi resources reference

Use this file for skills, prompt templates, themes, and Pi packages. Confirm
details in the installed `docs/skills.md`, `docs/prompt-templates.md`,
`docs/themes.md`, and `docs/packages.md`.

## Skills

A portable skill is a directory containing `SKILL.md`:

```text
pdf-tools/
├── SKILL.md
├── scripts/extract.sh
├── references/formats.md
└── assets/template.json
```

The frontmatter `name` and `description` are always advertised, so the
description is the routing mechanism: state what the skill does and when it
applies. The body loads only when the model or the user selects the skill, which
is why large or variant-specific material belongs in `references/`, `scripts/`,
or `assets/` and the body stays under roughly 500 lines.

Portable frontmatter:

| Field | Purpose |
|---|---|
| `name` | Command and display name; lowercase letters, digits, hyphens; no leading, trailing, or consecutive hyphens; max 64 characters |
| `description` | Routing description, max 1024 characters |
| `license` | License name or bundled file |
| `compatibility` | Environment requirements |
| `metadata` | Extra key-value data |
| `allowed-tools` | Experimental pre-approved tool list |
| `disable-model-invocation` | Hide from automatic selection; explicit `/skill:name` only |

Pi is lenient about many specification violations, but malformed `SKILL.md`
files and skills without a description are not loaded, and name collisions keep
the first discovered skill with a warning. Pi does not require the directory
name to match, but matching stays portable to other Agent Skills consumers.

Discovery is recursive and covers `<agent-dir>/skills/`, `~/.agents/skills/`,
and configured paths; project discovery covers `.pi/skills/` and project
`.agents/skills/` from the working directory up to the repository root, after
project trust. Arguments after `/skill:name` are appended as a user request.
`enableSkillCommands` only controls whether skills appear in command discovery;
a manually typed `/skill:name` still works. Run `/reload` after edits.

## Prompt templates

A direct `.md` child of a prompt directory becomes a slash command named after
the file. Optional frontmatter: `description` (defaults to the first non-empty
line) and `argument-hint` (`<required>`, `[optional]`).

| Syntax | Result |
|---|---|
| `$1`, `$2`, … | Positional argument |
| `$@` or `$ARGUMENTS` | All arguments joined with spaces |
| `${1:-default}`, `${@:-default}` | Argument or default |
| `${@:N}`, `${@:N:L}` | Arguments from position `N`, or `L` of them |

Arguments follow shell-like quoting, so `/review "API compatibility"` is one
argument. Extensions see raw input through the `input` event first, unless an
extension command with the same name handles it. Templates add prompt text; use
an extension for state, tools, events, or UI. Run `/reload` after changes.

## Context and system-prompt files

These change what the model knows and how it is instructed without adding a
tool or a command, which makes them the cheapest way to shape behavior — and
the easiest to overuse.

| File | Effect |
|---|---|
| `AGENTS.md`, `CLAUDE.md` | Instructions from the agent directory plus the working directory and its parents |
| `AGENTS.override.md` | Replaces `AGENTS.md`/`CLAUDE.md` in the same directory only |
| `SYSTEM.md` | Replaces Pi's default system prompt |
| `APPEND_SYSTEM.md` | Adds to the system prompt |

A context file applies to its directory and everything below it, so a project
`AGENTS.md` covers the repository while the agent-directory file covers every
session. For `SYSTEM.md` and `APPEND_SYSTEM.md`, the trusted project file
replaces the agent-directory file; same-named files are not concatenated.
`AGENTS.override.md` suppresses only its sibling file, not inherited context.

Context files load without project trust, while `.pi/SYSTEM.md`,
`.pi/APPEND_SYSTEM.md`, and project `.agents/skills` need it. That asymmetry is
why instructions from an unfamiliar directory stay untrusted input even after
trust is declined. Run `/reload` after editing any of them.

## Themes

Pi ships `system`, `dark`, and `light`. `system` derives its palette from the
terminal's reported colors and is reserved: a custom theme named `system` is
ignored. The `theme` setting accepts one name or a `"light/dark"` pair that
follows the terminal's reported appearance, and `--use-theme <name[/name]>`
picks the theme for one run without saving it.

Custom themes are strict JSON. Only documented top-level fields and color tokens
are accepted; extra metadata or ad-hoc keys under `colors`/`export` are
rejected, and reusable colors belong in `vars`.

| Field | Required | Purpose |
|---|---|---|
| `$schema` | no | `https://pi.dev/schemas/theme.schema.json` |
| `name` | yes | Unique, no `/`, not `system` |
| `appearance` | no | `dark` or `light`; detected from colors when omitted |
| `vars` | no | Reusable colors; may reference other variables |
| `colors` | yes | Terminal UI roles |
| `export` | no | HTML export page and panel backgrounds |

Color forms: `#0af` / `#00aaff`, `oklch(62% 0.1 200)`, `okhsl(250 60% 55%)`, an
ANSI index `39`, a variable name, or `""` for the terminal default. Chained
variables resolve; a missing or circular reference makes the theme invalid. Five
colors inherit when omitted: `scrollbarTrack` ← `muted`, `scrollbarThumb` ←
`text`, `searchMatchBg` ← `selectedBg`, `searchMatchText` ← `text`,
`thinkingMax` ← `thinkingXhigh`. Without `export` colors, HTML page and panel
backgrounds derive from `userMessageBg`.

The active user theme hot-reloads only from `<agent-dir>/themes/<name>.json`.
Themes from a project, a package, or the `themes` setting need `/reload`.
Duplicate names are reported as resource collisions.

## Packages

```bash
pi install npm:@scope/package@1.0.0
pi install git:github.com/user/repo@v1
pi install ./local-package
pi list
pi config          # --local starts at project scope
pi remove <source>
pi update --extensions
pi -e npm:@scope/package     # try for one run
```

Personal installs are written to `<agent-dir>/settings.json`; `-l`/`--local`
writes `.pi/settings.json` instead and is loaded only after trust. A git tag or
commit is pinned, and package updates reconcile the checkout without moving the
configured ref.

A package is an ordinary directory or npm package. Without a manifest, Pi
discovers TypeScript/JavaScript extensions, skill directories, Markdown prompts,
and JSON themes from conventional `extensions/`, `skills/`, `prompts/`,
`themes/` directories. An explicit manifest is better when resources live
elsewhere or need filtering; the `pi-package` keyword makes an npm package
eligible for the package gallery, and `pi.image`/`pi.video` add previews.

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./src/extension.ts"],
    "skills": ["./resources/skills"],
    "prompts": ["./resources/prompts/*.md"],
    "themes": ["./resources/themes/*.json"]
  }
}
```

Package filters in settings narrow what loads: omit a resource type to load
everything the package allows, use `[]` for none, `!pattern` to exclude globs,
`+path` to include one exact path, and `-path` to exclude one exact path.
Filters never expose resources the package did not declare.

Pi identifies npm packages by name, git packages by repository URL without the
ref, and local packages by resolved absolute path, so the same package does not
load twice through equivalent declarations. A project entry normally replaces
the user entry; `autoload: false` makes it a filtering delta. Installed packages
load with separate module roots, so two packages do not share a dependency
instance.

Dependencies: runtime imports belong in `dependencies`. The host provides
`@earendil-works/pi-ai`, `@earendil-works/pi-agent-core`,
`@earendil-works/pi-coding-agent`, `@earendil-works/pi-tui`, and `typebox`;
declare those in `peerDependencies` with `"*"` and do not bundle them. A
physical copy can bypass Pi's module mapping and create duplicate classes and
registries, so Pi warns when it sees that manifest mistake.

Extensions execute with the Pi process's permissions and skills can instruct the
model to run programs. Review third-party source, and review project package
declarations, before installing or granting trust.
