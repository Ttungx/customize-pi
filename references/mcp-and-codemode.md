# MCP and codemode reference

Use this file for `mcp.json`, server exposure, OAuth, MCP resource tools, and
the codemode script API. Confirm details in the installed `docs/mcp.md`,
`docs/codemode.md`, and the MCP section of `docs/cli.md`.

## Files and commands

- User servers: `<agent-dir>/mcp.json`.
- Project servers: `.pi/mcp.json`, read only after project trust.
- OAuth credentials: `<agent-dir>/mcp-auth.json`, keyed by server name and URL.
- Server logs: `<agent-dir>/mcp.log` (rotates to `mcp.log.1` past 5 MB).

```bash
pi mcp add <name> -- <command> [args...]         # --env KEY=VALUE, --cwd <dir>
pi mcp add <name> --url https://example.com/mcp  # --header, --bearer-token-env-var
pi mcp add <name> --url ... --oauth-client-id <id> --oauth-client-secret <secret> \
  --oauth-callback-port 8765 --oauth-client-name "Pi"
pi mcp add <name> ... --exposure codemode --description "What it offers"
pi mcp add -l <name> -- ...                      # write .pi/mcp.json
pi mcp list [--json]                            # connects; exits 1 on failure
pi mcp login <name> [--timeout 300]
pi mcp logout <name>
pi mcp remove <name>
```

`pi mcp list` connects to every enabled server, so it can start processes or
download packages. Warn before using it against expensive or stateful servers.
`--json` reports `resources: true` for servers that offer resources instead of
counting them (1.2+).

In a session, `/mcp` lists servers with state, tool count, exposure, and source,
and can inspect tools, reconnect, sign in/out, change exposure, and enable or
disable a server. In a trusted project, "Enable/Disable in this project" writes a
project override. Shell-level `pi mcp` commands do not load extensions.

## Server configuration

```json
{
  "mcpServers": {
    "local-tools": {
      "command": "npx",
      "args": ["-y", "some-mcp-server"],
      "env": { "API_KEY": "${TOOLS_KEY}" },
      "cwd": "./tools",
      "exposure": "codemode",
      "enabled": true,
      "timeout": 60,
      "description": "A short summary used in prompts and tool search"
    },
    "remote-tools": {
      "url": "https://example.com/mcp",
      "headers": { "Authorization": "!gh auth token" },
      "oauth": { "clientName": "Pi" }
    }
  },
  "autoEnableCodemode": false
}
```

Stdio servers use `command`, `args`, `env`, `cwd`; HTTP servers use `url`,
`headers`, `oauth`. Both accept `timeout` (seconds, default 60), `enabled:
false` (keep the entry without connecting), `exposure`, `toolExposure`, and
`description`. `type` is optional and, when present, must be `stdio`, `http`, or
`streamable-http`; legacy SSE is rejected, so point at the server's `/mcp`
endpoint instead. `command` is one executable, not a shell string. Relative
`cwd` resolves against the session directory; a leading `~/` names the home
directory. `env`/`headers` values interpolate `${NAME}` and may be a whole-value
`!command`. Invalid entries are reported and skipped.

`autoEnableCodemode: false` (top level, project overrides user) stops Pi from
activating codemode for MCP servers automatically.

A `.pi/mcp.json` entry without `command`, `url`, or `type` overrides only
`enabled`, `exposure`, and `toolExposure` of the same-named user server and
keeps its `env`, `headers`, and `auth`. That is how a project disables a global
server:

```json
{ "mcpServers": { "internal-tools": { "enabled": false } } }
```

Naming rules: server names may contain only letters, digits, `_`, and `-`.
Tool names are `mcp__<server>__<tool>` with every other character replaced by
`_`; tools whose names then collide all get a hash suffix. Names that differ
only in `-` and `_` count as the same server, and a file-configured server
overrides an extension-registered one with the same name.

## Exposure

| Exposure | Behavior |
|---|---|
| `codemode` (default) | Callable from scripts; not declared; listed only via `searchTools`/`ALL_TOOLS` |
| `deferred` | Not declared until `tool_search` loads a match for the next call |
| `direct` | Declared like a built-in tool and also callable from scripts |
| `hidden` | Registered but unreachable |

`codemode-deferred` is accepted as an alias for `codemode`. `toolExposure`
overrides per tool using exact names or `*` patterns; exact names win over
patterns and, among patterns, the first match wins. A `hidden` server can still
expose selected tools:

```json
{ "exposure": "deferred",
  "toolExposure": { "search_code": "direct", "get_*": "codemode", "delete_*": "hidden" } }
```

`describeNamespace()` and `searchTools()` accept the namespace as
`mcp__dev-radius`, `mcp__dev_radius`, `dev-radius`, or `dev_radius`.

Startup behavior: enabled servers connect in the background. The first prompt
waits up to 10 seconds only for servers with `direct` tools. Codemode scripts
wait for the servers they name; `searchTools()`/`ALL_TOOLS`, `tool_search`, and
the resource tools wait for all. Because the codemode description does not list
MCP tools, it does not change as servers connect. HTTP errors (408, 429, 5xx)
are retried twice; tool calls are never retried. Text over 20 KB reaches the
model with its middle elided around a `…N chars truncated…` marker and the full
text in a temp file; scripts still receive the whole result.

## OAuth and provider tokens

Run `/mcp login <name>` or `pi mcp login <name>`. Tokens live in
`mcp-auth.json` per server name and URL, so one URL can hold several accounts.
OAuth applies to HTTP servers without an `Authorization` header.

`oauth` options:

| Option | Use |
|---|---|
| `clientId`, `clientSecret`, `callbackPort`, `callbackUrl` | Pre-registered client; redirect must match |
| `scope` | Space-separated scopes for servers that advertise none; later scope requests are added |
| `clientName` | Send another name while registering, for servers that allowlist clients |
| `clientRegistration: "cimd"` | Identify via Pi's Client ID Metadata Document instead of registering; cannot be combined with `clientId`/`clientName` |
| `authServerMetadataUrl` | Override broken discovery; HTTPS only outside loopback |

Pi checks RFC 9728 protected-resource metadata, RFC 8414 issuer metadata, and
RFC 9207 `iss` on authorization responses. `authServerMetadataUrl` is trusted as
configured, so only point it at a document you trust.

Separately, an HTTP entry may use a provider's own login token:

```json
{ "auth": { "provider": "radius" } }
```

That is not MCP OAuth. Do not paste bearer tokens into project files or reports.

## Replacing the built-in MCP support

An installed extension that registers `/mcp` (for example `pi-mcp-adapter`)
replaces built-in MCP for its sessions: Pi no longer reads `mcp.json`, and
`/mcp` belongs to the extension. Disable the built-in without a replacement
using `"extensions": ["-builtin:mcp"]` or `pi config`, or for one run with
`--no-mcp`. An extension that replaces `codemode` or `tool_search` only works
for MCP when it declares `reachesTools`.

## MCP resource tools

Servers that offer resources add `list_mcp_resources`,
`list_mcp_resource_templates`, and `read_mcp_resource`. They reach every
enabled, non-hidden server with resources and take the widest exposure among
them (`direct`, then `codemode`/`deferred`). Text arrives as text, images as
images, and other binary resources as a temp-file path. `ui://` MCP App
resources are omitted because Pi does not render them.

## Codemode

Enable globally with `"defaultTools": ["+codemode"]` or for one run with
`pi --tools +codemode` (`--tools` with only `+`/`-` entries adjusts the default
selection; plain names replace it).

A script is raw JavaScript, not JSON and not a fenced block, running as an async
function body in QuickJS. There is no Node API, filesystem, network, or timer;
calls go through `tools` and `models`. An optional options line must be the
first line, or the call fails before running:

```js
// @options: {"max_output_tokens": 2000, "timeout_ms": 60000}
```

`max_output_tokens` defaults to 10000 and keeps the start and end of longer
output with a temp file for the full text. `timeout_ms` is a hard deadline for
the whole script, including the wait for a free worker; leave it generous for
image generation.

Globals:

| Global | Purpose |
|---|---|
| `tools.<name>(args)` | Call a tool; identifiers use `_` for invalid characters |
| `text(value)`, `image(value)`, `console.log(...)` | Add output items |
| `return value`, `exit()` | Return a value like `text()`, or end successfully |
| `store(key, value)` / `load(key)` | Small JSON state across calls, per branch |
| `ALL_TOOLS`, `searchTools(q, {limit, namespace})`, `describeTool(name)`, `describeNamespace(name)` | Discovery |
| `models` | `getModelsOfType`, `getAvailableOfType`, `getModelOfType`, `classify`, `generateImages` |

What a call resolves to depends on the tool. Tools with an output schema resolve
to data: `bash` gives `{ output, truncated, full_output_path?, exit_code,
wall_time_seconds }` and keeps up to 1 MiB of output, also on non-zero exit.
MCP tools give their `CallToolResult` including `isError` and
`structuredContent`. `read` gives text, or an image block for an image file that
`image()` can show. Other tools give their text output. Failures reject with an
`Error` carrying the tool's message, so use `Promise.allSettled()` when siblings
should survive. Probe with `"name" in tools`, never `typeof tools.name`.

Output limits: a script fails past 16,777,216 characters of text and base64
image data, or 100,000 `text()`/`image()`/`console` calls. With several text
items each is prefixed with `==> text N/M <==`, and console lines follow in one
`<console_output>` block.

Store limits: 262,144 characters of JSON per value and 1,048,576 in total. Writes
are kept only when the script succeeds, recorded as `codemode-store` session
entries, so each branch sees its own values. Do not store image data; show it
with `image()`, which also saves a temp file.

Worker limits: at most two codemode workers exist per Pi process across
sessions (`PI_CODEMODE_MAX_WORKERS` raises it). At capacity, scripts queue in
arrival order and fail if no worker frees up within 60 seconds or before their
`timeout_ms`. Each VM gets 256 MB. A script cannot start another codemode script.

Presentation: `codemode.mode: "on"` (default) leaves declared tools visible and
notes how to call them from scripts; `"only"` hides active tools and lists them
in the codemode description instead. `codemode.inlineBudget` (default 3000
estimated tokens) caps the declarations inlined in the description; tools that
do not fit are found with `searchTools()`.

`models.classify()` and `models.generateImages()` run with the session's
credentials and never throw on provider errors: check `stopReason` and
`errorMessage`. At most four run at once per script, and their usage is added to
the codemode result and to the session cost.
