# Safety, terminal, keybindings, and sessions

Use this file for trust and isolation, native Windows shells, fullscreen
terminal behavior, keybindings, and session commands. Confirm details in the
installed `docs/security.md`, `docs/containerization.md`, `docs/windows.md`,
`docs/terminal-setup.md`, `docs/keybindings.md`, and `docs/sessions.md`.

## Safety and trust

Pi runs tools, extensions, packages, MCP servers, and child processes with the
OS permissions of the account that launched it, and it does not ask before
every tool call. Files, comments, command output, MCP responses, skills, and
model output can steer the model through prompt injection.

Project trust decides which project resources load: `.pi/settings.json`,
`.pi/mcp.json`, `.pi/{extensions,skills,prompts,themes}`, `.pi/SYSTEM.md`,
`.pi/APPEND_SYSTEM.md`, and project `.agents/skills`. It does not restrict what
tool calls can reach afterward. Context files (`AGENTS.md`, `CLAUDE.md`) load
even without trust, so treat them as untrusted input.

Decisions are `--approve`/`--no-approve`, an extension `project_trust` handler,
a saved decision in `<agent-dir>/trust.json`, then `defaultProjectTrust`.
Saving via `/trust` only affects future processes.

Isolation that actually bounds damage:

| Setup | What stays protected |
|---|---|
| Directly, as the user | Whatever that user cannot access |
| Container, VM, or sandbox | Host files and processes not exposed to it |
| Pi outside, built-in tools inside | Host resources reached only through those tools |

Prefer running untrusted or unattended work in a container, VM, or dedicated
low-privilege account; see `docs/containerization.md`. Keep credentials outside
the environment where possible, or use narrow, short-lived ones.

Review before uploading anything: sessions, `/export`, `/share`, `/bug`, and
`/debug` can contain prompts, tool arguments, command output, file contents, and
secrets that appeared during the conversation.

## Native Windows

Bash resolution order: `shellPath`, then
`%ProgramFiles%\Git\bin\bash.exe`, then `%ProgramFiles(x86)%\Git\bin\bash.exe`,
then `bash.exe` on `PATH` (including Cygwin, MSYS2, or legacy WSL Bash). Set
`shellPath` with doubled backslashes in JSON:

```json
{ "shellPath": "C:\\cygwin64\\bin\\bash.exe" }
```

The optional `powershell` tool runs `pwsh.exe` when available, then Windows
PowerShell, always with `-NoProfile -NonInteractive -ExecutionPolicy Bypass`;
administrator-enforced execution policies still win. Discovery checks
`pwsh.exe` on `PATH`, `%ProgramFiles%\PowerShell\7\pwsh.exe`,
`powershell.exe` on `PATH`, then
`%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe`. The tool exists
only in a native Windows process.

To let the model use PowerShell instead of Bash:

```json
{ "defaultTools": ["-bash", "+powershell"] }
```

`["read", "powershell", "edit", "write"]` also works but discards any other
default tools you configured. `!` and `!!` editor commands always use Bash. When
Pi runs in WSL, `!` and the `bash` tool use the Linux environment instead.

Windows Terminal reserves or rewrites modified keys. Configure it to forward
`Shift+Enter` for newlines and see `docs/terminal-setup.md#windows-terminal` for
`Alt+Enter`. Use `/hotkeys` to confirm what Pi actually receives.

## Fullscreen and terminal capabilities

Fullscreen is the default and owns the viewport, so there is no native
scrollback. Set `tuiMode: "regular"` when the user wants the terminal's own
scrollback; `fullscreenExitOutput` chooses what remains after leaving fullscreen.
`fullscreenScrollbar`, `fullscreenCopyOnSelect`, and
`fullscreenWheelScrollLines` tune the experience, and `showHardwareCursor: true`
uses the terminal's cursor while still positioning it for IME.

`Home`/`End` move the editor cursor to the line start or end; the transcript
top/bottom moved to `Ctrl+Home`/`Ctrl+End`. On Windows and WSL, follow-up is
`Ctrl+Q` (not `Alt+Enter`), cycle-backward is `Alt+P`, paste-image is `Alt+V`,
and `app.suspend` has no default.

When a proxy or multiplexer makes detection wrong, override it (settings take
precedence over the environment):

| Capability | Setting | Environment |
|---|---|---|
| OSC 8 hyperlinks | `terminal.hyperlinks` | `PI_HYPERLINKS` |
| Inline images | `terminal.images` | `PI_IMAGE_PROTOCOL` |
| Truecolor | `terminal.trueColor` | `PI_TRUE_COLOR` |

Force only capabilities the whole terminal path supports; unsupported escape
sequences corrupt the display.

Program status (OSC 7501) lets terminals and agent dashboards show `working`,
`blocked`, `done`, `error`, or `idle`. Pi sends reports only after the terminal
confirms support; `PI_PROGRAM_STATUS=1` sends unconditionally and `0` disables.
tmux and screen do not forward them.

## Keybindings

`<agent-dir>/keybindings.json` maps action ids to one key or a list, with
`"$schema": "https://pi.dev/schemas/keybindings.schema.json"`. A configured
value replaces the default for that action, and an empty list disables it. Keys
are `modifier+key` with `ctrl`, `shift`, `alt`, and `super`; `super` needs a
terminal that reports it separately, usually via the Kitty keyboard protocol.
Run `/reload` after edits and `/hotkeys` to inspect the result.

Frequently rebound actions: `app.interrupt`, `app.clear`, `app.editor.external`,
`app.model.select`, `app.model.cycleForward`/`cycleBackward`,
`app.thinking.cycle`, `app.message.followUp`/`dequeue`,
`app.session.new`/`tree`/`fork`/`resume`, `app.tools.expand`,
`app.message.copy`, and the `tui.editor.*` movement actions.

## Sessions and context

Sessions are JSONL trees grouped by working directory, with the active branch
supplying conversation history.

| Command | Result |
|---|---|
| `/new`, `/resume`, `/import <path>` | Start, switch, or import a session |
| `/name [name]`, `/session` | Name or inspect the current session |
| `/tree` | Move within the session file |
| `/fork`, `/clone` | Branch into a new session, or copy the current branch |
| `/compact [instructions]` | Summarize old context; entries are kept |
| `/export [path]`, `/share`, `/copy` | Export, upload, or copy output |
| `/bug [description]` | Prepare a private report; review it first |

Storage precedence: `--session-dir`, then `PI_CODING_AGENT_SESSION_DIR`, then
`sessionDir`, then the agent session directory. `--no-session` keeps the session
in memory only. `/share` uses `sessionShareDestination`: `auto` prefers Radius
when authenticated and otherwise a private gist, `gist` and `radius` force one.
