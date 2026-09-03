# OpenAI Codex CLI as an External Agent Runtime — Research Findings

Researched 2026-09-03 from primary sources (github.com/openai/codex @ main / rust-v0.153.0, npm, developers.openai.com/codex).

## 1. Identity

- Repo: https://github.com/openai/codex — "Lightweight coding agent that runs in your terminal"; the CLI is a Rust binary (tags `rust-v*`, assets like `codex-aarch64-apple-darwin.tar.gz`). Apache-2.0. https://github.com/openai/codex
- Version/cadence: stable `0.153.0` published 2026-09-03; stable point releases roughly daily plus multiple `-alpha.N` pre-releases per day, ~162 assets per release, CI-published (github-actions). Same-day match between npm `@openai/codex` and `@openai/codex-sdk` (both 0.153.0). https://github.com/openai/codex/releases ; https://www.npmjs.com/package/@openai/codex ; https://www.npmjs.com/package/@openai/codex-sdk
- Companions: IDE extension (VS Code/Cursor/Windsurf), desktop app (`codex app`), Codex Web. https://github.com/openai/codex

## 2. Integration surfaces

Subcommands confirmed in `codex-rs/cli/src/main.rs`: `exec` (alias `e`), `review`, `login/logout`, `mcp`, `mcp-server`, `app-server` (experimental), `resume`, `apply` (alias `a`), `queue`, `sandbox`, `agents`, `remote-control`, `exec-server` (experimental), `execpolicy`, `debug`. **`codex proto` no longer exists** — it was removed in favor of `app-server`. https://github.com/openai/codex/blob/main/codex-rs/cli/src/main.rs

- **`codex exec`** (non-interactive): `codex exec --json [--output-schema FILE] [--sandbox workspace-write|danger-full-access] [--ephemeral] [-o FILE] [--skip-git-repo-check] [--ignore-user-config] [--ignore-rules] "<prompt>"`. `--json` emits JSONL events (see §3); `--full-auto` is deprecated. Resume: `codex exec resume --last` or `codex exec resume <SESSION_ID>`. https://developers.openai.com/codex/noninteractive
- **`codex app-server`** (experimental, but the strategic surface — "the interface Codex uses to power rich interfaces"; the VS Code extension, TUI, `codex agents`, and the Codex plugin for Claude Code are built on it): bidirectional JSON-RPC 2.0 over stdio (newline-delimited JSONL), or experimental `--listen ws://IP:PORT` / unix socket. Methods: `initialize`/`initialized`; `thread/start|resume|fork|list|read|archive|delete|unarchive|compact/start`, `turn/start|steer|interrupt`, `command/exec*`, `process/*`, `fs/*`, `mcpServer/tool/call`, `skills/list`, `config/read`, `review/start`. https://developers.openai.com/codex/app-server ; https://github.com/openai/codex/blob/main/codex-rs/app-server
- **`codex mcp-server`** (Codex as MCP server, stdio): **deprecated** — "Use the Codex app server". Exposes two tools: `codex` (params: `prompt`, `approval-policy`, `sandbox`, `model`, `cwd`, `base-instructions`, `config` overrides, ...) and `codex-reply` (`prompt` + `threadId`); continuation via `structuredContent.threadId`. https://developers.openai.com/codex/mcp-server (via https://developers.openai.com/codex/mcp-server)
- **TypeScript SDK `@openai/codex-sdk`**: thin wrapper that spawns the CLI and exchanges JSONL over stdin/stdout (Node 18+). API: `new Codex({env, config, configOverrides, baseUrl})`, `codex.startThread({workingDirectory, skipGitRepoCheck})`, `codex.resumeThread(id)` (rebuilds from `~/.codex/sessions`), `thread.run(input, {outputSchema})` (buffered; returns `turn.finalResponse`, `turn.items`), `thread.runStreamed(input)` (async generator). https://www.npmjs.com/package/@openai/codex-sdk ; https://developers.openai.com/codex/sdk
- **MCP client**: `~/.codex/config.toml` `[mcp_servers.<name>]` (stdio `command/args/env` or streamable HTTP `url` + `bearer_token_env_var`/OAuth); CLI mgmt `codex mcp add|list|login`. https://developers.openai.com/codex/mcp (redirects to https://learn.chatgpt.com/docs/extend/mcp?surface=cli)

## 3. Streaming

- `codex exec --json` JSONL event types: `thread.started` (carries `thread_id`), `turn.started`, `item.started`/`item.updated`/`item.completed`, `turn.completed` (with usage), `turn.failed`, `error` (with message). Item types: agent messages, reasoning, command executions (with command + aggregated output), file changes, MCP tool calls, web searches, plan updates. https://developers.openai.com/codex/noninteractive
- App-server notifications (finer-grained): `thread/started`, `turn/started`, `item/started`, `item/completed`, `item/agentMessage/delta` (token-level agent message deltas), `command/exec/outputDelta` and `process/outputDelta` (streaming command output), `turn/completed`, `thread/status/changed`, `thread/tokenUsage/updated`, `thread/queue/changed`, `fs/changed`, etc. https://developers.openai.com/codex/app-server
- SDK `runStreamed` surfaces the same item/turn events as structured JS objects (`item.completed`, `turn.completed` with `event.usage`). https://www.npmjs.com/package/@openai/codex-sdk
- WeKnora mapping is direct: thought→agentMessage/reasoning deltas, tool_call→commandExecution/McpToolCall item started, tool_result→item completed w/ output, final_answer→turn completed + final agent message, error→turn.failed/error event. References would need to come from MCP tool items (no first-class references event).

## 4. Lifecycle

- Threads = persisted conversations; Turns = one exchange; Items = messages/commands/file edits (app-server primitives). `thread/resume` and `codex exec resume <SESSION_ID>` / `--last`; SDK `resumeThread(id)` reads `~/.codex/sessions`. https://developers.openai.com/codex/app-server ; https://www.npmjs.com/package/@openai/codex-sdk ; https://developers.openai.com/codex/noninteractive
- Cancellation: `turn/interrupt` (app-server); `turn/steer` injects user input mid-turn (queue via `codex queue` in CLI). Fork/compact/archival supported (`thread/fork`, `thread/compact/start`). https://developers.openai.com/codex/app-server
- Rollout/session files live under `~/.codex/sessions` (SDK doc); WeKnora would keep its own persistence and treat these as opaque recovery state.

## 5. Approvals & sandbox

- Legacy CLI config: `approval_policy` (untrusted / on-failure / on-request / never) and `sandbox_mode` (read-only / workspace-write / danger-full-access) — docs now steer to **permission profiles**: built-ins `:read-only`, `:workspace`, `:danger-full-access`, plus named `[permissions.<name>]` with `filesystem` read/write/deny rules, `workspace_roots`, and `network.enabled`/`network.domains` (enforced via `features.network_proxy`). https://developers.openai.com/codex/permissions ; app-server sandbox values per turn: `readOnly`, `workspaceWrite`, `dangerFullAccess`, `externalSandbox`. https://developers.openai.com/codex/app-server
- **Programmatic approvals: yes, via app-server only.** Server→client JSON-RPC requests: `item/commandExecution/requestApproval` (payload: `itemId`, `threadId`, `turnId`, `command`, `cwd`, `reason`, `availableDecisions`, network context) and `item/fileChange/requestApproval`; client answers `accept` | `acceptForSession` | `decline` | `cancel` (+ experimental `acceptWithExecpolicyAmendment`); resolution confirmed by `serverRequest/resolved`. Per-turn `approvalPolicy` shown: `never`, `unlessTrusted`, `onRequest`. https://developers.openai.com/codex/app-server
- `codex exec` cannot intercept approvals — it runs to completion with no interactive prompt (that's why `--sandbox` defaults to read-only; bypass flags exist for CI). https://developers.openai.com/codex/noninteractive

## 6. Auth

- Two modes: "Sign in with ChatGPT" (subscription credit) or API key (usage-based). Login: `codex login --with-api-key` with the key piped on stdin (`printenv OPENAI_API_KEY | codex login --with-api-key`); enterprise `codex login --with-access-token` (CODEX_ACCESS_TOKEN). Credentials cached plaintext at `~/.codex/auth.json` or OS keyring (`cli_auth_credentials_store = file|keyring|auto`). https://developers.openai.com/codex/auth
- Per-invocation by a host: `CODEX_API_KEY` env var is supported **only in `codex exec`** (documented on the non-interactive page). Custom model providers take `env_key = "<ENV_VAR>"` read each run; `requires_openai_auth = true` reuses the stored login instead. https://developers.openai.com/codex/noninteractive ; https://developers.openai.com/codex/auth
- Caveat for WeKnora: ChatGPT-plan auth is a shared on-disk state (`$CODEX_HOME/auth.json`) — per-user credential isolation means one `CODEX_HOME` dir per credential or API-key-only mode.

## 7. Fit notes (embedding behind WeKnora's Go `AgentEngine`)

- **Best transport: `codex app-server` over stdio JSONL from Go.** One long-lived subprocess (`exec.Command("codex", "app-server")`), `initialize` handshake, then `thread/start` + `turn/start`; map notifications 1:1 onto WeKnora's SSE event types. It is the only surface with token-level deltas, streaming exec output, mid-turn steer/interrupt, and in-band approval requests. No Node needed (unlike the TS SDK); native single static binary per platform, installed from releases.openai.com or npm. https://developers.openai.com/codex/app-server
- **Simplest fallback: `codex exec --json`** per turn — stateless subprocess, JSONL parse → SSE bridge; but no deltas (item granularity only), no programmatic approvals (must run `approval_policy=never` in a sandbox), and each resume re-parses rollout files. Acceptable v1, weak for interactive UX.
- **Skip `codex mcp-server`** (deprecated) and **`@openai/codex-sdk`** (adds a Node sidecar for no capability the raw app-server doesn't expose).
- Persistence mismatch: Codex owns rollout/session files and thread IDs; WeKnora must store `threadId` per WeKnora session and use `thread/resume`, accepting that Codex's history format is opaque and versioned with the binary.
- Auth is the awkward part: no clean "take this credential for this call" except `CODEX_API_KEY` in exec mode and `CODEX_HOME` redirection; ChatGPT-plan tokens are shared file state, so multi-tenant WeKnora likely standardizes on API keys or a single service credential.
- Version churn: near-daily releases with an explicitly "experimental" app-server and a permissions system mid-migration from `approval_policy`/`sandbox_mode` — pin the binary version and add a protocol smoke test to CI; JSON-RPC method names have already been renamed once (`newConversation`/`sendUserTurn` → `thread/*`/`turn/*`).

## 8. Open questions

1. Is `codex app-server`'s wire protocol versioned/discovery-documented anywhere machine-readable (schema export), or is the markdown doc the only contract?
2. Does `CODEX_API_KEY` work with ChatGPT-plan entitlements or only platform billing (pricing differs)? Per-user ChatGPT auth from a server may violate plan terms — needs legal/ToS check.
3. Can WeKnora inject its own MCP servers (its tools) per-thread via app-server turn/thread params, or only via global `config.toml`? (`mcpServer/tool/call` suggests host-side tool calls exist but the client-vs-server split is unclear.)
4. What maps to WeKnora's `references` event? No first-class citation/reference event was found — likely must be extracted from item payloads or a post-turn query.
5. Windows/ARM support and sandboxing fidelity (Seatbelt vs Landlock vs no-sandbox) across deployment targets — sandbox guarantees differ per OS.
6. Resource model: one app-server daemon vs process-per-session; `--listen ws://` is "experimental and unsupported" — is stdio-per-session acceptable at WeKnora's concurrency?
