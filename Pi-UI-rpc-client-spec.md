# Pi-UI Rust RPC Client — Planning Spec

Status: planning. No implementation yet — review before building.

Sources of truth (pinned pi version, currently 0.80.10):

- Protocol types: `packages/coding-agent/src/modes/rpc/rpc-types.ts`
- Framing: `packages/coding-agent/src/modes/rpc/jsonl.ts`
- Reference client semantics: `packages/coding-agent/src/modes/rpc/rpc-client.ts`
- Behavioral docs: `packages/coding-agent/docs/rpc.md`
- Event payloads: `packages/coding-agent/src/core/agent-session.ts` (`AgentSessionEvent`), `packages/agent/src/types.ts` (`AgentEvent`)

## 1. Purpose

A Rust crate (`pi-rpc`, inside the Tauri app) that owns a `pi --mode rpc` subprocess per session and exposes:

- typed async command methods (request → response)
- a broadcast stream of session events for the Svelte frontend (via Tauri events)
- an extension-UI bridge that surfaces extension dialogs as native GUI dialogs

Transport lives behind a trait so a socket transport can replace stdio later (see Pi-UI-project-idea.md, Transport decision).

## 2. Process model and framing

- Spawn: `<pi-binary> --mode rpc [--provider X] [--model Y] [--name N] [--session-dir P] [-e <bridge-extension>]` with the project directory as `cwd`, piped stdin/stdout/stderr.
- One subprocess per open session tab. `new_session`, `switch_session`, `fork`, `clone` allow sequential sessions in one process, but per-tab processes give independent crash isolation and let each tab pin its own cwd. (Confirm during implementation; either is expressible.)
- Framing is strict JSONL: LF (`\n`) is the only record delimiter. Strip one optional trailing `\r`. Do NOT split on U+2028/U+2029 — they are valid inside JSON strings (Node `readline` is explicitly non-compliant; a naive Rust `lines()` iterator is fine, but a UTF-8-safe buffered splitter must handle multi-byte chars across chunk boundaries).
- stdout carries three kinds of lines, distinguished by the `type` field:
  - `type: "response"` — command response (correlated by `id`)
  - `type: "extension_ui_request"` — extension dialog/widget request (own `id` namespace)
  - everything else — an `AgentSessionEvent` (never has an `id`)
- Non-JSON lines on stdout: ignore (log at debug level). stderr: capture into a ring buffer, include in error reports.
- No startup handshake or banner exists. The TS client just sleeps 100ms. Spec: consider the process ready when the first command (`get_state`) succeeds; use that as the readiness probe instead of a timer.

## 3. Request/response semantics

- Client generates ids (`req_N`); the response echoes `id` plus `command` and a `success` boolean.
- Success responses carry command-specific `data`; failure is `{ type: "response", command, success: false, error: string }` — any command can fail.
- Client-side timeout: 30s per request (matches the TS client). Process exit/stdin failure rejects all pending requests.
- `prompt`/`steer`/`follow_up` are ack-only: the response means "accepted", the actual work arrives as events. Idle is signaled by `agent_settled` (NOT `agent_end` — retry/compaction/queued continuations may follow `agent_end`).
- `new_session`, `switch_session`, `fork`, `clone` return `{ cancelled: true }` when an extension vetoes — the GUI must treat these as soft failures, not errors.
- `get_entries(since?)` is the incremental transcript loader: the `since` entry id acts as a durable cursor.
- `fork(entryId)` returns the forked message text; `get_fork_messages` lists forkable messages.

## 4. Command surface (31 commands)

| Group | Commands |
|---|---|
| Prompting | `prompt(message, images?, streamingBehavior?)`, `steer(message, images?)`, `follow_up(message, images?)`, `abort`, `new_session(parentSession?)` |
| State | `get_state` → `RpcSessionState` |
| Model | `set_model(provider, modelId)`, `cycle_model`, `get_available_models` |
| Thinking | `set_thinking_level(level)`, `cycle_thinking_level` |
| Queue modes | `set_steering_mode("all"\|"one-at-a-time")`, `set_follow_up_mode(...)` |
| Compaction | `compact(customInstructions?)`, `set_auto_compaction(enabled)` |
| Retry | `set_auto_retry(enabled)`, `abort_retry` |
| Bash | `bash(command, excludeFromContext?)`, `abort_bash` |
| Session | `get_session_stats`, `export_html(outputPath?)`, `switch_session(path)`, `fork(entryId)`, `clone`, `get_fork_messages`, `get_entries(since?)`, `get_tree`, `get_last_assistant_text`, `set_session_name(name)` |
| Messages | `get_messages` |
| Commands | `get_commands` → extension commands + prompt templates + skills |

`RpcSessionState`: model, thinkingLevel, isStreaming, isCompacting, steering/followUp modes, sessionFile/sessionId/sessionName, autoCompactionEnabled, messageCount, pendingMessageCount.

`SessionStats` (for the cost dashboard): message counts, token breakdown (input/output/cacheRead/cacheWrite/total), `cost` (dollars), `contextUsage`.

## 5. Event model

From `AgentSessionEvent` (session-level) + `AgentEvent` (core). All serialized as flat JSON objects tagged by `type`.

| Event | Payload | GUI use |
|---|---|---|
| `agent_start` | — | spinner on |
| `agent_end` | `messages`, `willRetry` | run boundary (not idle) |
| `agent_settled` | — | idle signal; notify user; resolve prompt futures |
| `turn_start` / `turn_end` | — / `message`, `toolResults` | turn grouping, per-turn usage/cost from `message.usage` |
| `message_start` / `message_end` | `message` | transcript append/finalize |
| `message_update` | `message`, `assistantMessageEvent` | streaming render |
| `tool_execution_start` | `toolCallId`, `toolName`, `args` | tool card open (correlate by `toolCallId`) |
| `tool_execution_update` | + `partialResult` (cumulative) | live tool output — replace, don't append |
| `tool_execution_end` | + `result`, `isError` | tool card close, diff view from `result.details` |
| `queue_update` | `steering[]`, `followUp[]` | steering queue UI |
| `compaction_start` / `compaction_end` | `reason`, `result?`, `aborted`, `willRetry`, `errorMessage?` | context gauge, compaction notices |
| `auto_retry_start` / `auto_retry_end` | `attempt`, `maxAttempts`, `delayMs`, `errorMessage` / `success`, `finalError?` | retry banners |
| `entry_appended` | `entry: SessionEntry` | keep transcript/tree in sync without re-fetch |
| `session_info_changed` | `name?` | tab titles |
| `thinking_level_changed` | `level` | header indicator |
| `extension_error` | `extensionPath`, `event`, `error` | diagnostics surfacing (RPC-mode-only event, not in the core union) |

Streaming semantics that matter for the renderer:

- `message_update.assistantMessageEvent` deltas: `start`, `text_start/delta/end`, `thinking_start/delta/end`, `toolcall_start/delta/end`, `done`, `error`. Each carries a cumulative `partial` message — simplest correct renderer re-renders `partial` and treats deltas as invalidation hints.
- `tool_execution_update.partialResult` is accumulated output, not a delta: replace display on each update.

## 6. Extension UI bridge

- Dialog methods (`select`, `confirm`, `input`, `editor`): emit `extension_ui_request`, block until the GUI replies `extension_ui_response` with the matching `id`. Reply shapes: `{ id, value: string }`, `{ id, confirmed: boolean }`, or `{ id, cancelled: true }`.
- Fire-and-forget (`notify`, `setStatus`, `setWidget`, `setTitle`, `set_editor_text`): display or ignore; no reply.
- `timeout` on dialog requests is enforced agent-side (auto-resolves with a default) — the GUI does not track timeouts.
- Degraded in RPC mode (agent-side, not our problem to polyfill): `custom()` → undefined, footer/header/working-indicator no-ops, theme getters empty. Extensions see `ctx.mode === "rpc"` and `ctx.hasUI === true`.
- GUI mapping: select/confirm/input/editor → native modals; notify → toast; setStatus/setWidget → dockable panel slots; setTitle → window/tab title; set_editor_text → composer text.

## 7. Rust crate shape

- `transport` trait: `send(line)`, stream of incoming lines, `kill()`. Stdio implementation via `tokio::process`; socket impl later.
- `PiRpcClient`: pending-request map (`id` → oneshot sender), command methods mirroring section 4, `subscribe()` returning a `tokio::sync::broadcast` receiver of events, `extension_ui` sub-stream for the bridge.
- serde strategy:
  - Commands: `#[serde(tag = "type")]` enum, `id` skipped when None.
  - Responses: tagged on `type: "response"` but discriminated by `command` + `success` — deserialize to an intermediate `serde_json::Value`, then into a per-command payload. Attempt the typed path only after `success` check.
  - Events: `#[serde(tag = "type")]` enum with `#[serde(other)] Unknown` fallback for forward compatibility (new pi versions may add event types — never hard-fail on unknown lines).
  - Message payloads (`AgentMessage`, `SessionEntry`, `Model`, tool `details`): keep as typed-where-stable, `serde_json::Value` where extension-defined (tool details are arbitrary).
- Tauri wiring: one `PiRpcClient` per tab, events forwarded as Tauri events on a per-tab channel; commands exposed as Tauri commands.

## 8. Type sync strategy

- `rpc-types.ts` in the published `@earendil-works/pi-coding-agent` package is the source of truth; the app pins a supported pi version range (see Runtime management in the idea doc).
- Hand-maintain the Rust types (surface is small and stable) + guard with fixture tests: record real JSONL transcripts from the pinned pi binary and round-trip them in CI.
- Version gate: on runtime install/update, record `pi --version`; refuse or warn outside the supported range.

## 9. Gaps and open questions (RPC-mode limitations found during planning)

1. **No protocol version / handshake.** Mitigation: version gate at spawn (section 8). Candidate upstream ask: a `get_protocol_version` command.
2. **No `list_sessions` command.** The session sidebar needs a session list per cwd. Options: (a) read the JSONL session files directly in Rust — format is documented (`docs/session-format.md`), files under `~/.pi/agent/sessions/<project-slug>/`; (b) upstream a `list_sessions` command. Lean: (a) for MVP (read-only, no protocol dependency), upstream (b) later.
3. **Auth/login is not bridged in RPC mode** (no auth handling in `rpc-mode.ts`). Assumption: user logs in once via the pi CLI/TUI; credentials live in the shared `~/.pi/agent` config and RPC sessions reuse them. Open question: whether OAuth device-code flows can be driven from the GUI later (would need upstream bridging of `AuthInteraction`).
4. **Tool approval is not exposed over RPC** (`beforeToolCall` is an in-process hook). Plan: ship a small "pi-ui bridge" extension with the app, loaded via `-e`, that hooks the blockable `tool_call` extension event and raises `ctx.ui.confirm(...)` — it arrives at the GUI as an `extension_ui_request` and renders as a native approval dialog. This also becomes the GUI's permission policy layer (allow/deny/always-allow rules), with zero upstream changes.
5. **Slash commands**: built-in TUI commands are unavailable (TUI-only); `get_commands` covers extension/prompt/skill commands — invoked by sending `/name args` as a `prompt`. GUI-native panels replace the built-ins.

## 10. Testing plan

- Fixture tests: recorded JSONL from the pinned pi binary (happy path, tool calls, compaction, retry, extension dialogs, CRLF input, multi-byte UTF-8 across chunk boundaries, very large bash-output lines).
- Mock-subprocess tests for client logic (timeouts, exit mid-request, unknown event tolerance).
- Integration test spawning the real pinned binary: spawn → `get_state` → `prompt` → collect events until `agent_settled`.
- Windows: CRLF tolerance, path handling for `--session-dir`, process kill semantics (no SIGTERM on Windows — verify grace period behavior).
