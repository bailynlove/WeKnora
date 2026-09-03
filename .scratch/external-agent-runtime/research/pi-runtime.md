# Pi Coding Agent — Integration Surface Research

## 1. Identity

- **Pi** is an open-source, MIT-licensed "AI agent toolkit: unified LLM API, agent loop, TUI, coding agent CLI" — a TypeScript/Node monorepo by Mario Zechner (GitHub handle `badlogic`). Source: https://github.com/earendil-works/pi (the repo was `badlogic/pi-mono`; that URL now redirects to the `earendil-works` org — verified 2026-09-03).
- Packages (current names from the monorepo README): `@earendil-works/pi-ai` (multi-provider LLM API), `@earendil-works/pi-agent-core` (agent runtime w/ tool calling), `@earendil-works/pi-coding-agent` (the `pi` CLI), `@earendil-works/pi-tui`, `@earendil-works/chord`, `@earendil-works/pi-telemetry`. Source: https://github.com/earendil-works/pi
- **Residual ambiguity / restructure flag:** the ticket's remembered names (`pi-agent`, `pi-web-ui`, `@mariozechner/pi-*`) are stale — the README table now shows `pi-agent-core` and no `pi-web-ui` (https://github.com/earendil-works/pi). Identity as "the coding agent Pi" is nonetheless certain: WeKnora's context (coding agent runtime) matches only this project; the other "Pi" candidates (Inflection's chatbot, Raspberry Pi) are not coding agents.
- **Activity:** very active. npm `@earendil-works/pi-coding-agent` v0.84.4 published 2026-08-28; 0.84.0–0.84.4 all published Aug 2026 (~weekly cadence). Sources: https://registry.npmjs.org/@earendil-works/pi-coding-agent ; version list from registry `time` field.

## 2. Integration surfaces

TypeScript/Node everywhere; **no Go SDK**. Four transports (sources: CLI README https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md ; RPC doc https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/rpc.md ; SDK doc .../docs/sdk.md):

1. **In-process TS SDK**: `npm install @earendil-works/pi-coding-agent`; `createAgentSession({ sessionManager, modelRuntime })` → `AgentSession` with `session.prompt() / steer() / followUp() / abort()` and `session.subscribe(listener)`. Multi-session: `createAgentSessionRuntime()` / `AgentSessionRuntime`. Doc: .../docs/sdk.md
2. **RPC mode (subprocess, bidirectional)**: `pi --mode rpc [--no-session] [--session-dir <path>] [--provider <p>] [--model <m>]` — strict LF-delimited JSONL over stdin/stdout. Commands (one JSON/line, optional `id` correlation): `prompt`, `steer`, `follow_up`, `abort`, `new_session`, `switch_session`, `fork`, `clone`, `get_state`, `get_messages`, `get_entries`, `set_model`, `set_thinking_level`, `compact`, `bash`, `export_html`, etc. Responses `{type:"response", success, data|error}`. Typed client: `RpcClient` (`src/modes/rpc/rpc-client.ts`), types in `src/modes/rpc/rpc-types.ts`. Doc: .../docs/rpc.md
3. **JSON mode (one-shot, stdout-only)**: `pi --mode json "prompt"` emits a `{"type":"session",...}` header then typed JSON events per line; final text distinguishable via `message_end`. Doc: .../docs/json.md
4. **Plain print mode**: `pi -p "prompt"` (merges piped stdin). Doc: coding-agent README.
- **MCP: no first-class support.** No MCP client APIs, no `pi mcp` command; repo code search for "mcp" yields only incidental matches (verified via GitHub code search 2026-09-03). The README's "MCP integration" capability line means "buildable via extensions" (unverified beyond that claim). Docs: .../docs/extensions.md, .../docs/packages.md
- **No HTTP/WebSocket server mode** documented; embedding over HTTP would be self-built around the SDK or RPC subprocess.

## 3. Streaming

Event vocabulary (identical names across SDK, RPC, JSON modes; sources: rpc.md, json.md, sdk.md):

- Lifecycle: `agent_start`, `agent_end` (`{messages}`, `willRetry`), `agent_settled`, `turn_start`/`turn_end`, `queue_update`.
- Token granularity: `message_start`/`message_update`/`message_end`. `message_update` carries `assistantMessageEvent` deltas: `text_start/text_delta/text_end`, `thinking_start/thinking_delta/thinking_end`, `toolcall_start/toolcall_delta/toolcall_end`. **Deltas only — no cumulative snapshot**; `message_end.message` is authoritative.
- Tools: `tool_execution_start` `{toolCallId, toolName, args}` → `tool_execution_update` (accumulated `partialResult`) → `tool_execution_end` `{toolCallId, toolName, result, isError}` — correlated by `toolCallId`.
- Also: `compaction_start/end`, `auto_retry_start/end`, `extension_error`, bash-output streaming (`bash_execution_update`).

## 4. Lifecycle

- **Persistence**: sessions auto-save as **append-only JSONL tree files** (entries with `id`/`parentId`, enabling in-place branching) under `~/.pi/agent/sessions/` organized by cwd; override via `--session-dir` or `PI_CODING_AGENT_SESSION_DIR`; `--no-session` disables. Sources: coding-agent README; rpc.md.
- **Create/resume**: CLI `-c` (continue recent), `-r`/`--resume`, `--session <path|id>`, `--fork`; SDK `SessionManager.create(cwd)` / `continueRecent(cwd)` / `open(path)` / `inMemory()` / `list(cwd)`; RPC `new_session` (optional `parentSession`), `switch_session`, `fork(entryId)`, `clone`, `get_entries` with durable `since` cursor (includes pre-compaction history). Sources: README; sdk.md; rpc.md.
- **Cancellation**: RPC `abort` command; SDK `session.abort()`; `stopReason: "aborted"` on assistant messages (rpc.md, sdk.md).
- **Compaction**: `compact` + `set_auto_compaction`; reasons `manual|threshold|overflow` (rpc.md).

## 5. Tool calls & approvals

- Built-in tools: `read`, `bash`/`powershell`, `edit`, `write`, `grep`, `find`, `ls`; curate via `--tools`/`-t`, `--exclude-tools`, `--no-builtin-tools` (README).
- **Host-injected tools** = TypeScript **extensions**: `pi.registerTool({name, parameters (TypeBox), execute(toolCallId, params, signal, onUpdate, ctx)})`, loaded from `~/.pi/agent/extensions/`, `.pi/extensions/`, or `-e <source>` per run; can override built-ins by name; `onUpdate` gives incremental tool results. Doc: .../docs/extensions.md
- **Approval gate**: `pi.on("tool_call", handler)` fires before execution and **can block**: return `{block: true, reason}`; handler may mutate `event.input` in place. There is **no dedicated pause/resume API** — pausing is achieved by awaiting UI inside the handler (`ctx.ui.confirm/select/input/editor`), which in RPC mode surfaces to the host as an `extension_ui_request` JSON line (`select`/`confirm`/`input`/`editor`) that blocks until the host writes `extension_ui_response`; optional timeout auto-resolves. Sources: extensions.md; rpc.md.
- `tool_result` event allows result patching (extensions.md).

## 6. Auth & model config

- API keys via env (`ANTHROPIC_API_KEY`, etc.), `--api-key <key>` (overrides env), or `/login` subscription OAuth (Claude Pro/Max, ChatGPT, Copilot). Providers: Anthropic, OpenAI, Google, Bedrock, xAI, Groq, ... Source: coding-agent README; .../docs/providers.md.
- Per-run host control: spawn subprocess with chosen env vars / `--api-key` / `--provider` / `--model`; custom OpenAI/Anthropic-compatible endpoints via `~/.pi/agent/models.json` (`PI_CODING_AGENT_DIR` relocates the whole config dir — usable for per-host isolation). RPC `set_model {provider, modelId}` switches mid-session (rpc.md).
- Fully offline mode: `--offline` / `PI_OFFLINE=1` (README).

## 7. Fit notes (embedding behind WeKnora's Go `AgentEngine`)

- **Transport choice**: subprocess `pi --mode rpc` is the natural Go integration (strict JSONL framing — Go's `bufio.Scanner` is compliant; note the doc's warning that Node `readline` is not). In-process SDK would require a permanent Node sidecar service; only worth it if WeKnora wants multi-session runtime management without process-per-session.
- **Event mapping is clean**: `message_update.text_delta` → "thought" SSE deltas; `toolcall_*` + `tool_execution_start/end` → "tool_call"/"tool_result"; final `message_end`/`agent_end` assistant text → "final_answer"; `agent_end` with `stopReason:"error"`/`extension_error` → "error". References would have to flow through a custom tool result.
- **Approvals need a bridge extension**: WeKnora's human-approval flow maps to a shipped `-e` TypeScript extension whose `tool_call` handler forwards `ctx.ui.confirm` requests over RPC (`extension_ui_request`) to Go, which relays to the user and writes back `extension_ui_response`. Workable but is an extra artifact to version/ship, and blocking semantics hold a subprocess handler open.
- **Dual persistence**: pi insists on its own JSONL tree sessions; WeKnora must either run `SessionManager.inMemory()`/`--no-session` and mirror events into its DB, or reconcile via `get_entries` `since` cursors. Resuming a WeKnora session means `switch_session` on a long-lived process or `--session <path>` on a fresh spawn.
- **WeKnora's RAG tools need a callback**: custom tools execute inside the pi process, so knowledge-search tools must be a TS extension calling WeKnora over HTTP (or WeKnora proxies tool calls out — no server-side tool-call webhook exists).
- **Ops risk**: 0.x versioning with weekly releases and no documented protocol-stability guarantees; pin the version, and prefer the standalone binary build (`./scripts/build-binaries.sh`) to avoid host Node drift (monorepo README).

## 8. Open questions

- Does RPC/SDK expose setting a **custom system prompt / initial history** per session (`new_session` shows only `parentSession`)? Needs source check in `rpc-types.ts`.
- Can an API key be rotated mid-session, or only at spawn (`--api-key`/env)? `registerProvider` (extensions.md) may allow it.
- One long-lived RPC process per WeKnora session vs per-turn spawn: startup cost and memory profile of a resident `pi` process are unmeasured.
- JSON mode stdin prompting for multi-turn is undocumented; RPC mode is the safer multi-turn transport.
- Whatever happened to `pi-web-ui` (absent from current README) — was there a reusable web frontend, and does anything replace it?
- Confirm whether the README's "MCP integration" capability refers to a specific shipped extension or is aspirational (packages gallery at pi.dev/packages unverified).

## Source index

- Monorepo README: https://github.com/earendil-works/pi (redirect from https://github.com/badlogic/pi-mono)
- Coding-agent README: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md
- RPC protocol: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/rpc.md
- SDK guide: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md
- JSON mode: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/json.md
- Extensions: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md
- Packages: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/packages.md
- npm registry: https://registry.npmjs.org/@earendil-works/pi-coding-agent
