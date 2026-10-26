# Pi runtime reference

Use this file for models, `models.json`, providers and auth, classifier and
image models, virtual models, and the extension API. Confirm exact types in the
installed `docs/models.md`, `docs/models-configuration.md`, `docs/providers.md`,
`docs/extensions.md`, `docs/custom-provider.md`, and `docs/virtual-models.md`.

## Authentication and model selection

Credentials resolve in this order: a runtime `--api-key`, a stored `auth.json`
credential, an `apiKey` in `models.json`, then the provider's environment
variables or ambient cloud credentials. Provider extensions may add their own
behavior. Keep `auth.json` private.

- `/login` and `/logout` manage stored credentials; `/login` also offers
  Radius and headless flows such as pasting an Anthropic authorization code.
- `pi auth check|print-api-key|print-bearer-token --provider <id>|--model <id>`
  resolves credentials non-interactively. `check` prints `ready`, `not_ready`,
  or `invalid` and exits 0/1/2; `--json` and `--credentials` shape the output;
  `--no-refresh` avoids refreshing OAuth. The print commands write secrets to
  stdout.
- `/model` selects a model, `/thinking` sets its level, `/scoped-models`
  configures the `Ctrl+P` cycle, and `Ctrl+S` in either picker saves the choice
  as the startup default. `--model provider/id:thinking` and `--models <patterns>`
  do the same for one run.

## models.json

Use it for OpenAI-, Anthropic-, Google-, or Azure-compatible endpoints, or to
override metadata on a built-in model. Add
`"$schema": "https://pi.dev/schemas/models.schema.json"`.

```json
{
  "$schema": "https://pi.dev/schemas/models.schema.json",
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [{ "id": "qwen2.5-coder:7b" }]
    }
  },
  "modelOverrides": {
    "anthropic/claude-sonnet-5": { "promptCache": { "short": 300, "long": 3600 } }
  }
}
```

- `models` adds or replaces a model with the same ID on that provider.
  `modelOverrides` changes metadata for an existing built-in or
  extension-provided model and ignores unknown IDs.
- Setting only a built-in provider's `baseUrl` routes it through a proxy while
  keeping its models and authentication.
- `apiKey` and header values accept `$NAME`/`${NAME}` interpolation, a literal,
  or a leading `!command` that runs at request time and is not cached by Pi.
  `$$` and `$!` escape a literal `$` and `!`.
- `inputLimits.images.resize` controls how new images are encoded; omitted
  fields default to 2000×2000, 4.5 MiB encoded, and JPEG quality 80. Images are
  encoded once, so switching models does not rewrite history.
- `promptCache: { "short": seconds, "long": seconds }` declares best-effort
  cache lifetimes. Models without a lifetime for the active tier are not
  eligible for `cacheWarming`.
- `samplingParams` and `samplingParamsByThinkingLevel` apply to
  `openai-completions`, `openai-responses`, and `azure-openai-responses` only.
  Pi clamps the level first, then merges model defaults, the level override, and
  request parameters, later values winning per key.
- `compat` fields describe verified endpoint differences. Never enable one just
  because a service advertises compatibility.

`/model` reloads the file. `docs/models-configuration.md` is a generated
reference for every provider, model, override, and compatibility field, so
prefer it over guessing a key. Run `pi update --models` to refresh the catalog;
cached catalog data works offline.

## Local models

llama.cpp is integrated rather than configured through `models.json`: start
`llama-server` without `--model` so it runs as a router, connect with
`/login llama.cpp` (default `http://127.0.0.1:8080`), and load a GGUF with
`/llama` before selecting it in `/model`, because only loaded models appear
there. The built-in extension is `builtin:llama.cpp`, and
`--no-models-autoload` keeps loading explicit. For Ollama, LM Studio, vLLM,
SGLang, and proxies, use a compatible endpoint in `models.json` instead.

## Model types

| Type | Reached through | Notes |
|---|---|---|
| `chat` | `/model`, `Ctrl+P` | Normal conversation models |
| `classifier` | Codemode `models.classify()`, `ctx.modelRegistry.classify()` | Typed questions, no chat |
| `image` | Codemode `models.generateImages()`, `ctx.modelRegistry.generateImages()` | Prompt and optional input images |

Classifier options at 1.4: TypeSafe Jev (`typesafe`, `openrouter`,
`cloudflare-workers-ai`, `vercel-ai-gateway`, `opencode`), Cloudflare Clef,
Clef Flash, and Clef Omni (`cloudflare-workers-ai`, `openrouter`), and OpenAI
GPT-6 Luna (`openai`, Decisions API). Luna needs `OPENAI_API_KEY`; ChatGPT
sign-in does not work with the Decisions API, so log out of `openai` to use the
key.

Classifiers answer `choice`, `score`, and `bool` questions about a JSON `state`,
and models whose `input` includes `"image"` also accept `images` (PNG/JPEG for
Jev, up to four PNG/JPEG/WebP for Clef). Each `compat.imageInput` setting decides
the request format for a proxied service; OpenRouter accepts TypeSafe-format
image requests but silently drops the images.

Image models are the OpenRouter image catalog, for example
`google/gemini-2.5-flash-image`. Generated images are not saved by Pi: pass each
block to `image()`, which also writes a temp file whose path is returned.
Classifier and image usage counts toward session cost.

Use `models.getAvailableOfType(type)` to find IDs that work with the current
credentials, because IDs differ per provider (`typesafe/jev-latest` versus
`openrouter/typesafe/jev-1.14`).

## Virtual models

`pi.registerVirtualModel()` exposes one selectable model that routes each request
to a physical model, which is how an extension can add a router or a
classifier-driven dispatch layer. Selection and dispatch are recorded separately.
Routes cannot target another virtual model, router state must be
JSON-serializable and follows the session branch, and continuation or retry
requests should stay sticky when prompt caching or reasoning signatures require
it.

## Extensions

Extensions are TypeScript/JavaScript modules loaded with `jiti` and run with the
Pi process's OS permissions. They register tools, commands, flags, shortcuts,
providers, virtual models, MCP servers, renderers, lifecycle handlers, state, and
UI. Build an extension for behavior; use settings or a skill for anything that is
only configuration or instructions.

Lifecycle shape at 1.4: `input` → `before_agent_start` → `agent_start` → turns
(`turn_start`, `context`, `context_with_system`, provider hooks, `turn_end`) →
`agent_end` → `agent_before_settle` → `agent_settled`. Session startup runs
`project_trust` when needed, then `session_start` and `resources_discover`;
`session_before_switch`/`session_before_fork` precede `session_shutdown`. A run
can restart after `agent_end` for retries, recovery, compaction retries, or
queued work, so `agent_end` is not an idle signal — use `agent_settled`, which
now also carries `aborted`.

Practices that keep extensions correct:

- Never start processes, sockets, watchers, or timers in the factory; some
  invocations load extensions without a session. Use `session_start` and an
  idempotent `session_shutdown`.
- Declare `outputSchema` and return matching `structuredContent` for data. The
  model sees `content`, scripts see `structuredContent`. To report a failure that
  still carries data, return `isError: true` instead of throwing.
- `ctx.executeTool()` runs nested calls through validation and the `tool_call`/
  `tool_result` pipeline with `parentToolCallId` and `<parent id>/<n>` ids.
  Nested usage is added to the calling tool's result.
- Wrap read-modify-write file operations in `withFileMutationQueue()`, and
  truncate large model-facing results while pointing at the full output.
- `exposure` is `direct` (default), `model-only` (declared, never callable),
  `codemode`, `deferred`, or `hidden`. `namespace: { name, aliases,
  description, instructions }` groups tools; `annotations` carry MCP-style
  hints; `reachesTools(input)` and `prepareLoadout(loadout)` let an orchestrating
  tool describe and adjust what it reaches.
- `pi.registerToolRenderer()` covers calls to tools that are not registered yet,
  such as MCP tools in a resumed session.
- `pi.registerMcpServer()`/`pi.unregisterMcpServer()` are session-only and are
  not persisted; register again on every load. `pi.getMcpServers()` and the
  `mcp_servers_change` event serve extensions that implement their own MCP
  transport.
- `before_agent_start` is a union on `messageType`: `"user"` events carry
  `prompt` and `images`, `"custom"` events carry the starting `message`. Check it
  when a handler should ignore `pi.sendMessage(..., { triggerTurn: true })`
  runs. Prefer changing prompt sections, tools, or guidelines over returning a
  whole `systemPrompt`, which replaces the prompt for that run.
- `cache_warming_decision` can override an idle cache refresh; the last handler
  returning `{ action }` wins.

Read the installed `docs/extensions.md` and the exported declarations in
`dist/` before writing against event payloads or TypeScript types. Those APIs
change faster than configuration formats.

## Custom providers

Prefer the built-in API implementations through a provider extension when the
service matches Anthropic Messages, OpenAI Chat/Responses, Google, Azure,
Mistral, Bedrock, or another supported wire format. Implement custom streaming
only for genuinely unsupported protocols, and preserve abort, usage, tool-call,
provider-event, and terminal-stream contracts. Never log tokens, authorization
headers, or complete provider responses.
