# Pi-UI project idea

Premise: A lightweight and extensible GUI for the Pi coding harness.

- The goal is to make something that can eventually replicate a Codex or Claude desktop experience with Pi. The biggest benefit with Pi is how extensible it is, and to some degree this GUI needs to be able to take advantage of that.
- Current Phase - Pre-planning and brainstorming.

## Architecture decision

- **Path A: RPC subprocess.** The GUI spawns `pi --mode rpc` and talks JSONL over stdin/stdout, using Pi's RPC protocol as the only transport.
- **Shell: Tauri** (Rust + system webview). No Electron.
- Pi is consumed as-is via an **app-managed runtime**: the GUI downloads, installs, and updates its own pi binary — revving pi does not require an app rebuild. Repo role: consumer of published Pi artifacts (GitHub Release binaries, npm metadata), not a `packages/gui` in the fork. The Pi copy in this repo is for reference and for prototyping upstreamable changes.
- RPC client planning spec: [Pi-UI-rpc-client-spec.md](Pi-UI-rpc-client-spec.md).

## Why Path A works: Pi's mode architecture

- Pi's core is mode-agnostic. Four modes sit over the same core: interactive (TUI), print, json, rpc. Selected in `packages/coding-agent/src/main.ts` (`resolveAppMode()`).
- `AgentSession` (`packages/coding-agent/src/core/agent-session.ts:284`) is shared by all modes. The TUI (`src/modes/interactive/interactive-mode.ts:2810`) is a pure event consumer; the core never calls into the UI. The GUI does the same: subscribe and render.
- Key `AgentSession` API: `prompt()`, `steer()`, `followUp()`, `abort()`, `setModel()`, `setThinkingLevel()`, `compact()`, `navigateTree()`, `bindExtensions()`, `subscribe(listener)`.
- `AgentSessionEvent` stream: `agent_start/end`, `turn_start/end`, `message_start/update/end` (text/thinking/toolcall deltas), `tool_execution_start/update/end`, plus session extras: `agent_settled`, `queue_update`, `compaction_start/end`, `auto_retry_start/end`, `entry_appended`, `session_info_changed`, `thinking_level_changed`.

## RPC mode details (the Path A transport)

- Started with `pi --mode rpc` (entry: `packages/coding-agent/src/rpc-entry.ts`). JSONL over stdio. Docs: `packages/coding-agent/docs/rpc.md`; protocol types: `src/modes/rpc/rpc-types.ts`.
- Reference client: `RpcClient` (`src/modes/rpc/rpc-client.ts:55`). It is a Node/TS subprocess client; for Tauri the protocol gets reimplemented in Rust (JSONL stdio, types are simple — `rpc-types.ts` is the source of truth).
- Command surface: `prompt` / `steer` / `follow_up` / `abort`; model & thinking control; `compact`; retry; bash; session ops (`switch_session`, `fork`, `clone`, `get_entries` with durable cursor, `get_tree`, `get_fork_messages`, `set_session_name`); `get_commands` (extension commands + skills + prompt templates).
- Events: the same `AgentSessionEvent` stream serialized as JSON lines.
- Extension UI bridge: extension dialogs arrive as `extension_ui_request` JSON; the GUI answers with `extension_ui_response`. This is the template for rendering extension UI natively.
- Not available over RPC: built-in TUI slash commands (`/settings`, `/hotkeys`, model/session selectors — they live in `interactive-mode.ts`). They are thin wrappers over exported APIs; reimplement as GUI-native panels.
- No HTTP/WebSocket server mode exists upstream. If wanted later, wrap RPC/SDK ourselves (candidate to prototype in this repo and upstream).

## Transport decision: stdio RPC over server mode

Chosen: per-session stdio subprocess. A WebSocket/HTTP server mode was considered and deferred.

Why stdio wins for this app:

- It exists today, documented and maintained upstream (`docs/rpc.md`, `rpc-types.ts`, typed `RpcClient`). A server mode does not exist — it would be net-new upstream work before the app could even start.
- Matches the Tauri model: Rust spawns and owns the child process; stdio pipes are the natural sidecar pattern. No port allocation, no auth tokens, no daemon lifecycle.
- Security: nothing listens on a socket. A local server that can run bash and edit files needs authentication against other local processes — design surface Pi core deliberately does not have.
- Fits the managed runtime: per-session subprocesses mean an update takes effect on the next session spawn. A long-running daemon would pin the *old* binary and force a daemon restart (killing its sessions) on every runtime update.
- Session sharing with the TUI is already covered by the shared JSONL files — resume the same session from either frontend.

What a server mode would add (revisit triggers):

- Live multi-client attach (GUI + TUI watching the same running session simultaneously)
- Sessions/agents that outlive the app (daemon model, background agents)
- Remote/headless driving

If these become real requirements: prototype in this repo's Pi copy and PR upstream. Hedge for later: keep the Rust client's transport behind a trait (stdio implementation now, socket framing later) so the command/event surface — identical either way — stays transport-agnostic.

## Session management

- `SessionManager` (`packages/coding-agent/src/core/session-manager.ts:851`): append-only JSONL, tree-structured via `id`/`parentId`. Factories: `create/open/continueRecent/inMemory/forkFrom`. Listing: `SessionManager.list(cwd)` / `listAll()`.
- Session files live in `~/.pi/agent/sessions/<project-slug>/` by default. GUI and TUI share the same session files — unified history across both frontends.
- Tree API: `getEntries/getTree/getPath/getChildren`, `branch()`, `branchWithSummary()`, labels. Fork-from-any-message is the basis for the visual branch graph.

## Extension system compatibility

- Loading happens in the pi subprocess, identical to the TUI: `~/.pi/agent/extensions/*.ts` (global), `.pi/extensions/*.ts` (project, trust-gated), npm/git packages via `settings.json`, CLI `-e`.
- Extensions get ~30 events plus `registerTool/registerCommand/registerShortcut/registerFlag`, `sendMessage`, `appendEntry` (session-persisted state), `exec`, `registerProvider`.
- UI surface is the `ExtensionUIContext` interface (`src/core/extensions/types.ts:128`): `select/confirm/input/notify/editor`, `setStatus`, `setWidget`, `setFooter/setHeader`, `setTitle`, `custom()` overlays, theme accessors. Over RPC these serialize as `extension_ui_request`; the GUI renders them as native widgets.
- TUI-only bits that degrade: `renderCall/renderResult` returning pi-tui Components, `custom()` component overlays, component-factory widgets. Strategy: no-op or map to GUI equivalents; document a "GUI capability" subset for extension authors.
- `ExtensionMode` is `"tui" | "rpc" | "json" | "print"` — extensions can branch on mode; the GUI presents as `"rpc"` unless upstream adds a mode.

## Carries over for free vs needs GUI-native work

Free (via RPC/subprocess):

- Extensions, skills, prompt templates, slash commands (via `get_commands`)
- Sessions (shared JSONL files), `settings.json`, model/auth registry
- Auth flows: `AuthInteraction { prompt, notify }` callbacks → native dialogs
- Per-turn cost: `usage.cost` on assistant messages
- Tool approval: `beforeToolCall` hook → GUI confirm dialogs (also the basis for a GUI-side permission policy layer, since Pi core has none)
- HTML session export (`packages/coding-agent/src/core/export-html/`)

GUI-native reimplementation:

- Themes (terminal-color JSON does not map; own theme system)
- Keybindings (own system, optionally mirroring Pi defaults)
- Built-in TUI commands (`/settings` etc.) as native panels
- Transcript rendering: markdown, syntax highlighting, diff view for the edit tool

## Tauri specifics

- Rust core owns the pi subprocess: spawn, stdio, JSONL encode/decode, event fan-out to the webview via Tauri events; webview invokes Rust commands for RPC requests.
- Pi ships as an npm package and as standalone Bun-compiled binaries (see release process in `AGENTS.md`). A Tauri sidecar can bundle the standalone binary, so the app does not require Node on the user's machine.
- Frontend: **Svelte** (decided).

## Runtime management

Problem: a frozen bundled pi means every pi update requires an app release. Solution: the GUI manages its own pi runtime, installable and updatable from within the app.

- **Runtime location**: app data dir, versioned — `<app-data>/runtime/<version>/` with a `current` pointer. Versioned dirs give atomic swaps and rollback; stale dirs are cleaned up once no running session uses them.
- **First-run install**: download the platform archive from Pi's GitHub Release (`pi-darwin-arm64.tar.gz`, `pi-darwin-x64.tar.gz`, `pi-linux-x64.tar.gz`, `pi-linux-arm64.tar.gz`, `pi-windows-x64.zip`, `pi-windows-arm64.zip`), verify against the release's `SHA256SUMS`, extract. These are standalone Bun binaries — no Node required on the user's machine.
- **Update check**: `https://pi.dev/api/latest-version` (the same endpoint `pi update --self` queries) or the GitHub Releases API. Surface "update available" in the UI; user-triggered by default, optional auto-update.
- **Update flow**: download to a *new* versioned dir, verify checksum, flip `current`. Running sessions keep their already-spawned binary; new sessions pick up the new version. Never overwrite in place — this sidesteps Windows' locked-executable problem that forces Pi's own npm self-update into quarantine hacks (`packages/coding-agent/src/utils/windows-self-update.ts`).
- **Protocol compatibility is the coupling cost**: the Rust RPC client implements a protocol that is versioned with pi. The GUI declares a supported pi version range, gates auto-updates on that range, and warns on mismatch. Source of truth: `rpc-types.ts` in the published `@earendil-works/pi-coding-agent` package (pinned version; keep the Rust serde types in sync, ideally with a sync test).
- Optional escape hatch: "use system pi" mode pointing at the user's own install, with a version-compat warning.
- Do not shell out to `pi update --self`: it is npm-oriented and unavailable for binary installs. The GUI owns the download.

## Feature ideas

Parity with Codex/Claude desktop:

- Rich transcript: markdown, syntax highlighting, collapsible thinking blocks, tool-call cards with a real diff view for edits
- Model picker + thinking-level control, session sidebar, approval dialogs, notification when a run settles
- Command palette unifying slash commands, skills, and prompt templates

Pi-only differentiators:

- Session tree visualization with fork-from-any-message (invisible in the TUI; killer GUI feature)
- Steering queue UI: mid-run `steer()`/`followUp()` as a first-class input affordance
- Extension widgets as dockable panels; extension dialogs as native modals — the extensibility story survives the GUI
- Cost/usage dashboard per session and per turn; compaction controls with a context-usage gauge
- Multi-session tabs, parallel agents on different branches, one-click HTML export for sharing

## Phasing (draft)

1. MVP: Rust RPC client + single-session chat, streaming, tool cards, approval dialogs, model picker.
2. Sessions: sidebar, tree view, fork, shared history with the TUI.
3. Extensibility: `extension_ui` bridge, command palette, widgets.
4. Differentiators: multi-agent, cost dashboard, steering UX.

## Open decisions

- Frontend framework: decided — Svelte.
- Server mode: deferred (see Transport decision). Revisit if live multi-client attach or background/daemon sessions become real requirements.
